# PinkConnect — Production Install (multi-AZ, ~$300–500/mo)

Production-grade deploy: multi-AZ DocDB, redundant NAT, VPC endpoints,
WAF, BYOK encryption, CloudFront in front of the ALB, AWS Backup with
cross-region copy. ~2x the smoke deploy cost, fully redundant.

**Cost target:** ~$300–500/mo depending on traffic, NAT bandwidth, and
CloudFront price class.

**Time to deploy:** ~45 minutes end-to-end (a bit longer than smoke
because CloudFront propagation takes ~10 min and there are 2 extra
stacks). Most of that's waiting; commands themselves are fast.

**Read this before starting:**
- [`gotchas.md`](./gotchas.md) — especially "SG wire-up timing", "CDN ↔ ECS DNS coordination", "CloudFront certs must be in us-east-1"
- [`install-smoke.md`](./install-smoke.md) — if you haven't installed before, smoke first; production reuses the same machinery
- [`parameter-reference.md`](./parameter-reference.md) — full param tables
- [`teardown.md`](./teardown.md) — for cleanup

---

## Production prerequisites (do BEFORE starting the install)

Production has extras beyond what smoke needs. Don't start until all
these exist — the deploy commands reference them by ARN.

| Prerequisite | Why | How to create |
|---|---|---|
| **Customer-managed KMS CMK in deploy region** | BYOK encryption for SSM SecureStrings + Secrets Manager + DocDB cluster storage | `aws kms create-key --description "pinkconnect-prod" --region "$AWS_REGION" --profile "$AWS_PROFILE"` — capture the `KeyId` (or alias) and resolve to ARN. |
| **WAFv2 regional Web ACL in deploy region** | Inputs `WebAclArn` on the ECS stack; protects the ALB. At minimum: rate-based rule + AWSManagedRulesCommonRuleSet | Console (WAF & Shield → Web ACLs → Create) or `aws wafv2 create-web-acl --scope REGIONAL --region "$AWS_REGION"` |
| **Wildcard ACM cert in `us-east-1`** | One cert covers: CloudFront viewer cert (must be us-east-1), and the ALB cert if you're deploying in us-east-1 too | `aws acm request-certificate --domain-name "*.example.com" --validation-method DNS --region us-east-1 --profile "$AWS_PROFILE"` — insert validation CNAME, wait for ISSUED |
| **AWS Backup destination vault in a different region** | Cross-region copy target | If you want the cross-region copy to also be BYOK-encrypted, create a CMK in the destination region first (`aws kms create-key --region us-west-2 --profile "$AWS_PROFILE"`) and pass its ARN as `--encryption-key-arn` below: `aws backup create-backup-vault --backup-vault-name pinkconnect-prod-dr --encryption-key-arn <dest-region-cmk-arn> --region us-west-2 --profile "$AWS_PROFILE"`. Without `--encryption-key-arn` the vault uses the destination region's `aws/backup` AWS-managed key, so the copied recovery point is **not** BYOK-encrypted even though the source DocDB cluster is. Acceptable for many threat models; document the choice for your compliance team. |
| **Separate AWS account from any non-prod environment** | Blast-radius separation | Best practice — out of scope for this doc |

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
    KmsKeyArn="$KMS_KEY_ARN"
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
healthy. If you see a TLS error instead, see `troubleshooting.md`
("CloudFront returns 502 from CloudFront origin").

---

## 10. Validate the cross-region backup copy

The plan's `CopyAction` fires on scheduled runs; to validate
on-demand, run a backup + copy explicitly. See `gotchas.md` ("AWS
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

# Copy it cross-region
aws backup start-copy-job \
  --recovery-point-arn "$RP_ARN" \
  --source-backup-vault-name "$BACKUP_VAULT_NAME" \
  --destination-backup-vault-arn "$DEST_BACKUP_VAULT_ARN" \
  --iam-role-arn "$BACKUP_ROLE_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Verify the destination vault sees the copied recovery point
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

Same flow as `install-smoke.md` § 8 but pointed at the production
host. Confirms the JWT verify / encryption / proxy pipeline works
through the full production topology (CloudFront → ALB → multi-AZ
ECS → multi-AZ DocDB).

```bash
cd pinkfish-connections-admin-app-main
cp .env.example .env
# Edit .env: API_BASE_URL=https://${HOST}
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

## Tear down (when no longer needed)

See [`teardown.md`](./teardown.md). Order: CDN + backup first, then
ecs, then docdb, then networking. Plus the SSM params under
`/pinkconnect-prod/*`, the Secrets Manager entries, ACM certs,
ECR image, destination backup vault.

---

## Operational notes

- **JWT keypair rotation:** see `gotchas.md` ("JWT keypair regen requires force-redeploy").
- **Image upgrades:** push a new tag to ECR, then `aws cloudformation deploy --parameter-overrides ContainerImage=<new-uri>`. ECS rolls the service with the deployment circuit breaker enabled (auto-rollback on bad releases).
- **Adding more orgs:** PinkConnect partitions by `(selectedOrg, providerId)` claims in the JWT. Your app mints JWTs with whatever `selectedOrg` the user is in; no infra change needed.
- **Monitoring:** CloudWatch log group `/ecs/pinkconnect-prod`. Set up a metric filter for `mcp.server.config.invalid` to alert on config drift.
