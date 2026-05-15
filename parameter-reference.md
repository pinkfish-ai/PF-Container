# PinkConnect — CFN parameter reference

Every parameter on every template. Defaults preserve smoke-deploy
behavior unless noted; the production install changes the ones flagged
"prod-only" below.

---

## `cloudformation/pinkconnect-networking.yaml`

| Parameter | Default | Notes |
|---|---|---|
| `EnvironmentName` | `pinkconnect` | Prefix for resource names. Override to run multiple instances in one account. |
| `VpcCidr` | `10.40.0.0/16` | VPC CIDR. Override if it clashes with peered VPCs. |
| `PublicSubnetACidr` / `PublicSubnetBCidr` | `10.40.0.0/20` / `10.40.16.0/20` | Public subnet CIDRs (ALB, NAT). Must fit inside `VpcCidr`. |
| `PrivateSubnetACidr` / `PrivateSubnetBCidr` | `10.40.32.0/20` / `10.40.48.0/20` | Private subnet CIDRs (task, DocDB). |
| `NatGatewayCount` | `1` | **prod-only**: set `2` for one NAT per AZ. Removes the cross-AZ failure mode. |
| `EnableVpcEndpoints` | `false` | **prod-only**: set `true` for interface endpoints (ECR api+dkr, Secrets Manager, SSM, Logs, KMS) + S3 gateway endpoint. Removes NAT bandwidth for AWS service traffic, speeds up Fargate cold starts. |

---

## `cloudformation/pinkconnect-docdb.yaml`

| Parameter | Default | Notes |
|---|---|---|
| `EnvironmentName` | `pinkconnect` | Prefix. |
| `VpcId` / `PrivateSubnetAId` / `PrivateSubnetBId` | required | From networking stack outputs. |
| `MasterUsername` | `pinkconnect` | DocDB master user. |
| `MasterUserPassword` | required, `NoEcho` | Master password. Avoid `@`, `/`, `"` — they break the Mongo URI. |
| `InstanceClass` | `db.t4g.medium` | **prod-only**: bump to `db.r6g.large` or larger. t-class is burstable, unsuitable for sustained load. |
| `InstanceCount` | `1` | **prod-only**: `2` (HA — primary + cross-AZ replica) or `3` (extra read replica). |
| `BackupRetentionDays` | `7` | **prod-only**: bump to `35` (max) for compliance frameworks. |
| `PreferredBackupWindow` | `07:00-08:00` | UTC backup window. Pick a low-traffic slot. |
| `PreferredMaintenanceWindow` | `sun:09:00-sun:10:00` | UTC weekly maintenance window. |
| `KmsKeyArn` | `''` | **prod-only**: BYOK CMK ARN for cluster storage encryption. Empty = AWS-managed `alias/aws/rds`. |
| `DeletionProtection` | `'false'` | **prod-only**: set `'true'` to refuse cluster deletion until the protection is explicitly turned off. Default `'false'` keeps the smoke teardown clean; set `'true'` for prod. Teardown then needs an extra `aws docdb modify-db-cluster --no-deletion-protection` step before the stack delete. |

Outputs: `DocDbClusterArn`, `DocDbEndpoint`, `DocDbReaderEndpoint`, `DocDbPort`, `DocDbSecurityGroupId`.

---

## `cloudformation/pinkconnect-ecs.yaml`

