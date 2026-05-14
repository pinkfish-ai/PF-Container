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
# Required stacks, reverse order.
aws cloudformation delete-stack --stack-name pinkconnect-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-networking \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# SSM params under /pinkconnect/.
aws ssm delete-parameters --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --names $(aws ssm describe-parameters \
              --parameter-filters "Key=Name,Option=BeginsWith,Values=/pinkconnect/" \
              --query 'Parameters[].Name' --output text \
              --region "$AWS_REGION" --profile "$AWS_PROFILE")

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
`install-production.md` deploys). It also has two optional stacks (CDN
+ backup) that must come down before the ECS stack.

```bash
# Optional production stacks first.
aws cloudformation delete-stack --stack-name pinkconnect-cdn-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-backup-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Wait for them to finish before deleting the stacks they reference.
aws cloudformation wait stack-delete-complete --stack-name pinkconnect-cdn-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation wait stack-delete-complete --stack-name pinkconnect-backup-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Required stacks, reverse order.
aws cloudformation delete-stack --stack-name pinkconnect-ecs-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-docdb-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-networking-prod \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# SSM params under /pinkconnect-prod/.
aws ssm delete-parameters --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --names $(aws ssm describe-parameters \
              --parameter-filters "Key=Name,Option=BeginsWith,Values=/pinkconnect-prod/" \
              --query 'Parameters[].Name' --output text \
              --region "$AWS_REGION" --profile "$AWS_PROFILE")

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
- **The DocDB final snapshot.** `pinkconnect-docdb.yaml` has
  `DeletionPolicy: Snapshot` on the cluster, so deleting the stack
  takes one last snapshot named `<DBClusterIdentifier>-final-snapshot-<timestamp>`.
  List with `aws docdb describe-db-cluster-snapshots`; delete with
  `aws docdb delete-db-cluster-snapshot`.
- **The ECR repo + image.** If you created the `pinkconnect` ECR repo
  as part of the install, it's not in any stack. `aws ecr delete-repository --repository-name pinkconnect --force --region "$AWS_REGION" --profile "$AWS_PROFILE"`.
- **ACM certs and Route53 records you created manually for the install**
  (e.g. the validation CNAME, the wildcard cert). ACM certs can be
  deleted with `aws acm delete-certificate` once they're no longer
  attached to a load balancer or CloudFront distribution.
- **The Route53 hosted zone + the domain registration.** Leave alone
  unless you also want to release the domain — they're yours
  regardless of whether PinkConnect is deployed.
