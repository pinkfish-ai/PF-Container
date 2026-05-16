# PinkConnect + MCPfarm — Production Install (multi-AZ)

> **Status: validated against bundle v0.2.0 (2026-05-16).** This runbook
> is the production-hardening profile — multi-AZ, BYOK encryption, WAF,
> CloudFront, cross-region backup, autoscaling. PinkConnect §1-§11 and
> MCPfarm §12 were walked end-to-end on a single AWS account: real
> deploys, real OpenWeather smoke call through CloudFront → ALB →
> multi-AZ ECS → DocDB → Secrets Manager → vendor and back, real MCP
> dispatch through MCPfarm → PinkConnect → DocDB and back, real AWS
> Backup recovery point copied cross-region us-east-1 → us-west-2.
> Doc gaps surfaced during the run are committed into this revision.
> Still parked in `wip/` because it hasn't been run on a separate
> production AWS account yet — the bundle is small enough that this
> mostly matters for IAM/KMS isolation, not the runbook itself.

Production-grade deploy: multi-AZ DocDB, redundant NAT, VPC endpoints,
WAF on both PinkConnect and MCPfarm ALBs, BYOK encryption, CloudFront
in front of the PinkConnect ALB, AWS Backup with cross-region copy,
autoscaling on both ECS services. Fully redundant.

**Time to deploy:** ~60 minutes end-to-end (PinkConnect ~45 min;
MCPfarm adds another ~10-15). Most of that's waiting for CloudFront
propagation + DocDB; commands themselves are fast.

**Read this before starting:**
- [`../docs/gotchas.md`](../docs/gotchas.md) — especially "SG wire-up timing", "CDN ↔ ECS DNS coordination", "CloudFront certs must be in us-east-1"
- [`../install.md`](../install.md) — if you haven't installed before, smoke first; production reuses the same machinery
- [`../docs/parameter-reference.md`](../docs/parameter-reference.md) — full param tables
- [`../teardown.md`](../teardown.md) — for cleanup

---

## Production prerequisites (do BEFORE starting the install)

Production has extras beyond what smoke needs. Don't start until all
these exist — the deploy commands reference them by ARN.

| Prerequisite | Why | How to create |
|---|---|---|
| **Customer-managed KMS CMK in deploy region** | BYOK at rest for SSM SecureStrings (operator-encrypted with `--key-id`), DocDB cluster storage, and the ECS CloudWatch log groups. **Does not yet cover per-connection Secrets Manager entries** — those still use the account's `aws/secretsmanager` AWS-managed key because the container's secret-manager module doesn't pass `KmsKeyId` on `CreateSecret` (gap tracked in PIN-6349). | `aws kms create-key --description "pinkconnect-prod" --region "$AWS_REGION" --profile "$AWS_PROFILE"` — capture the `KeyId` (or alias) and resolve to ARN. **Then attach a key policy** that allows `logs.<region>.amazonaws.com` (so CloudWatch Logs can encrypt the `/ecs/*` log groups) and `backup.amazonaws.com` (so AWS Backup can encrypt recovery points). The default `kms:*` to root is **not** sufficient — Logs and Backup are service principals; their access must be granted via the key policy, not IAM. See the [snippet below the table](#kms-key-policy) for the exact policy JSON. |
| **WAFv2 regional Web ACL in deploy region** | Inputs `WebAclArn` on the ECS stack; protects the ALB. At minimum: rate-based rule + AWSManagedRulesCommonRuleSet | Console (WAF & Shield → Web ACLs → Create) or `aws wafv2 create-web-acl --scope REGIONAL --region "$AWS_REGION"` |
| **Wildcard ACM cert in `us-east-1`** | One cert covers: CloudFront viewer cert (must be us-east-1), and the ALB cert if you're deploying in us-east-1 too | `aws acm request-certificate --domain-name "*.example.com" --validation-method DNS --region us-east-1 --profile "$AWS_PROFILE"` — insert validation CNAME, wait for ISSUED |
| **AWS Backup destination vault in a different region** | Cross-region copy target | If you want the cross-region copy to also be BYOK-encrypted, create a CMK in the destination region first (`aws kms create-key --region us-west-2 --profile "$AWS_PROFILE"`) and apply the same key-policy snippet (the dest-region CMK only needs the `backup.amazonaws.com` statement — no Logs principal needed since no log groups live in the dest region). Then create the vault with that CMK: `aws backup create-backup-vault --backup-vault-name pinkconnect-prod-dr --encryption-key-arn <dest-region-cmk-arn> --region us-west-2 --profile "$AWS_PROFILE"`. Without `--encryption-key-arn` the vault uses the destination region's `aws/backup` AWS-managed key, so the copied recovery point is **not** BYOK-encrypted even though the source DocDB cluster is. Acceptable for many threat models; document the choice for your compliance team. |
| **Separate AWS account from any non-prod environment** | Blast-radius separation | Best practice — out of scope for this doc |

