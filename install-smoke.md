# PinkConnect — Smoke Install (single-AZ)

Cheapest deploy that exercises every layer end-to-end. Single-AZ
DocDB, single NAT, default IAM scoping, no WAF, no CDN, no
cross-region backup. Intended to prove the install works on your AWS
before standing up the production profile.

When not in use, tear down the three stacks (see `teardown.md`); only
the Route53 hosted zone persists.

**Time to deploy:** ~30 minutes end-to-end. Most of that is waiting
for DocumentDB (~8 min) and the ALB (~5 min) to provision.

**Reference docs to keep open while doing this install:**
- [`gotchas.md`](./gotchas.md) — read first, especially "SG wire-up timing"
- [`troubleshooting.md`](./troubleshooting.md) — for when something breaks
- [`parameter-reference.md`](./parameter-reference.md) — what every CFN param means
- [`teardown.md`](./teardown.md) — when you're done

---

## Prerequisites

- **AWS account** + IAM user/role with `AdministratorAccess` (or the equivalent scoped policy across CloudFormation, EC2, ECS, ECR, ELBv2, IAM, ApplicationAutoScaling, Logs, Route53, ACM, SSM, Secrets Manager, DocumentDB).
- **A domain in Route53** in this account (e.g. `example.com`). If it's registered elsewhere, point its nameservers at a Route53 hosted zone in this account first.
- **AWS CLI v2**, configured profile. Verify with `aws sts get-caller-identity --profile <name>`.
- **Docker** (Desktop or engine). Used once, to `docker load` the tarball and `docker push` it to ECR.
- The two binary artifacts from Pinkfish dropped into this directory:
  - `pinkconnect-<version>.tar.gz`
  - `pinkfish-connections-admin-app-main.zip`

Fix once at the top of the session:

```bash
export AWS_PROFILE=selfhost          # whatever profile name you configured
export AWS_REGION=us-east-1
export DOMAIN=example.com
export HOST=connect.example.com
```

---

## 1. Push the container image to your private ECR

```bash
aws ecr create-repository \
  --repository-name pinkconnect \
  --image-tag-mutability IMMUTABLE \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

ACCOUNT_ID=$(aws sts get-caller-identity --profile "$AWS_PROFILE" --query Account --output text)
IMAGE_TAG=3a0863ee1167   # match the tag in the tarball filename
export ECR_URI=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/pinkconnect:${IMAGE_TAG}

docker load -i pinkconnect-${IMAGE_TAG}.tar.gz
SOURCE_TAG=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep "pinkconnect:${IMAGE_TAG}" | head -1)
docker tag "$SOURCE_TAG" "$ECR_URI"

aws ecr get-login-password --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

docker push "$ECR_URI"
```

Verify: `aws ecr describe-images --repository-name pinkconnect --region "$AWS_REGION" --profile "$AWS_PROFILE"`.

---

## 2. Deploy the networking stack

```bash
aws cloudformation deploy \
  --stack-name pinkconnect-networking \
  --template-file cloudformation/pinkconnect-networking.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

~3 minutes. Capture outputs:

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

## 3. Deploy DocumentDB

```bash
DOCDB_PASS=$(openssl rand -base64 24 | tr -d '@/"+=' | head -c 24)pX1
echo "$DOCDB_PASS"   # SAVE THIS — needed for the URI in §4

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

~8 minutes. Single instance (`InstanceCount: 1`), single-AZ — fine
for smoke. Capture:

```bash
db() { aws cloudformation describe-stacks --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

export DOCDB_ENDPOINT=$(db DocDbEndpoint)
export DOCDB_SG=$(db DocDbSecurityGroupId)
```

---

## 4. Generate JWT keypair and populate SSM

```bash
unzip -o pinkfish-connections-admin-app-main.zip -d .
cd pinkfish-connections-admin-app-main
npm install
npm run keygen           # writes keys/private.pem + keys/public.pem
cd ..
```

Build the Mongo URI (URL-encode the password) and put all 5 secrets
into SSM:

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
database master password** — losing them makes every stored
per-connection credential unrecoverable. See `gotchas.md`.

---

## 5. Request the ACM certificate

```bash
export CERT_ARN=$(aws acm request-certificate \
  --domain-name "$HOST" \
  --validation-method DNS \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query CertificateArn --output text)
```

Read the validation CNAME and insert it as a Route53 record:

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

## 6. Deploy ECS — and wire the task SG mid-deploy

**Critical timing:** the ECS stack and the docdb stack don't know
about each other's SGs. CFN creates the ECS `Service` resource last
and waits ~15 min for the first task to be healthy; if the task SG
isn't authorized on the docdb SG within that window, the task fails
its health check and the stack rolls back. Fix: start the deploy in
the background, poll for the task SG to appear, authorize it
immediately. See `gotchas.md` if you want the full story.

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

