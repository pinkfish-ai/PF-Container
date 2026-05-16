# PinkConnect — Teardown

Delete everything you deployed. **Pick the section that matches the
profile you installed** — the smoke and production deploys use
different stack names and SSM/Secrets prefixes, so running the wrong
one leaves resources behind (and keeps your bill running).

The order is the same for both:

1. Optional stacks (CDN, backup) — deleting them first releases
   resources that reference the ECS stack.
2. ECS stack (releases the ALB, task SG, IAM roles).
3. DocumentDB stack (takes a final snapshot per `DeletionPolicy: Snapshot`).
4. Networking stack (frees NAT, EIP, IGW).
5. SSM params, Secrets Manager entries, and any orphaned snapshots — these aren't owned by any stack and won't be deleted automatically.

Set `$AWS_REGION` and `$AWS_PROFILE` at the top of the session before
running anything below.

---

## Tear down a **smoke** install

Smoke uses default stack names (`pinkconnect-*`) and SSM prefix
`/pinkconnect/`. CDN and backup stacks don't exist for smoke (those
are production-only).

```bash
# Optional MCPfarm stack first — has to go before pinkconnect-ecs
# because its ConnectUrl is set against PinkConnect's ALB and the SG
# pair would otherwise prevent clean teardown.
aws cloudformation delete-stack --stack-name mcpfarm-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true
aws cloudformation wait stack-delete-complete --stack-name mcpfarm-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true

# Revoke the manually-added docdb→task SG ingress rule before deleting
# the ECS stack. The install authorized this rule outside CFN, so CFN
# doesn't know to clean it up, and the ECS stack delete will hang
# waiting on a TaskSecurityGroup dependency violation.
TASK_SG=$(aws cloudformation describe-stack-resources \
  --stack-name pinkconnect-ecs --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'StackResources[?LogicalResourceId==`TaskSecurityGroup`].PhysicalResourceId' \
  --output text)
DOCDB_SG=$(aws cloudformation describe-stacks --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='DocDbSecurityGroupId'].OutputValue" \
  --output text)
aws ec2 revoke-security-group-ingress \
  --group-id "$DOCDB_SG" --source-group "$TASK_SG" \
  --protocol tcp --port 27017 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true

# Required stacks, reverse order. Wait between dependent deletes so
# the next stack's delete doesn't race a still-deleting parent.
aws cloudformation delete-stack --stack-name pinkconnect-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation wait stack-delete-complete --stack-name pinkconnect-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation wait stack-delete-complete --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-networking \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# SSM params under /pinkconnect/. `aws ... --output text` joins names
# with tabs, which doesn't word-split through the unquoted expansion
# `aws ssm delete-parameters` expects — pass them one-at-a-time via
# xargs after normalizing to newlines.
aws ssm describe-parameters \
    --parameter-filters "Key=Name,Option=BeginsWith,Values=/pinkconnect/" \
    --query 'Parameters[].Name' --output text \
    --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  | tr '\t' '\n' \
  | xargs -n1 -I{} aws ssm delete-parameter --name {} \
      --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Per-connection secrets under pinkconnect/.
aws secretsmanager list-secrets --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'SecretList[?starts_with(Name, `pinkconnect/`)].Name' --output text \
  | xargs -n1 -I{} aws secretsmanager delete-secret \
      --secret-id {} --force-delete-without-recovery \
      --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

---

## Tear down a **production** install

Production uses `-prod`-suffixed stack names and `/pinkconnect-prod/`
SSM prefix + `pinkconnect-prod/` Secrets Manager prefix (matching what
`wip/install-production.md` deploys). It has three optional stacks (CDN,
backup, mcpfarm) — all must come down before the PinkConnect ECS stack.

```bash
# Optional MCPfarm stack first — its ConnectUrl references pinkconnect-ecs-prod
aws cloudformation delete-stack --stack-name mcpfarm-ecs-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true
aws cloudformation wait stack-delete-complete --stack-name mcpfarm-ecs-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true

# Other optional production stacks (CDN, backup).
aws cloudformation delete-stack --stack-name pinkconnect-cdn-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-backup-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Wait for them to finish before deleting the stacks they reference.
aws cloudformation wait stack-delete-complete --stack-name pinkconnect-cdn-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation wait stack-delete-complete --stack-name pinkconnect-backup-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Revoke the manually-added docdb→task SG ingress rule before
# deleting the ECS stack (same reason as smoke teardown above).
TASK_SG=$(aws cloudformation describe-stack-resources \
  --stack-name pinkconnect-ecs-prod --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'StackResources[?LogicalResourceId==`TaskSecurityGroup`].PhysicalResourceId' \
  --output text)