### KMS key policy

After creating the CMK above, replace its default key policy with one that explicitly allows the AWS services that consume it. Save to `kms-policy.json` and apply with `aws kms put-key-policy --key-id <key-id> --policy-name default --policy file://kms-policy.json --region "$AWS_REGION" --profile "$AWS_PROFILE"`.

```json
{
  "Version": "2012-10-17",
  "Id": "pinkconnect-prod-cmk",
  "Statement": [
    {
      "Sid": "EnableRootAdmin",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::<account>:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowCloudWatchLogs",
      "Effect": "Allow",
      "Principal": {"Service": "logs.<region>.amazonaws.com"},
      "Action": ["kms:Encrypt", "kms:Decrypt", "kms:ReEncrypt*", "kms:GenerateDataKey*", "kms:DescribeKey"],
      "Resource": "*",
      "Condition": {"ArnLike": {"kms:EncryptionContext:aws:logs:arn": "arn:aws:logs:<region>:<account>:log-group:/ecs/*"}}
    },
    {
      "Sid": "AllowBackupService",
      "Effect": "Allow",
      "Principal": {"Service": "backup.amazonaws.com"},
      "Action": ["kms:Encrypt", "kms:Decrypt", "kms:ReEncrypt*", "kms:GenerateDataKey*", "kms:DescribeKey", "kms:CreateGrant"],
      "Resource": "*"
    }
  ]
}
```

The `ArnLike` condition scopes the Logs grant to `/ecs/*` groups only — that's where both PinkConnect and MCPfarm write. If you also want the dest-region CMK BYOK, save a copy with only the `AllowBackupService` statement and apply to the dest-region CMK.

Set up env at the top of the session:

```bash
export AWS_PROFILE=production        # whatever your prod profile is
export AWS_REGION=us-east-1
export DOMAIN=example.com
export HOST=prod.example.com         # customer-facing hostname (behind CloudFront)
export ORIGIN_HOSTNAME=origin-prod.example.com   # internal hostname for CloudFront→ALB
export KMS_KEY_ARN=arn:aws:kms:us-east-1:<account>:key/<key-id>
export WEB_ACL_ARN=arn:aws:wafv2:us-east-1:<account>:regional/webacl/<name>/<id>
export CERT_ARN=arn:aws:acm:us-east-1:<account>:certificate/<id>   # us-east-1 wildcard
export DEST_BACKUP_VAULT_ARN=arn:aws:backup:us-west-2:<account>:backup-vault:pinkconnect-prod-dr
```

---

## 1. Push the container image