# Wait for the deploy to finish
wait
```

If the deploy beats you to the `Service` resource and the first task
is already failing, keep going — the service auto-replaces it after
the SG is authorized, and the stack reaches `CREATE_COMPLETE` on
retry.

---

## 7. Verify

```bash
curl -i "https://${HOST}/health/live"
# HTTP/2 200   {"status":"ok"}

curl -i "https://${HOST}/health/ready"
# HTTP/2 200   {"status":"ready"}
```

`/health/live` flips green as soon as the process is up.
`/health/ready` only goes green once every required env var resolved,
so a 503 here = missing or unreadable SSM parameter. Check CloudWatch
log group `/ecs/pinkconnect` for the `mcp.server.config.invalid` line
— it lists exactly what's missing.

---

## 8. Smoke test with a real OpenWeather connection

Proves the full pipeline: JWT verify → admin deploy → user-connection
→ Secrets Manager → runtime proxy → upstream API. Get a free
OpenWeather API key from
[openweathermap.org/api](https://openweathermap.org/api) (or pick any
API-key service from `GET /admin/services`).

Start the admin app — it mints JWTs for the calls below:

```bash
cd pinkfish-connections-admin-app-main
cp .env.example .env
# Edit .env: API_BASE_URL=https://<your-HOST>
npm start &
cd ..
```

Deploy the openweather core service:

```bash
curl -X POST "http://localhost:3000/api/admin/services/openweather/deploy" \
  -H 'x-is-admin: true' \
  -H 'content-type: application/json' \
  -d '{}'
# 200 {"id":"...","deployed":true,"status":"active"}
```

Create a user-connection. **For API-key services, the key goes in
`custom_fields`, not `credentials`** (see `gotchas.md`):

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

Run a real upstream call through PinkConnect's proxy:

```bash
CONN_ID=<from previous response>
curl "http://localhost:3000/api/proxy/openweather/${CONN_ID}/data/2.5/weather?lat=44.34&lon=10.99"
# 200 with the OpenWeather JSON wrapped in {"output": {...}}. Proves:
# JWT verify → DB lookup → Secrets Manager read → decrypt → inject
# appid → upstream call → response. The wrapping comes from the
# admin app's /api/proxy/* route, which forwards the upstream body
# unchanged inside an `output` envelope so connection metadata can
# travel alongside in the future. Hitting PinkConnect directly
# (https://${HOST}/connect/openweather/${CONN_ID}/data/2.5/weather?...
# with auth-token: <JWT>) returns the raw upstream body without the
# envelope.
```

---

## 9. Optional — Deploy MCPfarm

MCPfarm is the Pinkfish MCP server packaged as a container. It runs in
the same VPC as PinkConnect, talks ONLY to your PinkConnect ALB for
credentialed upstream calls, and serves the same `/dynamic/<id>` route
Pinkfish prod uses — so any MCP client can call it without knowing
whether it's hitting Pinkfish or your self-host. This step is optional;
skip it if you only need the PinkConnect auth layer.

### 9.1 Push the MCPfarm image to your ECR

The MCPfarm image is shipped from Pinkfish as a tarball, the same way
PinkConnect's image was (see §2). Replace the tag with the version
Pinkfish sent you:

```bash
# Match the tag in the tarball filename — bundle v0.2.0 ships
# mcpfarm-v0.2.0.tar.gz. Check VERSION + RELEASE-NOTES.md in this
# directory for the current bundle's component versions.
MCP_TAG=v0.2.0
MCP_REPO=mcpfarm
MCP_ECR_URI="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MCP_REPO}"

aws ecr create-repository --repository-name "$MCP_REPO" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true
docker load -i "mcpfarm-${MCP_TAG}.tar.gz"
docker tag "pinkfish-ai/mcpfarm:${MCP_TAG}" "${MCP_ECR_URI}:${MCP_TAG}"
docker push "${MCP_ECR_URI}:${MCP_TAG}"
```

### 9.2 Cert for the MCPfarm hostname

If your `CustomDomainName` cert from §5 is a wildcard (`*.example.com`)
it already covers `mcp.example.com` — skip this step. Otherwise, request
a second cert covering `mcp.example.com` exactly as you did in §5, add
the DNS-validation CNAME, and capture the new `MCP_CERT_ARN`.

### 9.3 (Optional) Pick a rate-limiter backend

MCPfarm has a pluggable rate-limiter (PIN-6384). The CFN template
exposes `RateLimiterBackend` with these choices:

| Backend | When to use | Extra setup |
|---|---|---|
| `noop` (default) | Smoke installs; environments where your edge / app already throttles | None |
| `upstash` | Anything internet-facing that needs MCP-layer rate limiting | Sign up at https://upstash.com (free tier covers smoke-scale), then put the REST URL + token in SSM (commands below) |
| `elasticache` / `dynamodb` | Customizing — application code supports these but the CFN template doesn't wire their SSM params yet | Roll your own |

For smoke, leave `RateLimiterBackend` at its default (`noop`) and skip
the rest of this section. The container starts without any rate-limit
dependency.

If you want Upstash:

```bash
# After creating the Upstash Redis instance — REST URL + token from
# the Upstash console.
aws ssm put-parameter \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --name /pinkconnect/upstash-ratelimit-redis-url \
  --type SecureString --overwrite --value '<your-rest-url>'