DOCDB_SG=$(aws cloudformation describe-stacks --stack-name pinkconnect-docdb-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='DocDbSecurityGroupId'].OutputValue" \
  --output text)
aws ec2 revoke-security-group-ingress \
  --group-id "$DOCDB_SG" --source-group "$TASK_SG" \
  --protocol tcp --port 27017 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true

# Required stacks, reverse order. Wait between dependent deletes.
aws cloudformation delete-stack --stack-name pinkconnect-ecs-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation wait stack-delete-complete --stack-name pinkconnect-ecs-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# OPTIONAL: take a manual final snapshot before deleting the cluster.
# The DocDB template ships with DeletionPolicy: Delete (same as smoke),
# so the stack delete will NOT auto-create a final snapshot — make one
# yourself if you want a point-in-time you can restore from after
# teardown. AWS Backup recovery points (from the backup stack) cover
# scheduled backups, but the most recent one may be up to 24h stale.
SNAP_ID="pinkconnect-prod-final-$(date -u +%Y%m%d-%H%M)"
aws docdb create-db-cluster-snapshot \
  --db-cluster-identifier pinkconnect-prod-docdb \
  --db-cluster-snapshot-identifier "$SNAP_ID" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true
echo "Final snapshot: $SNAP_ID (delete with aws docdb delete-db-cluster-snapshot when you're sure you don't need it)"

# DocDB cluster has DeletionProtection=true (production install sets
# this so the cluster cannot be accidentally deleted). Disable it
# before the delete-stack call, otherwise the stack delete fails with
# `Cannot delete protected Cluster`. Skip if you set DeletionProtection
# to false at install time (the default).
aws docdb modify-db-cluster \
  --db-cluster-identifier pinkconnect-prod-docdb \
  --no-deletion-protection \
  --apply-immediately \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" 2>/dev/null || true

aws cloudformation delete-stack --stack-name pinkconnect-docdb-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation wait stack-delete-complete --stack-name pinkconnect-docdb-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-networking-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# SSM params under /pinkconnect-prod/. See note on the smoke section
# above — tab-separated output doesn't word-split through unquoted
# expansion. Use xargs.
aws ssm describe-parameters \
    --parameter-filters "Key=Name,Option=BeginsWith,Values=/pinkconnect-prod/" \
    --query 'Parameters[].Name' --output text \
    --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  | tr '\t' '\n' \
  | xargs -n1 -I{} aws ssm delete-parameter --name {} \
      --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Per-connection secrets under pinkconnect-prod/.
aws secretsmanager list-secrets --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'SecretList[?starts_with(Name, `pinkconnect-prod/`)].Name' --output text \
  | xargs -n1 -I{} aws secretsmanager delete-secret \
      --secret-id {} --force-delete-without-recovery \
      --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

If you used different `EnvironmentName`, `SsmPrefix`, or
`SecretStorePrefix` values during the production install (anything
other than the defaults `pinkconnect-prod`, `/pinkconnect-prod`,
`pinkconnect-prod/`), substitute your actual values into the commands
above.

### Production prerequisites cleanup *(optional)*

The production install (`wip/install-production.md` prerequisites
table) creates four AWS resources outside CloudFormation:

- Customer-managed KMS CMK in the deploy region
- WAFv2 regional Web ACL in the deploy region
- Wildcard ACM cert in `us-east-1` (covers ALB + CloudFront viewer)
- AWS Backup destination vault in a different region

Decide whether you want them gone — if you're going to re-install,
keeping them avoids re-creating + re-validating each one. Otherwise
clean up:

