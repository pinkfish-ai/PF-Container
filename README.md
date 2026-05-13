# PinkConnect — Self-Host Setup Guide

Deploy PinkConnect to your own AWS account using only what's in this
bundle. Top-to-bottom takes ~30 minutes once prerequisites are in place;
most of that is waiting for DocumentDB and the ALB to provision.

The bundle ships three CloudFormation templates that compose into a
clean, decoupled stack:

```
networking  ─►  docdb  ─►  ecs
   VPC          DocumentDB    ALB + Fargate service
   subnets      cluster + SG  task def + autoscaling
   NAT, IGW                   Route53 alias
```

Networking and the database are split out from the service so you can
redeploy or replace the service without touching data, and you can swap
DocumentDB for Atlas or self-managed MongoDB without touching the
networking stack.

---

## 0. What's in the bundle

This git repository contains the infrastructure code and documentation:

| File | Purpose |
|------|---------|
| `README.md` | This document. |
| `cloudformation/pinkconnect-networking.yaml` | VPC, subnets, NAT, route tables. |
| `cloudformation/pinkconnect-docdb.yaml` | Amazon DocumentDB cluster (MongoDB-wire compatible). |
| `cloudformation/pinkconnect-ecs.yaml` | ALB, Fargate cluster, task def, service, autoscaling, Route53 alias. |

Two binary artifacts are distributed **out of band** (too large for git;
hand-delivered or pulled from a signed link). Place both in the root of
this directory before starting:

| File | Source | Purpose |
|------|--------|---------|
| `pinkconnect-<version>.tar.gz` | Out-of-band from Pinkfish (1Password / signed link) | OCI image archive for the PinkConnect container (arm64). |
| `pinkfish-connections-admin-app-main.zip` | Out-of-band from Pinkfish | Small Node.js app that mints JWTs, registers OAuth providers via the admin API, and drives end-user connection flows. |

If you don't have them yet, ask Pinkfish before continuing.

---

## 1. Prerequisites

You need:

- An **AWS account** with an IAM user that has `AdministratorAccess` (or
  the equivalent scoped policy across CloudFormation, EC2, ECS, ECR,
  ELBv2, IAM, ApplicationAutoScaling, Logs, Route53, ACM, SSM, Secrets
  Manager, and DocumentDB).
- A **domain in Route53** in the same account, e.g. `example.com`.
  PinkConnect will live on a subdomain like `connect.example.com`. If
  the domain is registered elsewhere, point its nameservers at a
  Route53 hosted zone in this account first.
- **AWS CLI v2** installed locally and configured with a profile that
  authenticates to the target account (`aws sts get-caller-identity
  --profile <name>` should return your account ID).
- **Docker Desktop** (or any Docker engine). Used once, to `docker load`
  the image and `docker push` it to your private ECR repo. The host
  architecture doesn't have to match the image — `load` and `push`
  copy bytes; runtime is on Fargate.

Substitute your values into the placeholders below as you go. To keep
the examples concrete, this guide uses `connect.example.com` and AWS
profile `selfhost`.

```bash
export AWS_PROFILE=selfhost
export AWS_REGION=us-east-1
export DOMAIN=example.com
export HOST=connect.example.com
```

---

## 2. Push the container image to your private ECR

```bash
# Create the repo
aws ecr create-repository \
  --repository-name pinkconnect \
  --image-tag-mutability IMMUTABLE \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Note your account ID and the tag from the tarball filename
ACCOUNT_ID=$(aws sts get-caller-identity --profile "$AWS_PROFILE" \
  --query Account --output text)
IMAGE_TAG=3a0863ee1167   # whatever ships in the bundle
ECR_URI=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/pinkconnect:${IMAGE_TAG}

# Load, retag, push
docker load -i pinkconnect-${IMAGE_TAG}.tar.gz
# The tarball is tagged for an internal registry — retag for yours
SOURCE_TAG=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep "pinkconnect:${IMAGE_TAG}" | head -1)
docker tag "$SOURCE_TAG" "$ECR_URI"

aws ecr get-login-password --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  | docker login --username AWS --password-stdin \
      ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

docker push "$ECR_URI"
```

