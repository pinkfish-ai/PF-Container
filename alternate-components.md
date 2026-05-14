# PinkConnect — Alternate Components

The CFN templates in this repo are one opinionated way to run
PinkConnect. The container itself has a narrower set of actual
requirements, and most of the AWS bits are swappable. This document
spells out (a) what's invariant — the runtime contract the container
relies on — and (b) the common substitutions, with what changes for
each.

[`install-smoke.md`](./install-smoke.md) (cheap, single-AZ) and
[`install-production.md`](./install-production.md) (multi-AZ, hardened)
are the right docs to follow if you're using all of the defaults. Use
this one when you need to deviate. [`claude-setup.md`](./claude-setup.md)
is the Claude orchestrator entry that routes to whichever install
profile fits.

---

## The runtime contract

Anything that satisfies the following can host PinkConnect:

| Requirement | Detail |
|---|---|
| **Container runtime** | OCI image, **arm64 only**. Any scheduler that runs Linux containers on Graviton/arm64 nodes. Fargate, ECS-on-EC2 (Graviton), EKS (arm64 nodes), App Runner, Cloud Run (if arm64 available in the region), on-prem K8s, plain `docker run` on an arm64 host. |
| **Port** | Container listens on `:8090` (HTTP). Set `PORT` to override; the rest of the bundle assumes 8090. |
| **Health checks** | `GET /health/live` (process up) and `GET /health/ready` (process up + all required env vars resolved + Mongo reachable). Whatever load balancer / orchestrator you put in front, point its health check at `/health/ready`. |
| **Required env vars** | `MONGODB_URI`, `ADMIN_TOKEN`, `OAUTH_ENCRYPTION_KEY`, `TOKEN_ENCRYPTION_KEY`, `JWT_PUBLIC_KEY` (when `AUTH_MODE=internal`, the default), `CALLBACK_URL`. Plus `AUTH_JWKS_URL` / `AUTH_ISSUER` / `AUTH_AUDIENCE` if `AUTH_MODE=external`. The container's startup validator prints a structured `mcp.server.config.invalid` log line if any are missing. |
| **Optional env vars** | `SECRET_STORE_PROVIDER` (`aws` or `mongo`, default `mongo`), `SECRET_STORE_PREFIX` (default `pinkconnect/`), `USAGE_TRACKING_ENABLED` (default `true`), `PORT`, `NODE_ENV`, `LOG_LEVEL`, `UPSTASH_RATELIMIT_REDIS_*` (only if usage tracking is on). |
| **Database** | MongoDB-wire-compatible. The container uses the standard Mongo driver. TLS supported via `tlsCAFile` URI option. |
| **Outbound network** | The container needs HTTPS egress to (a) third-party provider APIs you're proxying (Google, Slack, OpenWeather, etc.), (b) OAuth provider authorize / token endpoints for the providers you enable, and (c) AWS APIs (Secrets Manager, SSM) if `SECRET_STORE_PROVIDER=aws`. |
| **IAM (only if storing creds in AWS Secrets Manager)** | Task/pod identity needs `secretsmanager:GetSecretValue`, `CreateSecret`, `UpdateSecret`, `PutSecretValue`, `DescribeSecret`, `DeleteSecret`, `ListSecrets`, `TagResource` on `${SecretStorePrefix}*`. Plus `ssm:GetParameters` / `kms:Decrypt` if reading static config from SSM. |
| **TLS / public endpoint** | OAuth providers require HTTPS callbacks. Anything that terminates TLS and forwards plain HTTP to the container on 8090 works (ALB, nginx, Cloudflare Tunnel, Caddy, etc.). The hostname must match `CALLBACK_URL` (`https://<host>/connect/callback`). |

If the platform you're deploying to provides those, the container runs.
Everything else in the CFN templates — NAT gateway choice, ALB-vs-NLB,
Route53 vs other DNS, ECS vs EKS — is replaceable.

---

## Substitution 1 — Database (instead of DocumentDB)

### MongoDB Atlas

The most common swap. Free tier (M0) works for evaluation; M10+ for
real workloads.

1. Skip `cloudformation/pinkconnect-docdb.yaml`. Don't deploy that
   stack at all.
2. Stand up your Atlas cluster, get the standard connection string
   (`mongodb+srv://<user>:<pass>@cluster.xxxx.mongodb.net/`).
3. Drop the DocumentDB-specific URI options when you build the
   `MONGODB_URI` to put in SSM (or wherever your secrets live):
   - Drop `replicaSet=rs0` (Atlas auto-discovers)
   - Drop `retryWrites=false` (Atlas supports retryable writes)
   - Drop `authMechanism=SCRAM-SHA-1` (Atlas defaults are fine)
   - Drop `tlsCAFile=/app/global-bundle.pem` (Atlas uses public CA
     trust; the bundled file is the AWS RDS CA)
   - Keep `tls=true`
4. In Atlas → Network Access, allow inbound from your container's
   egress IPs (the NAT gateway EIP if you're still using the
   networking stack from this repo; otherwise wherever your container
   gets its public IP from).
