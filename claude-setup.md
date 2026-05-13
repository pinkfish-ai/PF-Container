# PinkConnect — Self-Host (Claude install guide)

This is the dense install runbook **for Claude to follow.** For the
human-facing overview, see [`README.md`](./README.md).

Drop the repo into a Claude Code session (or Claude.ai with a terminal
connector) and ask Claude to "install PinkConnect into my AWS account
following claude-setup.md." The human approves AWS prompts and pastes
credentials when asked; Claude runs the commands, watches outputs, and
recovers from errors. A human can also follow it manually — the
sequencing works either way, it's just denser than a typical install
doc because it assumes an LLM is reading.

Top-to-bottom takes ~30 minutes once prerequisites are in place; most
of that is waiting for DocumentDB (~8 min) and the ALB (~5 min) to
provision.

```
networking  ─►  docdb  ─►  ecs
   VPC          DocumentDB    ALB + Fargate service
   subnets      cluster + SG  task def + autoscaling
   NAT, IGW                   Route53 alias
```

The three stacks compose into one deployment but stay decoupled: you
can redeploy the service without touching data, and you can swap
DocumentDB for Atlas or self-managed MongoDB without touching the
networking stack.

---

## For Claude (orchestrator)

If you're Claude reading this to run the install, here's how to wear
the orchestrator hat:

**Up front, ask the human:**
1. AWS profile name and account ID (run `aws sts get-caller-identity
   --profile <name>` to verify before doing anything destructive)
2. Region (default `us-east-1`; ARM64 Fargate is supported in all
   common regions but confirm)
3. Domain (must be in Route53 in this account) and the subdomain
   PinkConnect should live on (e.g. `connect.example.com`)
4. Whether both binary artifacts are on disk:
   - `pinkconnect-<version>.tar.gz` (the image, ~125 MB)
   - `pinkfish-connections-admin-app-main.zip` (the admin app)

   If either is missing, stop and tell the human to email
   **pf-support@pinkfish.ai** to get them. The install can't proceed
   without both.

**Execution order is sequential by phase but parallel within phases:**

| Phase | What | Approx wait | Parallelizable? |
|---|---|---|---|
| 1 | Verify prereqs (profile, Docker, binaries) | <1 min | — |
| 2 | Create ECR, load + push image | 2 min | yes — run alongside §3 |
| 3 | Deploy networking stack | 3 min | yes |
| 4 | Deploy DocumentDB stack | 8 min | starts after §3, runs alongside §5–6 |
| 5 | Generate JWT keypair, populate SSM | 1 min | runs alongside §4 |
| 6 | Request ACM cert, insert DNS validation CNAME | 2 min | runs alongside §4 |
| 7 | Deploy ECS stack + wire SG mid-deploy | 8 min | requires §3–6 done |
| 8 | Verify `/health/ready` | <1 min | requires §7 |
| 9 | Deploy a service + create a connection (smoke test) | 1 min | requires §8 |