(Same as smoke. If you've already pushed in the same account, skip.)

```bash
aws ecr create-repository \
  --repository-name pinkconnect \
  --image-tag-mutability IMMUTABLE \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

ACCOUNT_ID=$(aws sts get-caller-identity --profile "$AWS_PROFILE" --query Account --output text)
IMAGE_TAG=3a0863ee1167
export ECR_URI=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/pinkconnect:${IMAGE_TAG}

docker load -i pinkconnect-${IMAGE_TAG}.tar.gz
SOURCE_TAG=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep "pinkconnect:${IMAGE_TAG}" | head -1)
docker tag "$SOURCE_TAG" "$ECR_URI"

aws ecr get-login-password --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

docker push "$ECR_URI"
```

---

## 2. Deploy networking — multi-NAT + VPC endpoints

```bash
aws cloudformation deploy \
  --stack-name pinkconnect-networking-prod \
  --template-file cloudformation/pinkconnect-networking.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --parameter-overrides \
    EnvironmentName=pinkconnect-prod \
    VpcCidr=10.41.0.0/16 \
    PublicSubnetACidr=10.41.0.0/20 PublicSubnetBCidr=10.41.16.0/20 \
    PrivateSubnetACidr=10.41.32.0/20 PrivateSubnetBCidr=10.41.48.0/20 \
    NatGatewayCount=2 \
    EnableVpcEndpoints=true
```

~5 min (VPC endpoints add 1–2 min). The non-default CIDR avoids
collisions if you ever deploy a smoke profile in the same account.
Capture outputs:

```bash
nw() { aws cloudformation describe-stacks --stack-name pinkconnect-networking-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

export VPC_ID=$(nw VpcId)
export PUB_A=$(nw PublicSubnetAId)
export PUB_B=$(nw PublicSubnetBId)
export PRIV_A=$(nw PrivateSubnetAId)
export PRIV_B=$(nw PrivateSubnetBId)
```

---

## 3. Deploy DocumentDB — multi-AZ, larger instance, 35-day retention, BYOK

```bash
DOCDB_PASS=$(openssl rand -base64 24 | tr -d '@/"+=' | head -c 24)pX1
echo "$DOCDB_PASS"   # SAVE THIS

aws cloudformation deploy \
  --stack-name pinkconnect-docdb-prod \
  --template-file cloudformation/pinkconnect-docdb.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --parameter-overrides \
    EnvironmentName=pinkconnect-prod \
    VpcId=$VPC_ID \
    PrivateSubnetAId=$PRIV_A \
    PrivateSubnetBId=$PRIV_B \
    MasterUserPassword="$DOCDB_PASS" \
    InstanceClass=db.r6g.large \
    InstanceCount=2 \
    BackupRetentionDays=35 \
    KmsKeyArn="$KMS_KEY_ARN" \
    DeletionProtection=true
```

~10–12 min (2 instances + larger class take longer). Capture:

```bash
db() { aws cloudformation describe-stacks --stack-name pinkconnect-docdb-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

export DOCDB_ENDPOINT=$(db DocDbEndpoint)
export DOCDB_SG=$(db DocDbSecurityGroupId)
export DOCDB_CLUSTER_ARN=$(db DocDbClusterArn)
```

---

## 4. JWT keypair and populate SSM

(Same as smoke but with the prod KMS CMK encrypting the SSM
SecureStrings via the `--key-id` flag.)

```bash
unzip -o pinkfish-connections-admin-app-main.zip -d .
cd pinkfish-connections-admin-app-main
npm install
# keygen refuses to overwrite if keys/ already has files (e.g. from a
# prior install or smoke run) — clear them first if you want a fresh
# production keypair. The customer-facing app must sign JWTs with the
# matching private.pem, so regen here implies coordinating the new
# public key into your app's signing setup.
rm -f keys/private.pem keys/public.pem
npm run keygen
cd ..

ENC_PASS=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1], safe=''))" "$DOCDB_PASS")
MONGO_URI="mongodb://pinkconnect:${ENC_PASS}@${DOCDB_ENDPOINT}:27017/?tls=true&tlsCAFile=/app/global-bundle.pem&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false&authMechanism=SCRAM-SHA-1&authSource=admin"

put() { aws ssm put-parameter --type SecureString --overwrite \
  --key-id "$KMS_KEY_ARN" \
  --name "$1" --value "$2" --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query Version --output text; }

put /pinkconnect-prod/mongodb-uri          "$MONGO_URI"
put /pinkconnect-prod/oauth-encryption-key "$(openssl rand -base64 32)"
put /pinkconnect-prod/token-encryption-key "$(openssl rand -base64 32)"
put /pinkconnect-prod/admin-token          "$(openssl rand -hex 32)"
put /pinkconnect-prod/jwt-public-key       "$(cat pinkfish-connections-admin-app-main/keys/public.pem)"
```

Note the prefix is `/pinkconnect-prod/` to keep prod secrets distinct
from any smoke deploy in the same account. The ECS stack's
`SsmPrefix` param picks them up.

---

## 5. ALB cert in deploy region

The ALB needs a cert in its own region (the wildcard cert from the
prerequisites lives in `us-east-1`). The ECS stack reads it from
`ALB_CERT_ARN` in §6, so set it now.

**If `$AWS_REGION` is `us-east-1`**, you can reuse the prerequisite
wildcard cert for both the ALB and CloudFront:

```bash
export ALB_CERT_ARN="$CERT_ARN"
```

**If `$AWS_REGION` is not `us-east-1`**, issue a second wildcard cert
in the ALB's region (the prerequisite cert in us-east-1 can't be
attached to a non-us-east-1 ALB):