5. Skip the SG wire-up step in your install doc's "Deploy ECS"
   section — Atlas isn't in your VPC, so there's no SG to authorize.

The install otherwise looks identical: same `pinkconnect-ecs.yaml`,
same SSM secrets, same flow as `install-smoke.md` / `install-production.md`.

### Self-managed MongoDB

Same shape as Atlas except you choose where to run it:

- **In the same VPC**: put it on EC2 or run it on EKS. Open port
  27017 on its SG from the ECS task SG (same SG-wire-up step as
  DocumentDB, just pointing at your Mongo SG instead).
- **Outside the VPC**: same as Atlas — allow inbound from your
  container egress IPs.

Connection string is plain `mongodb://user:pass@host:27017/`. Drop
DocumentDB-specific options. Keep `tls=true` and add `tlsCAFile=...`
only if you've configured TLS with a private CA.

### FerretDB (Postgres-backed)

FerretDB speaks the Mongo wire protocol over a Postgres backend. Works
in principle (and we haven't tested it), with the caveat that some
operators FerretDB doesn't yet support could surface as runtime errors
when PinkConnect issues unusual queries. If you go this route, treat
it as a beta path and watch the container logs the first time you
exercise less-common admin endpoints.

---

## Substitution 2 — Per-connection secret store (instead of AWS Secrets Manager)

The container has a `SECRET_STORE_PROVIDER` env var: `aws` (default in
this bundle's CFN) writes per-connection credentials to AWS Secrets
Manager; `mongo` stores them in a collection in the same MongoDB
cluster.

| Mode | Pros | Cons |
|---|---|---|
| `aws` | Per-secret KMS encryption, IAM-scoped access, easy rotation via Secrets Manager APIs, audit trail in CloudTrail. | Costs ~$0.40 per secret per month at scale; AWS-only. |
| `mongo` | Zero extra infra. Works anywhere the DB works. | Credentials encrypted with `OAUTH_ENCRYPTION_KEY` / `TOKEN_ENCRYPTION_KEY` only — no per-secret KMS, no IAM scoping. The same Mongo backup story applies to the credentials. |

To use `mongo` mode with the bundled CFN, pass `SecretStoreProvider=mongo`
as a parameter override on the ECS stack — the task def picks it up
automatically:

```bash
aws cloudformation deploy --stack-name pinkconnect-ecs ... \
  --parameter-overrides ... SecretStoreProvider=mongo
```

(For non-CFN deploys — K8s / docker compose / etc. — set the env var
`SECRET_STORE_PROVIDER=mongo` however your platform injects env vars.)

No IAM changes needed for the container; if you want least-privilege,
remove the Secrets Manager grants from the task role since they're
unused in mongo mode.

---

## Substitution 3 — Compute platform (instead of Fargate)

Anything arm64 + Linux works. The image is the same; what changes is
the scheduler config.

### EKS (arm64 nodes)

You replace `pinkconnect-ecs.yaml` with K8s manifests. Sketch:

```yaml
# Deployment — arm64 node selector + the 12 env vars from the runtime contract
apiVersion: apps/v1
kind: Deployment
metadata: { name: pinkconnect }
spec:
  replicas: 1
  selector: { matchLabels: { app: pinkconnect } }
  template:
    metadata: { labels: { app: pinkconnect } }
    spec:
      nodeSelector: { kubernetes.io/arch: arm64 }
      containers:
        - name: pinkconnect
          image: <your-registry>/pinkconnect:<tag>
          ports: [{ containerPort: 8090 }]
          env:
            - { name: PORT, value: "8090" }
            - { name: NODE_ENV, value: "production" }
            # ...
            - name: MONGODB_URI
              valueFrom: { secretKeyRef: { name: pinkconnect, key: mongodb-uri } }
            # ...repeat for ADMIN_TOKEN, OAUTH_ENCRYPTION_KEY, TOKEN_ENCRYPTION_KEY, JWT_PUBLIC_KEY
          livenessProbe:  { httpGet: { path: /health/live,  port: 8090 } }
          readinessProbe: { httpGet: { path: /health/ready, port: 8090 } }
---
apiVersion: v1
kind: Service
metadata: { name: pinkconnect }
spec:
  selector: { app: pinkconnect }
  ports: [{ port: 80, targetPort: 8090 }]
---
# Plus an Ingress or LB Service that terminates HTTPS on your domain.
```

Secrets come from a `Secret` resource (populated however you populate
secrets — Sealed Secrets, External Secrets Operator pointing at AWS
Secrets Manager / Vault / etc.). If `SECRET_STORE_PROVIDER=aws`, you
also need IAM (IRSA) for the pod to talk to AWS Secrets Manager and
SSM.

### Plain Docker / Compose

For a single-box demo on a Graviton EC2 (or Apple Silicon Mac), a
`docker-compose.yaml` with the image + a local Mongo is enough.
There's no HTTPS in front, so OAuth callbacks won't work — but
API-key services and the admin API work fine over HTTP for local
testing.

### App Runner, ECS-on-EC2, on-prem

Same playbook: arm64 container + the runtime contract above + an
HTTPS endpoint pointed at port 8090.

---

## Substitution 4 — Static-config injection (instead of SSM)

The CFN templates use SSM Parameter Store referenced via the task
def's `Secrets:` block (`{ Name: MONGODB_URI, ValueFrom:
arn:aws:ssm:...:parameter/pinkconnect/mongodb-uri }`). ECS fetches
the parameter at task launch and injects it into the container as an
env var.

You can deliver the same env vars any other way:

| Source | How |
|---|---|
| K8s `Secret` | Each env var via `valueFrom.secretKeyRef`. |
| HashiCorp Vault | Vault Agent sidecar populates env vars from a template, or use the K8s Secrets engine to project Vault secrets as K8s `Secret`. |
| AWS Secrets Manager (instead of SSM) | Change the task def `ValueFrom` to a Secrets Manager ARN. ECS supports both. |
| File mounted at container start | Less common, but valid — the container reads `process.env` only, so anything that puts values into the process's environment works (e.g. `docker run --env-file`). |

What the container cares about: the env vars must be set in the
process environment before `lib/server.js` starts. The plumbing is
your choice.

---

## Substitution 5 — TLS termination / public endpoint (instead of ALB)

The container speaks plain HTTP on `:8090`. Anything that terminates
HTTPS and forwards to it works.

### Cloudflare Tunnel

Useful if you don't want to expose the container to the public internet
at all.

1. Skip the `LoadBalancer`, `HttpListener`, `HttpsListener`,
   `TargetGroup`, and `DnsRecord` resources in
   `pinkconnect-ecs.yaml` (or just don't deploy that stack).
2. Run `cloudflared` as a sidecar (or separate task in the same VPC).
3. Point the tunnel at `http://pinkconnect:8090`.
4. Cloudflare handles HTTPS termination + your domain; the task has
   no public IP.

Tradeoff: adds a Cloudflare dependency. Removes ALB cost (~$17/mo)
and NAT cost (~$33/mo) if you also drop NAT (the task only needs
egress, which a NAT or a public subnet with a public IP both
provide).

### nginx / Caddy in front

If you already run an HTTPS-terminating reverse proxy, point one of
its `server` blocks at `http://<task-ip>:8090`. Same `/health/ready`
health check pattern; same `CALLBACK_URL` must match the public
hostname.

### API Gateway

HTTP API or REST API both work as a proxy. Less common because API
Gateway pricing changes the economics for high-request workloads, but
fine for low-volume.

---

## Substitution 6 — Network topology

The networking stack creates a VPC + 2 AZs + public + private subnets
+ NAT. You can replace pieces independently.

### Use an existing VPC

Skip `pinkconnect-networking.yaml` entirely. Pass your existing VPC
ID and subnet IDs directly as parameters to `pinkconnect-ecs.yaml`
(and `pinkconnect-docdb.yaml` if you're using DocumentDB).
Requirements:
- At least two private subnets in different AZs for the task.
- At least two public subnets in different AZs for the ALB (if using
  the ALB).
- A route to the internet from the private subnets (NAT, transit
  gateway, or whatever your existing setup uses).
- DNS hostnames enabled on the VPC.

### Drop the NAT gateway

The NAT is the chunkiest fixed cost (~$33/mo in us-east-1). Two
options to drop it:

1. **Run the task in public subnets with a public IP.** Set
   `AssignPublicIp: ENABLED` on the ECS service's
   `AwsvpcConfiguration`. The task gets a routable public IP via the
   internet gateway; no NAT needed. Tradeoff: the task is technically
   reachable from the internet on its own IP, but only the ALB-SG can
   talk to it inbound, so functionally identical to the private setup.
2. **VPC endpoints for AWS services.** Interface endpoints for ECR,
   SSM, Secrets Manager, CloudWatch Logs — task can reach AWS APIs
   without NAT. Doesn't help with external provider API calls, which
   are the actual upstream traffic — those still need NAT or a public
   IP. So this only wins if you're going to use option 1 anyway.

---

## What you can't swap

A few things are baked into the image:

- **arm64-only.** No amd64 image is published. If you need amd64,
  ask Pinkfish — multi-arch builds are doable but aren't shipped today.
- **Node 24 runtime.** The image bundles Node 24; you can't downgrade.
  Process inside the container assumes Node 24 features.
- **MongoDB-wire driver only.** The persistence layer doesn't have a
  pluggable storage backend — it talks Mongo. FerretDB and similar
  count because they speak Mongo wire; SQL databases don't.
- **JWT verification: RS256 only (currently).** The container expects
  RS256-signed JWTs. ES256 / HS256 would require a code change. If
  your IdP only emits something else, you'd need to bridge with a
  token-exchange shim.
- **Health-check paths and port.** `/health/live`, `/health/ready`,
  port 8090. Not configurable beyond `PORT`.

---

## Where to ask

If you're planning a non-default deployment and want a sanity check
before you start building, contact Pinkfish. The container's runtime
contract is stable; the AWS bits in this repo are not the only way.