aws ssm put-parameter \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --name /pinkconnect/upstash-ratelimit-redis-token \
  --type SecureString --overwrite --value '<your-rest-token>'
```

…then pass `RateLimiterBackend=upstash` in the deploy step below.

### 9.4 Deploy `mcpfarm-ecs.yaml`

```bash
# CONNECT_URL is the customer-facing URL of YOUR PinkConnect (the
# PublicUrl Output of the pinkconnect-ecs stack from §6).
PINKCONNECT_URL=$(aws cloudformation describe-stacks \
  --stack-name pinkconnect-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='PublicUrl'].OutputValue" \
  --output text)

MCP_HOST=mcp.example.com           # Edit
MCP_CERT_ARN=arn:aws:acm:...       # From §5 wildcard OR §9.2

aws cloudformation deploy \
  --stack-name mcpfarm-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --template-file cloudformation/mcpfarm-ecs.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    VpcId="$VPC_ID" \
    PublicSubnetAId="$PUB_A" \
    PublicSubnetBId="$PUB_B" \
    PrivateSubnetAId="$PRIV_A" \
    PrivateSubnetBId="$PRIV_B" \
    ContainerImage="${MCP_ECR_URI}:${MCP_TAG}" \
    ConnectUrl="$PINKCONNECT_URL" \
    CustomDomainName="$MCP_HOST" \
    Route53HostedZoneId="$HOSTED_ZONE_ID" \
    CertificateArn="$MCP_CERT_ARN"
    # Add `RateLimiterBackend=upstash` here if you set up Upstash in §9.3.
    # Default (no override) is `noop` — no rate limiting, no Upstash dependency.
```

### 9.5 Smoke-test the dispatch path

You already created an OpenWeather connection in §8. The MCP server
calls back to PinkConnect (via `CONNECT_URL`) to fetch the stored
credential, then makes the upstream call to api.openweathermap.org.

```bash
# The admin app you started in §8 already mints JWTs signed against the
# keypair you generated — grab one from its debug endpoint:
JWT=$(curl -s http://localhost:3000/api/debug/jwt | jq -r .token)

# (The token is RS256-signed with the private key in keys/private.pem,
# carries the providerId + selectedOrg claims from .env, and is verified
# by MCPfarm against /pinkconnect/jwt-public-key. If you want to sign
# JWTs from your own application instead, see the JWT signing notes in
# gotchas.md — the smoke doesn't require a specific issuer.)

PCID='<connection_id from §8>'   # the OpenWeather connection

# The Accept header is REQUIRED by the MCP HTTP transport — without it
# the server returns `{"jsonrpc":"2.0","error":{"code":-32000,"message":
# "Not Acceptable: Client must accept application/json"}}`. The MCP
# spec mandates both `application/json` and `text/event-stream`.
curl -sS -X POST "https://${MCP_HOST}/dynamic/openweather" \
  -H "Authorization: Bearer ${JWT}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "openweather_get_current_weather",
      "arguments": { "PCID": "'"$PCID"'", "lat": 51.5074, "lon": -0.1278 }
    }
  }'
# → JSON-RPC result with real OpenWeather data for London.
```

That single curl proves:

- ✅ MCPfarm container is deployed and reachable in your AWS
- ✅ JWT auth works at the MCP layer (verified against PinkConnect's signing key)
- ✅ Rate-limit middleware is in front (Upstash-backed)
- ✅ MCPfarm dispatches the call through YOUR PinkConnect via `CONNECT_URL`
- ✅ PinkConnect injects YOUR stored OpenWeather credential
- ✅ Real vendor API returns data
- ✅ Response flows back up the chain — **zero Pinkfish-hosted services in the path**.

---

## What's missing vs. production

If you're going to leave this deployment running for real traffic,
read `install-production.md` — it's the same install but with:

- Multi-AZ DocDB (HA across AZs)
- Multi-NAT for cross-AZ failure tolerance
- VPC endpoints (faster cold starts, less NAT bandwidth)
- WAF on the ALB
- Customer-managed KMS keys (BYOK)
- 365-day log retention
- 35-day backup retention with cross-region copy
- CloudFront in front of the ALB
- 2+ ECS tasks instead of 1

See `install-production.md` for the parameter changes.

---

## Tear it down

Follow [`teardown.md`](./teardown.md) → **"Tear down a smoke install"**.

There's an order-of-operations subtlety (the install-time SG ingress
authorization isn't owned by CloudFormation, so it must be revoked
before deleting the ECS stack or that delete will hang) plus
post-stack cleanup for SSM params, Secrets Manager entries, ACM
cert, and the ECR repo. teardown.md is the single source of truth so
you don't get a half-cleaned environment from following a stale
shortcut.