```bash
export ALB_CERT_ARN=$(aws acm request-certificate \
  --domain-name "*.${DOMAIN}" \
  --validation-method DNS \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query CertificateArn --output text)

# Insert the validation CNAME into Route53 — repeat the same steps as
# the prerequisite cert. Wait for ISSUED before moving to §6:
aws acm wait certificate-validated \
  --certificate-arn "$ALB_CERT_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

Confirm before continuing:

```bash
test -n "$ALB_CERT_ARN" || echo "ALB_CERT_ARN is empty — STOP and resolve before §6"
```

---

## 6. Deploy ECS — hardened, CDN-aware

This deploys the ECS stack with all hardening flags on, plus the two
new flags for CDN integration (`CreateDnsRecord=false`,
`OriginHostname=<origin-hostname>`). The CDN stack in §7 will own the
customer-facing DNS record.

```bash
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones --profile "$AWS_PROFILE" \
  --query "HostedZones[?Name=='${DOMAIN}.'].Id" --output text \
  | awk -F/ '{print $NF}')

aws cloudformation deploy \
  --stack-name pinkconnect-ecs-prod \
  --template-file cloudformation/pinkconnect-ecs.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    EnvironmentName=pinkconnect-prod \
    VpcId=$VPC_ID \
    PublicSubnetAId=$PUB_A PublicSubnetBId=$PUB_B \
    PrivateSubnetAId=$PRIV_A PrivateSubnetBId=$PRIV_B \
    ContainerImage="$ECR_URI" \
    CustomDomainName="$HOST" \
    CallbackUrl="https://${HOST}/connect/callback" \
    Route53HostedZoneId="$HOSTED_ZONE_ID" \
    CertificateArn="$ALB_CERT_ARN" \
    SsmPrefix=/pinkconnect-prod \
    SecretStorePrefix=pinkconnect-prod/ \
    DesiredCount=2 \
    LogRetentionDays=365 \
    KmsKeyArn="$KMS_KEY_ARN" \
    WebAclArn="$WEB_ACL_ARN" \
    CreateDnsRecord=false \
    OriginHostname="$ORIGIN_HOSTNAME" &

# Mid-deploy SG wire-up (same pattern as smoke; see gotchas.md)
until TASK_SG=$(aws cloudformation describe-stack-resources \
       --stack-name pinkconnect-ecs-prod --region "$AWS_REGION" --profile "$AWS_PROFILE" \
       --query 'StackResources[?LogicalResourceId==`TaskSecurityGroup`].PhysicalResourceId' \
       --output text 2>/dev/null) && [ -n "$TASK_SG" ]; do sleep 5; done

aws ec2 authorize-security-group-ingress \
  --group-id "$DOCDB_SG" --source-group "$TASK_SG" \
  --protocol tcp --port 27017 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

wait
```

After completion, the ALB is reachable at `$ORIGIN_HOSTNAME` (the
ECS stack created that A-alias). `$HOST` is **not** yet resolvable —
CloudFront in §7 creates it.

---

## 7. Deploy the CDN stack

```bash
aws cloudformation deploy \
  --stack-name pinkconnect-cdn-prod \
  --template-file cloudformation/pinkconnect-cdn.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --parameter-overrides \
    EnvironmentName=pinkconnect-prod \
    CustomDomainName="$HOST" \
    OriginHostname="$ORIGIN_HOSTNAME" \
    CloudFrontCertificateArn="$CERT_ARN" \
    Route53HostedZoneId="$HOSTED_ZONE_ID" \
    PriceClass=PriceClass_100
