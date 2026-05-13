# PinkConnect — Teardown

Delete everything you deployed. Order matters: stacks that depend on
other stacks come down first.

The order:

1. Optional stacks (CDN, backup) — deleting them first releases
   resources that reference the ECS stack.
2. ECS stack (releases the ALB, task SG, IAM roles).
3. DocumentDB stack (takes a final snapshot per `DeletionPolicy: Snapshot`).
4. Networking stack (frees NAT, EIP, IGW).
5. SSM params, Secrets Manager entries, and any orphaned snapshots — these aren't owned by any stack and won't be deleted automatically.

```bash
# Optional stacks — only run the lines you actually deployed.

aws cloudformation delete-stack --stack-name pinkconnect-cdn \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

aws cloudformation delete-stack --stack-name pinkconnect-backup \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# Required stacks, reverse-order.

aws cloudformation delete-stack --stack-name pinkconnect-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-docdb \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws cloudformation delete-stack --stack-name pinkconnect-networking \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

# SSM params — not owned by any stack.

aws ssm delete-parameters --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --names $(aws ssm describe-parameters \
              --parameter-filters "Key=Name,Option=BeginsWith,Values=/pinkconnect/" \
              --query 'Parameters[].Name' --output text \
              --region "$AWS_REGION" --profile "$AWS_PROFILE")

# Per-connection secrets, if any.

aws secretsmanager list-secrets --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'SecretList[?starts_with(Name, `pinkconnect/`)].Name' --output text \
  | xargs -n1 -I{} aws secretsmanager delete-secret \
      --secret-id {} --force-delete-without-recovery \
      --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

## Things that survive teardown

- **AWS Backup vault and recovery points.** The `pinkconnect-backup`
  stack's vault has `DeletionPolicy: Retain`. Deleting the stack
  removes the plan and selection but leaves the vault and any
  recovery points untouched. To fully clean up, delete the recovery
  points first (`aws backup list-recovery-points-by-backup-vault` →
  `aws backup delete-recovery-point`), then delete the vault.
- **The destination-region vault (cross-region copy).** Pinkfish
  customers create this manually; it's not in any stack. Delete with
  `aws backup delete-backup-vault --backup-vault-name <name> --region <dest-region>`
  after clearing recovery points.
- **The DocDB final snapshot.** `pinkconnect-docdb.yaml` has
  `DeletionPolicy: Snapshot` on the cluster, so deleting the stack
  takes one last snapshot. List with `aws docdb describe-db-cluster-snapshots`;
  delete with `aws docdb delete-db-cluster-snapshot`.
- **The ECR repo + image.** If you created `pinkconnect` ECR repo as
  part of the install, it's not in any stack. `aws ecr delete-repository
  --repository-name pinkconnect --force --region "$AWS_REGION"
  --profile "$AWS_PROFILE"`.
- **ACM certs and Route53 records you created manually for the
  install** (e.g. the validation CNAME). ACM cert can be deleted with
  `aws acm delete-certificate` once it's no longer attached to a
  load balancer / CloudFront distribution.
- **The Route53 hosted zone + the domain registration itself.** Leave
  these alone unless you also want to release the domain — they're
  yours regardless of whether PinkConnect is deployed.