The image is arm64-only. The task definition pins ARM64; no action
needed on your side.

---

## 3. Deploy the networking stack

```bash
aws cloudformation deploy \
  --stack-name pinkconnect-networking \
  --template-file cloudformation/pinkconnect-networking.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

Takes ~3 minutes. Capture the outputs:

```bash
aws cloudformation describe-stacks --stack-name pinkconnect-networking \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Stacks[0].Outputs' --output table
```

You'll use these in the next two stacks:

| Output | Used by |
|--------|---------|
| `VpcId` | docdb, ecs |
| `PublicSubnetAId`, `PublicSubnetBId` | ecs (ALB lives in public subnets) |
| `PrivateSubnetAId`, `PrivateSubnetBId` | docdb, ecs (tasks + DB in private subnets) |

Convenience: pull them into env vars so the rest of the guide pastes
cleanly.

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
# Generate a strong master password. Avoid '@', '/', '"' — they break Mongo URIs.
DOCDB_PASS=$(openssl rand -base64 24 | tr -d '@/"+=' | head -c 24)pX1
echo "$DOCDB_PASS"   # save this somewhere — you need it again

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

Takes 5–10 minutes. The cluster has `DeletionPolicy: Snapshot` so it
won't silently disappear if the stack is deleted.

Capture:

```bash
db() { aws cloudformation describe-stacks --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

export DOCDB_ENDPOINT=$(db DocDbEndpoint)
export DOCDB_SG=$(db DocDbSecurityGroupId)
```

---

## 5. Generate the JWT keypair and populate SSM

PinkConnect verifies user JWTs using a public key it reads from SSM at
startup. The bundled admin app generates the matching keypair.

```bash
unzip -o pinkfish-connections-admin-app-main.zip -d .
cd pinkfish-connections-admin-app-main
npm install
npm run keygen
# writes keys/private.pem (admin app uses to sign) and keys/public.pem
# (PinkConnect uses to verify)
cd ..
```

Now put all the secrets into SSM. The ECS task definition references
each parameter by name; container won't become healthy until they're
all present.

```bash
# URL-encode the DocDB password for the Mongo URI
ENC_PASS=$(python3 -c "import urllib.parse,sys; \
  print(urllib.parse.quote(sys.argv[1], safe=''))" "$DOCDB_PASS")

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

The two encryption keys protect every per-connection OAuth credential
written to Secrets Manager. **If you lose them, those credentials are
unrecoverable** — back them up the same way you'd back up a database
master password.

The admin token authenticates the operator-only `/admin/services/*`
endpoints. Save it somewhere you can read later:

```bash
aws ssm get-parameter --name /pinkconnect/admin-token --with-decryption \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query Parameter.Value --output text
```

---

## 6. Request an ACM certificate

```bash
CERT_ARN=$(aws acm request-certificate \
  --domain-name "$HOST" \
  --validation-method DNS \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query CertificateArn --output text)
echo "$CERT_ARN"

# Read the validation CNAME ACM wants
aws acm describe-certificate --certificate-arn "$CERT_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
```

Insert the `Name`/`Value` pair as a CNAME in your Route53 hosted zone
(via the console, or via `aws route53 change-resource-record-sets`).
Validation typically completes within a few minutes on a Route53 zone.

Wait for `Status: ISSUED`:

```bash
aws acm describe-certificate --certificate-arn "$CERT_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Certificate.Status' --output text
```

---

## 7. Deploy the ECS service stack — and wire the SG mid-deploy

**Important:** the ecs stack and the docdb stack intentionally don't
know about each other's security groups, so the ECS task can't reach
DocumentDB until you authorize the task SG as ingress on the DocumentDB
SG. CloudFormation creates the ECS `Service` resource last and waits up
to ~15 minutes for the first task to become healthy — meaning if you
wait for the stack to finish before wiring the SG, the first task
fails its health check, CloudFormation eventually times out, and the
stack rolls back. The fix is to wire the SG *while* the stack is still
creating.

Two shells, or kick the deploy into the background.

### 7a. Start the deploy

```bash
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones \
  --query "HostedZones[?Name=='${DOMAIN}.'].Id" --output text --profile "$AWS_PROFILE" \
  | awk -F/ '{print $NF}')

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
```

### 7b. As soon as the task SG exists, authorize it on the DocDB SG

The `TaskSecurityGroup` resource is created within the first 30–60
seconds — well before the `Service` resource starts waiting for
healthy tasks. Poll for it in the second shell:

```bash
until TASK_SG=$(aws cloudformation describe-stack-resources \
       --stack-name pinkconnect-ecs --region "$AWS_REGION" --profile "$AWS_PROFILE" \
       --query 'StackResources[?LogicalResourceId==`TaskSecurityGroup`].PhysicalResourceId' \
       --output text 2>/dev/null) && [ -n "$TASK_SG" ]; do sleep 5; done
echo "TaskSG: $TASK_SG"

aws ec2 authorize-security-group-ingress \
  --group-id "$DOCDB_SG" \
  --source-group "$TASK_SG" \
  --protocol tcp --port 27017 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

The first task may already have started failing health checks by the
time you wire the SG. That's fine — the service will spawn a fresh
task that reaches DocumentDB successfully, and the stack reaches
`CREATE_COMPLETE` once it stabilizes.

### 7c. If you've already deployed and the stack rolled back

If you missed the window and the stack rolled back, delete it,
authorize the SG ahead of time using the *previous* TaskSecurityGroup
ID (it's stable across redeploys of the same stack name as long as the
SG name is unchanged), and redeploy. Or simpler: just re-run §7a, and
authorize as soon as the new TaskSG appears.

### 7d. Recovering an existing service that lost connectivity

If the SG was correct but the service has been running with stale
state (e.g. you rotated the DocDB password), force a fresh deploy:

```bash
aws ecs update-service \
  --cluster pinkconnect-cluster \
  --service pinkconnect-svc \
  --force-new-deployment \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

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
so a 503 here means a missing or unreadable SSM parameter — check the
CloudWatch log group `/ecs/pinkconnect` for the structured
`mcp.server.config.invalid` line.

---

## 9. Register your first OAuth provider

Per-provider OAuth credentials live in AWS Secrets Manager under the
`SecretStorePrefix` namespace. Register them via the admin API rather
than hand-creating Secrets Manager entries — the API writes the catalog
row and the secret in one transaction.

```bash
ADMIN_TOKEN=$(aws ssm get-parameter --name /pinkconnect/admin-token \
  --with-decryption --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query Parameter.Value --output text)

# Example: Google OAuth
curl -X POST "https://${HOST}/admin/services/google/deploy" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"credentials":{"client_id":"<google-client-id>","client_secret":"<google-client-secret>"}}'
```

The redirect URI registered with Google (or any other provider) must
match `https://${HOST}/connect/callback` exactly.

Full admin surface (all require `Authorization: Bearer <admin-token>`):

| Method + Path | Purpose |
|---------------|---------|
| `GET /admin/services` | List services in the catalog. |
| `GET /admin/services/:key/definition` | Show the provider definition baked into the image. |
| `POST /admin/services/:key/deploy` | Deploy a service (catalog row + provider OAuth creds in one call). |
| `PUT /admin/services/:key/credentials` | Rotate provider OAuth creds. |
| `DELETE /admin/services/:key/credentials` | Remove provider creds (disables the service). |
| `POST /admin/services/sync` | Re-sync the catalog from the image's bundled definitions. |

---

## 10. Use the admin app to drive a connection flow

The bundled admin app mints user JWTs and walks a browser through the
OAuth handshake. Configure its `.env`:

```bash
cd pinkfish-connections-admin-app-main
cp .env.example .env
```

Edit `.env`:

```
API_BASE_URL=https://connect.example.com
PORT=3000
JWT_PRIVATE_KEY_PATH=./keys/private.pem
JWT_PROVIDER_ID=local-dev-user
JWT_SELECTED_ORG=local-dev-org
JWT_TYPE=user
JWT_EXPIRES_IN=1h
OAUTH_REDIRECT_URL=http://localhost:3000/oauth/done
```

Start it:

```bash
npm start
# admin app on http://localhost:3000
```

Open `http://localhost:3000`, pick the provider you registered in §10,
click through, and watch the connection appear in the list. Per-user
refresh tokens land in Secrets Manager under the same namespace.

---

## 11. Troubleshooting

| Symptom | Cause + fix |
|---------|------------|
| ECS service stuck "deployment in backoff" after first deploy. | First task launch retried too many times. `aws ecs update-service --cluster pinkconnect-cluster --service pinkconnect-svc --force-new-deployment`. |
| `ResourceInitializationError: invalid ssm parameters: /pinkconnect/...`. | SSM parameter missing or the task exec role can't decrypt it. Re-run §5 for the missing one — ECS auto-recovers on the next attempt; no stack redeploy needed. |
| Container restart loop, log shows "Missing required environment variables". | Same root cause as above. The startup config validator lists every missing variable on stdout. |
| Container can't reach Mongo (`MongoServerSelectionError` / connect timeout). | Task SG isn't authorized on the DB SG. Re-check §8. |
| `MongoNetworkError` with TLS handshake failure. | Mongo URI is missing `tlsCAFile=/app/global-bundle.pem`. The bundle is baked into the image; the connection string must reference that exact path. |
| ALB target health check failing despite the container looking healthy. | Health check is `GET /health/ready`. The container returns 503 there until every required env var resolves. Check CloudWatch `/ecs/pinkconnect` for the `mcp.server.config.invalid` line. |
| OAuth callback fails with `invalid_redirect`. | Provider's redirect URI doesn't match `https://${HOST}/connect/callback`. Update it in the provider's developer console. |
| `core_services` collection empty after first deploy. | Catalog auto-seed runs at boot. It's idempotent — restart the task or call `POST /admin/services/sync`. |

---

## 12. Teardown

```bash
# Reverse order. Deleting the ECS stack first releases the ALB and the
# task SG; deleting docdb takes a snapshot per the DeletionPolicy.
aws cloudformation delete-stack --stack-name pinkconnect-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-networking \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# SSM params and Secrets Manager entries don't belong to any stack —
# delete by hand if you want them gone.
aws ssm delete-parameters \
  --names $(aws ssm describe-parameters \
              --parameter-filters "Key=Name,Option=BeginsWith,Values=/pinkconnect/" \
              --query 'Parameters[].Name' --output text \
              --region "$AWS_REGION" --profile "$AWS_PROFILE") \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

---

## Appendix A — Using something other than DocumentDB

The `pinkconnect-docdb.yaml` stack is optional. Any MongoDB-wire
compatible cluster works (Atlas, self-managed MongoDB, Ferret, etc.) —
build the connection string your provider gives you, drop the
DocumentDB-specific options (`replicaSet=rs0`, `retryWrites=false`,
`authMechanism=SCRAM-SHA-1`), keep `tls=true` if applicable, and store
the result at `/pinkconnect/mongodb-uri`.

If the cluster lives outside this VPC, also peer the VPCs or expose the
cluster through a service endpoint so the Fargate task can reach it.

## Appendix B — Parameter reference

`pinkconnect-ecs.yaml` parameters not covered inline:

| Parameter | Default | Notes |
|-----------|---------|-------|
| `EnvironmentName` | `pinkconnect` | Prefix for resource names. Useful if you run multiple instances per account. |
| `SsmPrefix` | `/pinkconnect` | Where the task reads its static secrets from. |
| `SecretStorePrefix` | `pinkconnect/` | Secrets Manager namespace for per-connection OAuth creds. The task role gets read/write/delete on `${SecretStorePrefix}*`. |
| `AuthMode` | `internal` | `internal` reads `${SsmPrefix}/jwt-public-key`. Set `external` and pass `AuthJwksUrl` / `AuthIssuer` / `AuthAudience` to verify against an external IdP instead. |
| `UsageTrackingEnabled` | `false` | Set `true` only if you've stored Upstash Redis credentials at `${SsmPrefix}/upstash-ratelimit-redis-{url,token}`. |
| `DesiredCount` / `MaxCount` | `1` / `5` | Target-tracking autoscaling on average CPU. |
| `TaskCpu` / `TaskMemory` | `1024` / `2048` | Fargate task size. |