```

~10 min — most of that is CloudFront global propagation. After the
stack reaches `CREATE_COMPLETE`, `$HOST` resolves through CloudFront.

---

## 8. Deploy the backup stack

> **Re-installing in the same account?** The backup vault has
> `DeletionPolicy: Retain` — it survives stack deletion. If you've
> torn down a prior production install, the leftover vault
> (`pinkconnect-prod-vault`) will block this deploy with a name
> conflict, surfaced as an `EarlyValidation::ResourceExistenceCheck`
> hook failure. Either delete the orphaned vault first:
> `aws backup list-recovery-points-by-backup-vault --backup-vault-name pinkconnect-prod-vault --region "$AWS_REGION" --profile "$AWS_PROFILE"`
> (confirm 0 recoveries or delete them), then
> `aws backup delete-backup-vault --backup-vault-name pinkconnect-prod-vault --region "$AWS_REGION" --profile "$AWS_PROFILE"`,
> or change `EnvironmentName` to keep the old vault around.

```bash
aws cloudformation deploy \
  --stack-name pinkconnect-backup-prod \
  --template-file cloudformation/pinkconnect-backup.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    EnvironmentName=pinkconnect-prod \
    DocDbClusterArn="$DOCDB_CLUSTER_ARN" \
    RetentionDays=35 \
    DestinationBackupVaultArn="$DEST_BACKUP_VAULT_ARN"