**Success criteria per phase** (Claude verifies, doesn't ask the human):

| Phase | Success signal |
|---|---|
| 1 | `aws sts get-caller-identity` returns the expected account; `docker info` doesn't error; both binaries `ls`-visible |
| 2 | `aws ecr describe-images --repository-name pinkconnect` lists the pushed tag |
| 3 | Stack status `CREATE_COMPLETE`; 5 outputs (VpcId + 4 subnet IDs) |
| 4 | Stack status `CREATE_COMPLETE`; `DocDbEndpoint`, `DocDbSecurityGroupId` outputs |
| 5 | 5 SSM parameters under `/pinkconnect/*` exist |
| 6 | `aws acm describe-certificate` returns `Status: ISSUED` |
| 7 | Stack status `CREATE_COMPLETE`; `TaskSecurityGroupId` output |
| 8 | `curl https://<host>/health/ready` returns `200 {"status":"ready"}` |
| 9 | Proxy call through PinkConnect returns real upstream data |

**Non-obvious things that will trip you if you don't know them up front
(see [Gotchas](#gotchas) for detail):**

- The image is **arm64-only**. The task definition pins `ARM64`; don't
  override.
- The ECS service waits up to 15 min for tasks to become healthy. The
  task can't reach DocumentDB until you authorize the task SG on the
  DocDB SG — **do this while the ECS deploy is still running**, not
  after, or the stack rolls back. Use a background poll for the
  `TaskSecurityGroup` resource and authorize as soon as it exists.
- For API-key services, the per-connection key goes in `custom_fields`,
  not `credentials`, when calling `POST /manage/user-connections/...`.
- `GET /admin/*` requires a JWT with `is_admin: true` claim. The admin
  app's UI controls this via a checkbox; if you bypass the UI, set
  header `x-is-admin: true` or sign the JWT with that claim directly.
- ECR repo is created with `ImageTagMutability: IMMUTABLE`. New
  versions need a fresh tag.
- The admin app's catalog and connections panels are **snapshots**, not
  live. After every deploy/create/revoke, the human (or you) needs to
  click "Load catalog" / "Load connections" again. Or query the
  backend directly and report state to the human.
- DocDB master password cannot contain `@`, `/`, `"`. The README's
  example generator strips them. If you let the human pick, validate.

**When you hit something I didn't:** read the relevant CFN template
(they're short, ~100–400 lines each), read the container's
`/ecs/pinkconnect` CloudWatch log group, and use what you find. The
container's startup config validator prints a structured
`mcp.server.config.invalid` log line listing every missing env var,
which is the single most useful signal when `/health/ready` is stuck
at 503.

---

## 0. What's in the bundle

This git repo:

| File | Purpose |
|------|---------|
| `README.md` | This document. |
| `cloudformation/pinkconnect-networking.yaml` | VPC, subnets, NAT, route tables. |
| `cloudformation/pinkconnect-docdb.yaml` | Amazon DocumentDB cluster (MongoDB-wire compatible). |
| `cloudformation/pinkconnect-ecs.yaml` | ALB, Fargate cluster, task def, service, autoscaling, Route53 alias. |

Two binary artifacts delivered out of band by Pinkfish. Drop both into
the root of this directory before starting:

| File | Purpose |
|------|---------|
| `pinkconnect-<version>.tar.gz` | OCI image archive for the PinkConnect container (arm64, ~125 MB). |
| `pinkfish-connections-admin-app-main.zip` | Small Node.js app that mints RS256 JWTs, registers OAuth providers via the admin API, and drives end-user connection flows. |

If you don't have them, **email pf-support@pinkfish.ai** and ask for
the two artifacts above. Don't continue until both are sitting in this
directory — Claude can't conjure them and the install can't proceed
without them.

---

## 1. Prerequisites

Required:

- AWS account + IAM user/role with `AdministratorAccess` (or the
  equivalent scoped policy across CloudFormation, EC2, ECS, ECR, ELBv2,
  IAM, ApplicationAutoScaling, Logs, Route53, ACM, SSM, Secrets
  Manager, DocumentDB).
- A domain in Route53 in this account (e.g. `example.com`). If the
  domain is registered elsewhere, point its nameservers at a Route53
  hosted zone in this account first.
- AWS CLI v2, configured profile. Verify with:

  ```bash
  aws sts get-caller-identity --profile <name>
  ```

- Docker (Desktop or engine). Used once, to `docker load` the tarball
  and `docker push` to your private ECR. Host architecture is
  irrelevant for load/push — runtime is Fargate.

Fix once at the top of the session; reused everywhere:

```bash
export AWS_PROFILE=selfhost
export AWS_REGION=us-east-1
export DOMAIN=example.com
export HOST=connect.example.com
```

---

## 2. Push the container image to your private ECR

```bash
aws ecr create-repository \
  --repository-name pinkconnect \
  --image-tag-mutability IMMUTABLE \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

ACCOUNT_ID=$(aws sts get-caller-identity --profile "$AWS_PROFILE" --query Account --output text)
IMAGE_TAG=3a0863ee1167   # match the tag in the tarball filename
ECR_URI=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/pinkconnect:${IMAGE_TAG}

docker load -i pinkconnect-${IMAGE_TAG}.tar.gz
# The tarball ships with an internal-registry tag; retag for your ECR.
SOURCE_TAG=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep "pinkconnect:${IMAGE_TAG}" | head -1)
docker tag "$SOURCE_TAG" "$ECR_URI"

aws ecr get-login-password --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

docker push "$ECR_URI"
```

Verify with `aws ecr describe-images --repository-name pinkconnect`.

---

## 3. Deploy the networking stack

```bash
aws cloudformation deploy \
  --stack-name pinkconnect-networking \
  --template-file cloudformation/pinkconnect-networking.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

~3 minutes. Capture the outputs into env vars:

```bash
nw() { aws cloudformation describe-stacks --stack-name pinkconnect-networking \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

export VPC_ID=$(nw VpcId)
export PUB_A=$(nw PublicSubnetAId)
export PUB_B=$(nw PublicSubnetBId)
export PRIV_A=$(nw PrivateSubnetAId)
export PRIV_B=$(nw PrivateSubnetBId)
```

---

## 4. Deploy DocumentDB

```bash
# Master password: avoid '@', '/', '"' — they break Mongo URIs.
DOCDB_PASS=$(openssl rand -base64 24 | tr -d '@/"+=' | head -c 24)pX1
echo "$DOCDB_PASS"   # SAVE THIS — needed for the URI in §5

aws cloudformation deploy \
  --stack-name pinkconnect-docdb \
  --template-file cloudformation/pinkconnect-docdb.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --parameter-overrides \
    VpcId=$VPC_ID \
    PrivateSubnetAId=$PRIV_A \
    PrivateSubnetBId=$PRIV_B \
    MasterUserPassword="$DOCDB_PASS"
```

~8 minutes. The cluster has `DeletionPolicy: Snapshot`. Capture:

```bash
db() { aws cloudformation describe-stacks --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

export DOCDB_ENDPOINT=$(db DocDbEndpoint)
export DOCDB_SG=$(db DocDbSecurityGroupId)
```

---

## 5. Generate JWT keypair and populate SSM

PinkConnect verifies user JWTs against a public key it reads from SSM
at boot. The bundled admin app generates the matching keypair:

```bash
unzip -o pinkfish-connections-admin-app-main.zip -d .
cd pinkfish-connections-admin-app-main
npm install
npm run keygen           # writes keys/private.pem + keys/public.pem
cd ..
```

Build the Mongo URI (URL-encode the password so special chars don't
break parsing) and put all five secrets into SSM:

```bash
ENC_PASS=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1], safe=''))" "$DOCDB_PASS")

MONGO_URI="mongodb://pinkconnect:${ENC_PASS}@${DOCDB_ENDPOINT}:27017/?tls=true&tlsCAFile=/app/global-bundle.pem&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false&authMechanism=SCRAM-SHA-1&authSource=admin"

put() { aws ssm put-parameter --type SecureString --overwrite \
  --name "$1" --value "$2" --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query Version --output text; }

put /pinkconnect/mongodb-uri          "$MONGO_URI"
put /pinkconnect/oauth-encryption-key "$(openssl rand -base64 32)"
put /pinkconnect/token-encryption-key "$(openssl rand -base64 32)"
put /pinkconnect/admin-token          "$(openssl rand -hex 32)"
put /pinkconnect/jwt-public-key       "$(cat pinkfish-connections-admin-app-main/keys/public.pem)"
```

**Treat `oauth-encryption-key` and `token-encryption-key` like a
database master password.** They protect every per-connection
credential at rest in Secrets Manager. Losing them makes every stored
credential unrecoverable.

---

## 6. Request the ACM certificate

```bash
CERT_ARN=$(aws acm request-certificate \
  --domain-name "$HOST" \
  --validation-method DNS \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query CertificateArn --output text)
```

Get the validation CNAME and insert it as a Route53 record:

```bash
aws acm describe-certificate --certificate-arn "$CERT_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
# Insert the returned Name/Value as a CNAME in your hosted zone.
```

Wait for `Status: ISSUED` (1–5 min on a Route53 zone):

```bash
aws acm describe-certificate --certificate-arn "$CERT_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Certificate.Status' --output text
```

---

## 7. Deploy ECS — and wire the task SG mid-deploy

**Read this section before running the command.** The ecs stack and
the docdb stack don't know about each other's security groups. The
ECS task can't reach DocumentDB until the task SG is authorized as
ingress on the docdb SG. CloudFormation creates the `Service` resource
last and waits ~15 min for the first task to be healthy; if the SG
isn't wired in that window, the task fails its health check and the
stack rolls back.

The solution: run the deploy in the background, poll for the task SG
to be created (it appears in the first ~30s of the deploy), authorize
it immediately, then wait for the stack to complete.

```bash
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones --profile "$AWS_PROFILE" \
  --query "HostedZones[?Name=='${DOMAIN}.'].Id" --output text \
  | awk -F/ '{print $NF}')

# Kick off the deploy in the background
aws cloudformation deploy \
  --stack-name pinkconnect-ecs \
  --template-file cloudformation/pinkconnect-ecs.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    VpcId=$VPC_ID \
    PublicSubnetAId=$PUB_A PublicSubnetBId=$PUB_B \
    PrivateSubnetAId=$PRIV_A PrivateSubnetBId=$PRIV_B \
    ContainerImage="$ECR_URI" \
    CustomDomainName="$HOST" \
    CallbackUrl="https://${HOST}/connect/callback" \
    Route53HostedZoneId="$HOSTED_ZONE_ID" \
    CertificateArn="$CERT_ARN" &

# As soon as the TaskSecurityGroup resource exists, authorize it
until TASK_SG=$(aws cloudformation describe-stack-resources \
       --stack-name pinkconnect-ecs --region "$AWS_REGION" --profile "$AWS_PROFILE" \
       --query 'StackResources[?LogicalResourceId==`TaskSecurityGroup`].PhysicalResourceId' \
       --output text 2>/dev/null) && [ -n "$TASK_SG" ]; do sleep 5; done

aws ec2 authorize-security-group-ingress \
  --group-id "$DOCDB_SG" --source-group "$TASK_SG" \
  --protocol tcp --port 27017 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Wait for the deploy to finish (it should now complete cleanly)
wait
```

If the deploy beats you to the `Service` resource (first task already
failing health checks): keep going — the service auto-replaces the
failing task once the SG is authorized, and the stack reaches
`CREATE_COMPLETE` on the retry.

---

## 8. Verify

```bash
curl -i "https://${HOST}/health/live"
# HTTP/2 200   {"status":"ok"}

curl -i "https://${HOST}/health/ready"
# HTTP/2 200   {"status":"ready"}
```

`/health/live` flips green as soon as the process is up.
`/health/ready` only goes green once every required env var resolved,
so a 503 here = missing or unreadable SSM parameter. Check CloudWatch
log group `/ecs/pinkconnect` for the structured
`mcp.server.config.invalid` line — it lists exactly what's missing.

---

## 9. Smoke test with a real connection (OpenWeather worked example)

This proves the full pipeline: JWT verification → admin deploy →
user-connection → encryption → Secrets Manager → runtime proxy →
upstream API. Use your own OpenWeather API key (free tier from
[openweathermap.org/api](https://openweathermap.org/api), or any
API-key service that has a definition baked into the image — list
them with `GET /admin/services`).

```bash
ADMIN_TOKEN=$(aws ssm get-parameter --name /pinkconnect/admin-token \
  --with-decryption --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query Parameter.Value --output text)
```

Start the admin app (it'll mint JWTs for the next two calls):

```bash
cd pinkfish-connections-admin-app-main
cp .env.example .env
# Edit .env: set API_BASE_URL=https://${HOST}, leave the rest as default
npm start &
cd ..
```

Deploy the openweather core service (no OAuth creds needed for
API-key services):

```bash
curl -X POST "http://localhost:3000/api/admin/services/openweather/deploy" \
  -H 'x-is-admin: true' \
  -H 'content-type: application/json' \
  -d '{}'
# 200 {"id":"...","deployed":true,"status":"active"}
```

Create a user connection. **Key detail:** for API-key services, the
key goes in `custom_fields.api_key`, not `credentials.api_key`. (For
OAuth services, `credentials.client_id` / `credentials.client_secret`
is correct.)

```bash
curl -X POST "http://localhost:3000/api/connections/core/openweather" \
  -H 'content-type: application/json' \
  -d '{"name":"My OpenWeather","custom_fields":{"api_key":"<your-key>"}}'
# 200 {"connection_id":"d9b...","identifier":"<first8>****"}
```

Verify the encrypted secret landed in Secrets Manager:

```bash
aws secretsmanager list-secrets --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'SecretList[?starts_with(Name, `pinkconnect/`)].Name' --output table
# should include pinkconnect/creds-<connection_id>
```

And run a real upstream call through PinkConnect's proxy:

```bash
CONN_ID=<from previous response>
curl "http://localhost:3000/api/proxy/openweather/${CONN_ID}/data/2.5/weather?lat=44.34&lon=10.99"
# 200 with real OpenWeather JSON. Proves: JWT verify → DB lookup →
# Secrets Manager read → decrypt → inject appid → upstream call → response.
```

---

## Gotchas

Things that aren't obvious from reading the templates or the admin
app source. Elevated here so Claude doesn't have to learn them by
breaking things first.

| Gotcha | Detail |
|---|---|
| **Image is arm64-only** | The task definition pins `CpuArchitecture: ARM64`. Don't override unless Pinkfish gives you an amd64 build. |
| **SG wire-up timing** | The §7 background-deploy + poll-for-task-SG dance is mandatory. Doing them sequentially rolls the stack back. |
| **`custom_fields` vs `credentials`** | For API-key services, the per-connection key is a `custom_fields` value (because the service definition declares it under `custom_fields`). For OAuth services, `credentials.client_id` + `credentials.client_secret` at deploy time, no per-connection custom_fields. |
| **`is_admin` claim** | `GET /admin/*` requires `is_admin: true` in the JWT. The admin app's UI ships a checkbox; if you're hitting `/api/admin/*` from elsewhere, set header `x-is-admin: true` or sign the JWT with the claim. |
| **Catalog and connections are snapshots** | The admin app's panels don't auto-refresh. After every deploy/create/revoke, reload. Or hit the JSON endpoints directly. |
| **`ImageTagMutability: IMMUTABLE`** | Pushing a new image requires a fresh tag. CFN `ContainerImage` parameter has to change to trigger a redeploy — there's no `:latest` shortcut. |
| **DocDB password char set** | `@`, `/`, `"` break Mongo URIs. The README's generator strips them; if a human picks the password, validate. |
| **`/health/ready` is 503 until ready** | The container deliberately returns 503 on `/health/ready` until every required env var resolves. ALB target health check uses this exact path, so a 503 here keeps the task out of service. Logs print `mcp.server.config.invalid` listing what's missing. |
| **NAT gateway is the chunky cost** | The networking stack creates a single NAT (~$0.045/hr in us-east-1) regardless of whether the service is doing anything. Tear down when not in use. |
| **Encryption keys are unrecoverable** | If you lose `/pinkconnect/oauth-encryption-key` or `/pinkconnect/token-encryption-key`, every stored per-connection credential is dead. Back them up like a DB master password. |
| **JWT keypair regen requires force-redeploy** | If you regenerate the keypair and update `/pinkconnect/jwt-public-key`, the running task is still holding the old value in env. `aws ecs update-service --force-new-deployment` to pull the new value. Existing connections survive (they're keyed on JWT claims, not the signing key). |

---

## Troubleshooting

| Symptom | Cause + fix |
|---|---|
| Stack stuck `CREATE_IN_PROGRESS` on `Service` for ~15 min, then rolls back | SG wire-up didn't happen in time. See §7. |
| `ResourceInitializationError: invalid ssm parameters: /pinkconnect/...` | Missing SSM param or task exec role can't decrypt. Re-run the matching §5 line; ECS auto-recovers next attempt. |
| Container restart loop, log shows `Missing required environment variables` | Same as above; the validator lists missing vars. |
| `MongoServerSelectionError` / connect timeout from container logs | Task SG isn't authorized on DocDB SG. Re-run the `authorize-security-group-ingress` line from §7. |
| `MongoNetworkError` with TLS handshake failure | Mongo URI missing `tlsCAFile=/app/global-bundle.pem`. The bundle is baked at that exact path; the URI option must match. |
| ALB target health check failing but container looks fine in logs | Health check is `GET /health/ready`. Returns 503 until env vars resolve. Look for `mcp.server.config.invalid`. |
| OAuth callback fails with `invalid_redirect` | OAuth app's redirect URI doesn't match `https://<host>/connect/callback`. Update in the provider's console. |
| `core_services` collection empty after first deploy | Auto-seed runs idempotently at boot. Restart the task or `POST /admin/services/sync`. |
| `cloudformation delete-stack` hangs on networking | NAT gateway / EIP releases can take ~5 min. If it's longer, check for orphaned ENIs in the VPC. |

---

## Teardown

```bash
# Reverse order. Service first (releases ALB + task SG), then docdb
# (takes a snapshot per DeletionPolicy), then networking.
aws cloudformation delete-stack --stack-name pinkconnect-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-networking \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# SSM params and Secrets Manager entries don't belong to a stack —
# delete by hand if you want them gone.
aws ssm delete-parameters --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --names $(aws ssm describe-parameters \
              --parameter-filters "Key=Name,Option=BeginsWith,Values=/pinkconnect/" \
              --query 'Parameters[].Name' --output text \
              --region "$AWS_REGION" --profile "$AWS_PROFILE")

# Per-connection secrets, if any
aws secretsmanager list-secrets --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'SecretList[?starts_with(Name, `pinkconnect/`)].Name' --output text \
  | xargs -n1 -I{} aws secretsmanager delete-secret \
      --secret-id {} --force-delete-without-recovery \
      --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

---

## Appendix A — Using something other than DocumentDB

The `pinkconnect-docdb.yaml` stack is optional. Any MongoDB-wire
compatible cluster works (Atlas, self-managed MongoDB, FerretDB). Build
the connection string your provider gives you, drop the
DocumentDB-specific options (`replicaSet=rs0`, `retryWrites=false`,
`authMechanism=SCRAM-SHA-1`), keep `tls=true` if applicable, and store
the result at `/pinkconnect/mongodb-uri`. The cluster has to be
reachable from the private subnets the ECS task lives in — peer VPCs
or expose via a service endpoint if it lives outside this networking
stack.

## Appendix B — ECS stack parameter reference

`cloudformation/pinkconnect-ecs.yaml` parameters not covered inline:

| Parameter | Default | Notes |
|-----------|---------|-------|
| `EnvironmentName` | `pinkconnect` | Prefix for resource names. Set to something unique if you run multiple instances per account. |
| `SsmPrefix` | `/pinkconnect` | SSM path where the task reads its static secrets. |
| `SecretStorePrefix` | `pinkconnect/` | Secrets Manager namespace for per-connection OAuth creds. Task role gets read/write/delete on `${SecretStorePrefix}*`. |
| `AuthMode` | `internal` | `internal` reads `${SsmPrefix}/jwt-public-key`. Set `external` and pass `AuthJwksUrl` / `AuthIssuer` / `AuthAudience` to verify against an external IdP instead. |
| `UsageTrackingEnabled` | `false` | `true` requires Upstash Redis creds at `${SsmPrefix}/upstash-ratelimit-redis-{url,token}`. |
| `DesiredCount` / `MaxCount` | `1` / `5` | Target-tracking autoscaling on average CPU. |
| `TaskCpu` / `TaskMemory` | `1024` / `2048` | Fargate task size. |

---

## License + support

License: TBD — contact Pinkfish for terms before redistributing.

Support: **pf-support@pinkfish.ai**. Same address for getting the two
binary artifacts (image tarball + admin app zip) listed in §0.