| Parameter | Default | Notes |
|---|---|---|
| `EnvironmentName` | `pinkconnect` | Prefix. |
| `VpcId`, `PublicSubnetAId`/`B`, `PrivateSubnetAId`/`B` | required | From networking outputs. |
| `ContainerImage` | required | Full ECR image URI for the PinkConnect image. arm64-only. |
| `CustomDomainName` | required | Customer-facing hostname (e.g. `connect.example.com`). |
| `Route53HostedZoneId` | required | Route53 hosted zone containing `CustomDomainName`. |
| `CertificateArn` | required | ACM cert for `CustomDomainName` in the ALB's region. |
| `CallbackUrl` | required | `https://<host>/connect/callback`. |
| `SsmPrefix` | `/pinkconnect` | SSM path where the task reads static secrets. |
| `SecretStorePrefix` | `pinkconnect/` | Secrets Manager namespace; task role is scoped to `${SecretStorePrefix}*`. |
| `SecretStoreProvider` | `aws` | `aws` = AWS Secrets Manager (KMS-encrypted, IAM-scoped, capped at 500K secrets/region). `mongo` = same MongoDB cluster (no extra infra, no AWS quota, but app-layer encryption only — no per-secret KMS). |
| `AuthMode` | `internal` | `internal` reads `${SsmPrefix}/jwt-public-key`. `external` verifies against an external IdP (set `AuthJwksUrl` / `AuthIssuer` / `AuthAudience`). |
| `AuthJwksUrl` / `AuthIssuer` / `AuthAudience` | `''` | Set when `AuthMode=external`. |
| `UsageTrackingEnabled` | `false` | `true` requires Upstash Redis at `${SsmPrefix}/upstash-ratelimit-redis-{url,token}`. |
| `DesiredCount` / `MaxCount` | `1` / `5` | Target-tracking autoscaling on average CPU. **prod**: `DesiredCount: 2+` (one task per AZ minimum). |
| `TaskCpu` / `TaskMemory` | `1024` / `2048` | Fargate task size. |
| `InternalAlb` | `false` | `true` makes the ALB internal-only (private subnets). Use for VPN/PrivateLink-only access. |
| `KmsKeyArn` | `''` | **prod-only**: BYOK CMK ARN. When set, IAM is scoped to this key (exec role `kms:Decrypt`, task role `kms:Decrypt + GenerateDataKey`). Actually BYOK at rest: SSM SecureStrings the operator puts with `--key-id <this CMK>` (see install-production.md §4) and DocDB cluster storage (via the docdb stack's same param). **Not** BYOK today: per-connection Secrets Manager entries the container creates — those still use the account's `aws/secretsmanager` AWS-managed key because the container doesn't pass `KmsKeyId` on `CreateSecret`. Closing the gap is tracked in PIN-6349. |
| `LogRetentionDays` | `90` | **prod tuning**: 90 covers SOC2/ISO27001. Bump to `365+` for HIPAA/SOX/FedRAMP (allowed: 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653). |
| `WebAclArn` | `''` | **prod-only**: WAFv2 Web ACL ARN to associate with the ALB. Recommended at minimum: rate-based + `AWSManagedRulesCommonRuleSet`. |
| `CreateDnsRecord` | `true` | Default creates a Route53 A-alias for `CustomDomainName` → ALB. Set `false` when fronting with CloudFront (CDN stack owns that record). |
| `OriginHostname` | `''` | When set (e.g. `origin-prod.example.com`), creates a second A-alias for the ALB. CloudFront uses this as the origin so HTTPS-to-origin validates. ALB cert must cover `OriginHostname` — wildcard cert easiest. |

Outputs: `AlbDnsName`, `PublicUrl`, `ClusterName`, `ServiceName`, `TaskSecurityGroupId`.

---

## `cloudformation/pinkconnect-cdn.yaml` (optional)

| Parameter | Default | Notes |
|---|---|---|
| `EnvironmentName` | `pinkconnect` | Prefix. |
| `CustomDomainName` | required | Same hostname the ECS stack would have used (`connect.example.com`); CDN owns this record now. |
| `OriginHostname` | required | Hostname the ALB is reachable at (set as `OriginHostname` on the ECS stack). Different from `CustomDomainName`. |
| `CloudFrontCertificateArn` | required | ACM cert for `CustomDomainName`, **must be in us-east-1**. |
| `Route53HostedZoneId` | required | Hosted zone containing `CustomDomainName`. |
| `PriceClass` | `PriceClass_100` | NA + EU only, cheapest. `PriceClass_200` adds Asia/ME/Africa. `PriceClass_All` adds South America + Australia. |

Outputs: `DistributionId`, `DistributionDomainName`, `PublicUrl`.

---

## `cloudformation/pinkconnect-backup.yaml` (optional)

| Parameter | Default | Notes |
|---|---|---|
| `EnvironmentName` | `pinkconnect` | Prefix. |
| `DocDbClusterArn` | required | From the docdb stack's `DocDbClusterArn` output. |
| `RetentionDays` | `35` | Source-vault retention. Max allowed in a `BackupPlan` rule. |
| `ScheduleExpression` | `cron(0 5 ? * * *)` | UTC, 6-field cron. Default = daily at 05:00 UTC. |
| `StartWindowMinutes` | `60` | Max delay before job must start. |
| `CompletionWindowMinutes` | `180` | Max time job has to complete. |
| `DestinationBackupVaultArn` | `''` | Empty = single-region backups. Set to a vault ARN in a different region (customer pre-creates) to enable cross-region copy via the plan's `CopyAction`. |

Outputs: `BackupVaultName`, `BackupVaultArn`, `BackupRoleArn`, `BackupPlanId`.

---

## `cloudformation/mcpfarm-ecs.yaml` (optional)

MCPfarm is the Pinkfish MCP server packaged as its own ECS service.
Sits alongside PinkConnect — shares the same VPC and the same
`/pinkconnect/*` SSM namespace for JWT signing + Upstash creds —
and serves the existing `/dynamic/<id>` MCP route, but loads tool
definitions from the baked-in `mcp-server-definitions/` filesystem
catalog instead of fetching them from a DDB-backed Go API. Deploy it
after `pinkconnect-ecs.yaml` so its `ConnectUrl` parameter can
reference the PinkConnect ALB.

| Parameter | Default | Notes |
|---|---|---|
| `EnvironmentName` | `mcpfarm` | Stack-wide name prefix (cluster, service, role, log group, SG, etc.). |
| `VpcId` / `PublicSubnetAId` / `PublicSubnetBId` / `PrivateSubnetAId` / `PrivateSubnetBId` | required | From the `pinkconnect-networking` stack outputs — MCPfarm shares the same VPC as PinkConnect. |
| `ContainerImage` | required | ECR image URI for the Pinkfish MCP server image (arm64 only). |
| `ConnectUrl` | required | Customer's PinkConnect base URL (the PinkConnect ALB's PublicUrl). Tool handlers POST credentialed upstream calls through this URL. **No Pinkfish-hosted endpoints permitted.** |
| `CustomDomainName` | required | DNS name for the MCPfarm ALB (e.g. `mcp.example.com`). Must be covered by `CertificateArn`. |
| `Route53HostedZoneId` | required | Hosted zone holding `CustomDomainName`. |
| `CertificateArn` | required | ACM cert in the ALB region covering `CustomDomainName`. A wildcard from the PinkConnect install works. |
| `SsmPrefix` | `/pinkconnect` | SSM namespace for shared secrets (`jwt-public-key`, `upstash-ratelimit-redis-url`, `upstash-ratelimit-redis-token`). Defaults to PinkConnect's namespace because those are shared operational secrets; override only if you want a separate namespace. |
| `DesiredCount` / `MaxCount` | `1` / `5` | Task count knobs. Default 1 is smoke-only; bump to 2+ for production. |
| `TaskCpu` / `TaskMemory` | `1024` / `2048` | Fargate sizing per task. |
| `ContainerPort` | `3020` | Port the MCP server listens on inside the container. Override only for a custom-baked image. |
| `HealthCheckPath` | `/health` | ALB target health-check path. |
| `InternalAlb` / `KmsKeyArn` / `LogRetentionDays` / `WebAclArn` / `CreateDnsRecord` | `false` / `''` / `90` / `''` / `true` | Standard hardening flags, same shape as `pinkconnect-ecs.yaml`. |

Outputs: `AlbDnsName`, `PublicUrl`, `ClusterName`, `ServiceName`, `TaskSecurityGroupId`.