```

Capture the role ARN for on-demand backup commands below:

```bash
bk() { aws cloudformation describe-stacks --stack-name pinkconnect-backup-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

export BACKUP_ROLE_ARN=$(bk BackupRoleArn)
export BACKUP_VAULT_NAME=$(bk BackupVaultName)
```

---

## 9. Verify

Customer-facing endpoint (through CloudFront):

```bash
curl -i "https://${HOST}/health/live"
# HTTP/2 200   {"status":"ok"}

curl -i "https://${HOST}/health/ready"
# HTTP/2 200   {"status":"ready"}
```

Confirm the path actually goes through CloudFront:

```bash
curl -sI "https://${HOST}/health/ready" | grep -i "via\|x-cache"
# Expected: Via: 1.1 ... .cloudfront.net  /  X-Cache: Miss from cloudfront
```

If the headers show CloudFront, the CloudFront → ALB origin path is
healthy. If you see a TLS error instead, see `../docs/troubleshooting.md`
("CloudFront returns 502 from CloudFront origin").

---

## 10. Validate the cross-region backup copy

The plan's `CopyAction` fires on scheduled runs; to validate
on-demand, run a backup + copy explicitly. See `../docs/gotchas.md` ("AWS
Backup `CopyAction` only fires on scheduled runs") for why.

```bash
# Trigger a source recovery point
BACKUP_JOB_ID=$(aws backup start-backup-job \
  --backup-vault-name "$BACKUP_VAULT_NAME" \
  --resource-arn "$DOCDB_CLUSTER_ARN" \
  --iam-role-arn "$BACKUP_ROLE_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query BackupJobId --output text)

echo "Backup job: $BACKUP_JOB_ID — waiting for completion..."

# Poll for completion (~few min for an empty cluster)
until [ "$(aws backup describe-backup-job --backup-job-id "$BACKUP_JOB_ID" \
            --region "$AWS_REGION" --profile "$AWS_PROFILE" \
            --query State --output text)" = "COMPLETED" ]; do sleep 30; done

# Grab the recovery point ARN
RP_ARN=$(aws backup describe-backup-job --backup-job-id "$BACKUP_JOB_ID" \
          --region "$AWS_REGION" --profile "$AWS_PROFILE" \
          --query RecoveryPointArn --output text)

# Copy it cross-region — async; capture the CopyJobId so we can poll.
COPY_JOB_ID=$(aws backup start-copy-job \
  --recovery-point-arn "$RP_ARN" \
  --source-backup-vault-name "$BACKUP_VAULT_NAME" \
  --destination-backup-vault-arn "$DEST_BACKUP_VAULT_ARN" \
  --iam-role-arn "$BACKUP_ROLE_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query CopyJobId --output text)

echo "Copy job: $COPY_JOB_ID — waiting for completion..."

# Poll until COMPLETED (or fail loudly on terminal failure states).
until STATE=$(aws backup describe-copy-job --copy-job-id "$COPY_JOB_ID" \
                --region "$AWS_REGION" --profile "$AWS_PROFILE" \
                --query 'CopyJob.State' --output text) && \
      [[ "$STATE" =~ ^(COMPLETED|FAILED|PARTIAL)$ ]]; do sleep 30; done
test "$STATE" = "COMPLETED" || { echo "Copy job ended in $STATE — check console"; exit 1; }

# Now verify the destination vault actually sees the copied recovery point.
DEST_VAULT_NAME=$(echo "$DEST_BACKUP_VAULT_ARN" | awk -F: '{print $NF}' | awk -F/ '{print $NF}')
DEST_REGION=$(echo "$DEST_BACKUP_VAULT_ARN" | awk -F: '{print $4}')

aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "$DEST_VAULT_NAME" \
  --region "$DEST_REGION" --profile "$AWS_PROFILE" \
  --query 'RecoveryPoints[].RecoveryPointArn'
# Should include a recovery point ARN ending in the destination region
```

This validates: destination vault exists, IAM role works, cross-region
transfer succeeds. CFN already validated the `CopyAction` block syntax
at deploy time.

**About destination-vault encryption:** the copied recovery point is
encrypted with whatever key the destination vault uses. If you created
the destination vault without `--encryption-key-arn` (the default per
the prerequisites table above), the copy uses the destination region's
`aws/backup` AWS-managed key — **not** the source CMK, even if the
source DocDB cluster is BYOK-encrypted. To get BYOK end-to-end across
regions, re-create the destination vault with a CMK in the destination
region. Acceptable for many threat models to skip; document the choice
for your compliance team.

---

## 11. Smoke test with a real OpenWeather connection

Same flow as `../install.md` § 8 but pointed at the production
host. Confirms the JWT verify / encryption / proxy pipeline works
through the full production topology (CloudFront → ALB → multi-AZ
ECS → multi-AZ DocDB).

**WAF false-positive on `http://localhost*` redirect URLs.** The admin
app's `.env.example` ships with
`OAUTH_REDIRECT_URL=http://localhost:3000/oauth/done`. If your prereq
Web ACL includes `AWSManagedRulesCommonRuleSet` (recommended baseline),
its `EC2MetaDataSSRF_BODY` rule blocks every request body containing
`http://localhost*` with a 403 — including the connection-create POST
below, which embeds `redirect_url` in the body. Set
`OAUTH_REDIRECT_URL` to a non-localhost value (any HTTPS URL works for
this test — the OAuth callback never fires for API-key services like
OpenWeather) before running this section, or scope-down the rule to
exclude `/manage/*`.

```bash
cd pinkfish-connections-admin-app-main
cp .env.example .env
# Edit .env: API_BASE_URL=https://${HOST}
# Edit .env: OAUTH_REDIRECT_URL=https://${HOST}/oauth/done   (NOT http://localhost*)
npm start &
cd ..

# Deploy openweather core service
curl -X POST "http://localhost:3000/api/admin/services/openweather/deploy" \
  -H 'x-is-admin: true' -H 'content-type: application/json' -d '{}'

# Create connection (custom_fields, not credentials)
curl -X POST "http://localhost:3000/api/connections/core/openweather" \
  -H 'content-type: application/json' \
  -d '{"name":"My OpenWeather","custom_fields":{"api_key":"<your-key>"}}'

# Real proxy call through PinkConnect
CONN_ID=<from previous response>
curl "http://localhost:3000/api/proxy/openweather/${CONN_ID}/data/2.5/weather?lat=44.34&lon=10.99"
# 200 with real OpenWeather JSON
```

---

## 12. Deploy MCPfarm — hardened, production-grade

MCPfarm is the MCP server layer on top of PinkConnect. Production
hardening: multi-AZ tasks, autoscaling, WAF on the MCPfarm ALB, BYOK
KMS on the log group, real rate-limiter backend (Upstash by default
for prod — `noop` only makes sense for smoke), 365-day log retention.

Skip this section entirely if you only want the PinkConnect credential
proxy layer.

### 12.1 Push the MCPfarm image

```bash
MCP_TAG=v0.2.0   # match VERSION + RELEASE-NOTES.md for this bundle
MCP_REPO=mcpfarm
MCP_ECR_URI="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MCP_REPO}"

aws ecr create-repository --repository-name "$MCP_REPO" \
  --image-tag-mutability IMMUTABLE \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true

docker load -i "mcpfarm-${MCP_TAG}.tar.gz"
docker tag "pinkfish-ai/mcpfarm:${MCP_TAG}" "${MCP_ECR_URI}:${MCP_TAG}"
docker push "${MCP_ECR_URI}:${MCP_TAG}"
```

### 12.2 Cert for the MCPfarm hostname

The prereq us-east-1 wildcard cert (`$CERT_ARN`) covers
`mcp.example.com` if your zone is `example.com`. If `$AWS_REGION` is
us-east-1, reuse `$CERT_ARN`. Otherwise issue a wildcard in
`$AWS_REGION` for the ALB (same shape as §5):

```bash
# If $AWS_REGION == us-east-1:
export MCP_CERT_ARN="$CERT_ARN"

# Otherwise:
export MCP_CERT_ARN=$(aws acm request-certificate \
  --domain-name "*.${DOMAIN}" \
  --validation-method DNS \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query CertificateArn --output text)
# Insert validation CNAME (same as §5), wait for ISSUED.
```

### 12.3 Pick a rate-limiter backend

Production should pick a real backend — `noop` (the smoke default)
disables rate-limiting entirely, which is rarely appropriate for an
internet-facing MCP layer. Options:

| Backend | When | Setup |
|---|---|---|
| `upstash` (recommended) | Default production choice — managed Redis, free tier covers light load, paid tiers scale to millions of req/day | Sign up at https://upstash.com, capture REST URL + token, put into SSM (commands below) |
| `elasticache` | You already run ElastiCache in this VPC | App-side code in place; CFN secret wiring not yet plumbed in this template — you'll need to add the SSM param refs by hand |
| `dynamodb` | You prefer a fully serverless backend keyed by AWS | Same status as elasticache — app code yes, CFN wiring no |
| `noop` | Edge already rate-limits (CloudFront, API Gateway, your own L7 throttle) | None — pick noop in §12.4's deploy params and skip the SSM puts below |

For Upstash:

```bash
aws ssm put-parameter --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --type SecureString --overwrite \
  --key-id "$KMS_KEY_ARN" \
  --name /pinkconnect-prod/upstash-ratelimit-redis-url \
  --value '<your-rest-url>'

aws ssm put-parameter --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --type SecureString --overwrite \
  --key-id "$KMS_KEY_ARN" \
  --name /pinkconnect-prod/upstash-ratelimit-redis-token \
  --value '<your-rest-token>'
```

`/pinkconnect-prod/jwt-public-key` was already populated in §4 —
MCPfarm shares the same key (so JWTs minted by the customer app verify
against either service identically).

### 12.4 Deploy `mcpfarm-ecs.yaml` with production hardening

```bash
MCP_HOST=mcp.${DOMAIN}                       # customer-facing
RATE_LIMITER_BACKEND=upstash                 # or 'noop' / 'elasticache' / 'dynamodb'
PINKCONNECT_URL=$(aws cloudformation describe-stacks \
  --stack-name pinkconnect-ecs-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='PublicUrl'].OutputValue" \
  --output text)

aws cloudformation deploy \
  --stack-name mcpfarm-ecs-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --template-file cloudformation/mcpfarm-ecs.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    EnvironmentName=mcpfarm-prod \
    VpcId="$VPC_ID" \
    PublicSubnetAId="$PUB_A" PublicSubnetBId="$PUB_B" \
    PrivateSubnetAId="$PRIV_A" PrivateSubnetBId="$PRIV_B" \
    ContainerImage="${MCP_ECR_URI}:${MCP_TAG}" \
    ConnectUrl="$PINKCONNECT_URL" \
    CustomDomainName="$MCP_HOST" \
    Route53HostedZoneId="$HOSTED_ZONE_ID" \
    CertificateArn="$MCP_CERT_ARN" \
    SsmPrefix=/pinkconnect-prod \
    DesiredCount=2 \
    MaxCount=6 \
    LogRetentionDays=365 \
    KmsKeyArn="$KMS_KEY_ARN" \
    WebAclArn="$WEB_ACL_ARN" \
    RateLimiterBackend="$RATE_LIMITER_BACKEND"
```

**~8-10 min.** Same parameter pattern as the PinkConnect ECS stack —
`KmsKeyArn` encrypts the log group; `WebAclArn` attaches your prereq
WAF Web ACL to the MCPfarm ALB; `DesiredCount=2 / MaxCount=6` runs two
tasks across AZs and lets ApplicationAutoScaling scale up to six on
CPU pressure.

### 12.5 Verify

```bash
curl -sI "https://${MCP_HOST}/health/live"
# HTTP/2 200

# Catalog (PIN-6414): list all MCP servers the snapshot covers
TOKEN=$(curl -s http://localhost:3000/api/debug/jwt | jq -r .token)
curl -sH "Authorization: Bearer $TOKEN" \
  "https://${MCP_HOST}/catalog?includeChildren=true" | jq 'length'
# 812 (bundle 0.2.0 — local + remote MCP servers)
```

### 12.6 End-to-end MCP dispatch smoke

You already created an OpenWeather connection in §11. Use it to prove
the MCP → PinkConnect → upstream path works in production:

```bash
CONN_ID='<from §11>'
TOKEN=$(curl -s http://localhost:3000/api/debug/jwt | jq -r .token)

curl -sS -X POST "https://${MCP_HOST}/dynamic/openweather" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d "{
    \"jsonrpc\": \"2.0\",
    \"id\": 1,
    \"method\": \"tools/call\",
    \"params\": {
      \"name\": \"openweather_get_current_weather\",
      \"arguments\": { \"PCID\": \"${CONN_ID}\", \"lat\": 51.5074, \"lon\": -0.1278 }
    }
  }" | jq '.result.structuredContent | {name, temp_c: (.main.temp - 273.15)}'
# → London weather (multi-AZ MCPfarm task → PinkConnect ALB → DocDB →
#    Secrets Manager → real OpenWeather API → back through the chain)
```

This single curl confirms:
- ✅ MCPfarm reachable through its production ALB (WAF + multi-AZ)
- ✅ JWT verifies against the shared SSM signing key
- ✅ Rate-limit middleware in front (Upstash if you picked it)
- ✅ MCPfarm dispatches via `$PINKCONNECT_URL`
- ✅ PinkConnect fetches your encrypted OpenWeather credential
- ✅ Real upstream call to OpenWeather, response flows back

---

## Tear down (when no longer needed)

See [`../teardown.md`](../teardown.md). Order: **MCPfarm first** (because
its `ConnectUrl` references PinkConnect's ALB), then CDN + backup
first, then ecs, then docdb, then networking. Plus the SSM params
under `/pinkconnect-prod/*`, the Secrets Manager entries, ACM certs,
ECR images (`pinkconnect` and `mcpfarm`), destination backup vault.

---

## Operational notes

- **JWT keypair rotation:** see `../docs/gotchas.md` ("JWT keypair regen requires force-redeploy"). Both PinkConnect and MCPfarm read the same `/pinkconnect-prod/jwt-public-key` SSM param — `force-new-deployment` both services after rotating.
- **Image upgrades:** push a new tag to ECR, then `aws cloudformation deploy --parameter-overrides ContainerImage=<new-uri>`. ECS rolls the service with the deployment circuit breaker enabled (auto-rollback on bad releases). Same pattern for `pinkconnect-ecs-prod` and `mcpfarm-ecs-prod`.
- **Adding more orgs:** PinkConnect partitions by `(selectedOrg, providerId)` claims in the JWT. Your app mints JWTs with whatever `selectedOrg` the user is in; no infra change needed for either service.
- **Monitoring:** CloudWatch log groups `/ecs/pinkconnect-prod` and `/ecs/mcpfarm-prod`. Set up a metric filter on each for `mcp.server.config.invalid` to alert on config drift. The Upstash rate-limiter has its own dashboard at https://console.upstash.com for request volume + 429 rates.
- **MCPfarm has no persistent state.** It reads the baked snapshot at container start. Unlike PinkConnect (which holds the credential DB in DocDB), no backup is needed — losing all MCPfarm tasks just means a fresh deploy from the same image returns identical behavior.