```bash
# 1. WAF Web ACL. Requires LockToken (versioning).
WAF_ARN=$(aws wafv2 list-web-acls --scope REGIONAL --region "$AWS_REGION" \
  --profile "$AWS_PROFILE" \
  --query "WebACLs[?Name=='pinkconnect-prod-acl'].ARN" --output text)
WAF_ID=$(echo "$WAF_ARN" | awk -F/ '{print $NF}')
WAF_NAME=$(echo "$WAF_ARN" | awk -F/ '{print $(NF-1)}')
WAF_LOCK=$(aws wafv2 get-web-acl --name "$WAF_NAME" --id "$WAF_ID" \
  --scope REGIONAL --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query LockToken --output text)
aws wafv2 delete-web-acl --name "$WAF_NAME" --id "$WAF_ID" \
  --scope REGIONAL --lock-token "$WAF_LOCK" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# 2. Destination backup vault — clear any recovery points first.
DEST_VAULT_NAME=$(echo "$DEST_BACKUP_VAULT_ARN" | awk -F: '{print $NF}' | awk -F/ '{print $NF}')
DEST_REGION=$(echo "$DEST_BACKUP_VAULT_ARN" | awk -F: '{print $4}')
aws backup list-recovery-points-by-backup-vault \
    --backup-vault-name "$DEST_VAULT_NAME" --region "$DEST_REGION" \
    --profile "$AWS_PROFILE" --query 'RecoveryPoints[].RecoveryPointArn' \
    --output text \
  | tr '\t' '\n' \
  | xargs -n1 -I{} aws backup delete-recovery-point \
      --backup-vault-name "$DEST_VAULT_NAME" --recovery-point-arn {} \
      --region "$DEST_REGION" --profile "$AWS_PROFILE"
# RP deletes are near-instant; the vault delete fails-loud if any survive.
aws backup delete-backup-vault \
  --backup-vault-name "$DEST_VAULT_NAME" \
  --region "$DEST_REGION" --profile "$AWS_PROFILE"

# 3. KMS CMKs (deploy region + dest region if you BYOK'd the dest vault).
#    Use schedule-key-deletion (7-30 day pending window — 7 is the min).
#    The key is immediately disabled; deletion happens after the window.
aws kms schedule-key-deletion --key-id "$KMS_KEY_ARN" \
  --pending-window-in-days 7 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
# Repeat for the dest-region CMK if you created one (KMS keys are
# regional — schedule deletion in the region the key lives in).

# 4. Wildcard ACM cert. Detach from any remaining LB / CloudFront
#    distributions first — `aws acm delete-certificate` fails-loud if
#    still attached. Skip this step if you want to keep the cert for
#    other uses.
aws acm delete-certificate --certificate-arn "$CERT_ARN" \
  --region us-east-1 --profile "$AWS_PROFILE"
```

KMS deletes are reversible during the pending window (`aws kms
cancel-key-deletion`); the WAF, vault, and ACM cert deletes are not.

---

## Things that survive teardown

These outlive every `delete-stack` and need manual cleanup if you
want them gone. The list applies to both smoke and production —
substitute the right stack names / prefixes when running the cleanup
commands.

- **AWS Backup vault and recovery points** (production only). The
  `pinkconnect-backup-prod` stack's vault has `DeletionPolicy: Retain`.
  Deleting the stack removes the plan and selection but leaves the
  vault and any recovery points untouched. To fully clean up, delete
  the recovery points first (`aws backup list-recovery-points-by-backup-vault`
  → `aws backup delete-recovery-point`), then delete the vault.
- **The destination-region backup vault** (production with cross-region
  copy). You created this manually outside any stack. Delete with
  `aws backup delete-backup-vault --backup-vault-name <name> --region <dest-region>`
  after clearing recovery points.
- **Manual DocDB snapshot** *(production only, if you took one)*. The
  template ships with `DeletionPolicy: Delete` for both profiles —
  stack-delete will NOT auto-create a final snapshot. The production
  teardown above takes one manually (named
  `pinkconnect-prod-final-<timestamp>`); list with
  `aws docdb describe-db-cluster-snapshots`, delete with
  `aws docdb delete-db-cluster-snapshot` once you're sure you don't
  need it. Continuous-backup recovery via AWS Backup (35-day retention
  + cross-region copy) is the production durability story; the manual
  snapshot just captures the final post-teardown moment.
- **The ECR repos + images.** Both the `pinkconnect` repo (created
  in §1 of the install) and the `mcpfarm` repo (created in §9.1 if
  you deployed MCPfarm) are outside CFN. Clean up both:
  `aws ecr delete-repository --repository-name pinkconnect --force --region "$AWS_REGION" --profile "$AWS_PROFILE"` and the same for `mcpfarm`.
- **ACM certs and Route53 records you created manually for the install**
  (e.g. the validation CNAME, the wildcard cert). ACM certs can be
  deleted with `aws acm delete-certificate` once they're no longer
  attached to a load balancer or CloudFront distribution.
- **The Route53 hosted zone + the domain registration.** Leave alone
  unless you also want to release the domain — they're yours
  regardless of whether PinkConnect is deployed.
