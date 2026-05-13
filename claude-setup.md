# PinkConnect — Self-Host (Claude orchestrator entry)

You (Claude) are about to install PinkConnect into a customer's AWS
account. This file is the orchestrator entry — it tells you how to
behave and which other file to drive the install from. The human
approves AWS prompts and pastes credentials when asked; you run the
commands, watch outputs, and recover from errors.

If a human is reading this directly: the human-facing overview is in
[`README.md`](./README.md). The actual step-by-step install runbooks
are [`install-smoke.md`](./install-smoke.md) and
[`install-production.md`](./install-production.md) — pick one.

---

## For Claude — orchestration rules

### 1. Ask the human four things up front

1. **AWS profile name and account ID.** Verify with
   `aws sts get-caller-identity --profile <name>` before doing
   anything destructive. Stop if it errors.
2. **Region.** Default `us-east-1`; ARM64 Fargate is supported in all
   common regions but confirm.
3. **Domain.** Must already exist as a Route53 hosted zone in this
   account. Ask which subdomain PinkConnect should live on (e.g.
   `connect.example.com` for smoke, `prod.example.com` for production).
4. **Profile: smoke or production?** See the table below. Don't guess
   — the answer changes which install doc you follow, what
   prerequisites the human needs ready, and what it costs.

### 2. Decide which install doc to follow

| Customer answer | Follow | Cost | Use case |
|---|---|---|---|
| **Smoke** | [`install-smoke.md`](./install-smoke.md) | ~$145/mo | Single-AZ, no WAF, no CDN, no cross-region backup. Validate the install works on the customer's AWS, then either keep running for dev or tear down. |
| **Production** | [`install-production.md`](./install-production.md) | ~$300–500/mo | Multi-AZ DocDB, redundant NAT, VPC endpoints, WAF on the ALB, BYOK CMK, 365-day logs, 35-day backup retention with cross-region copy, CloudFront in front of the ALB. Customer-facing service. Has prerequisites the human pre-creates (see install-production.md § "Production prerequisites"). |

Drive the install end-to-end from whichever install doc the human
picked. The doc is self-contained — you don't need to merge content
from multiple files.

### 3. Confirm both binary artifacts are on disk

```bash
ls pinkconnect-*.tar.gz pinkfish-connections-admin-app-main.zip
```

Both must exist. If either is missing, **stop** and tell the human to
get them from Pinkfish — you can't conjure them, and the install
can't proceed without both.

### 4. Read these before deploying

- [`gotchas.md`](./gotchas.md) — non-obvious behaviors that bite mid-install. Read in full before starting step 1.
- [`troubleshooting.md`](./troubleshooting.md) — symptom → cause+fix when something breaks during/after install.
- [`parameter-reference.md`](./parameter-reference.md) — what every CFN parameter does. Reference when you need to know what a flag means.
- [`teardown.md`](./teardown.md) — delete-everything sequence for when the customer's done.
- [`alternate-components.md`](./alternate-components.md) — swap-out playbook (Atlas instead of DocDB, EKS instead of Fargate, Cloudflare instead of CloudFront, etc.). Only relevant if the human asks "can we use X instead of Y?"

### 5. Phase ordering and parallelization (both profiles)

Same shape for smoke and production; production adds two extra stacks
(cdn, backup) and uses hardened parameters throughout.

| Phase | What | Wait | Parallelizable |
|---|---|---|---|
| 1 | Verify prereqs (profile, Docker, binaries) | <1 min | — |
| 2 | Create ECR, load + push image | 2 min | yes — with §3 |
| 3 | Deploy networking stack | 3–5 min | yes |
| 4 | Deploy DocumentDB stack | 8–12 min | starts after §3, runs alongside §5–6 |
| 5 | Generate JWT keypair, populate SSM | 1 min | runs alongside §4 |
| 6 | Request ACM cert, insert DNS validation CNAME | 2 min | runs alongside §4 |
| 7 | Deploy ECS stack with mid-deploy SG wire-up | 8 min | requires §3–6 done |
| 7b *(prod only)* | Deploy CDN stack | 10 min (CloudFront propagation) | requires §7 |
| 7c *(prod only)* | Deploy backup stack | 1 min | requires §3, §4 done |
| 8 | Verify `/health/ready` | <1 min | requires §7 (and §7b for prod, since prod's `$HOST` resolves through CloudFront) |
| 9 | Deploy openweather + create connection + proxy call | 1 min | requires §8 |
| 10 *(prod only)* | Validate cross-region backup copy via on-demand `start-copy-job` | 5 min | requires §7c |

### 6. Success criteria per phase (verify, don't ask the human)

| Phase | Success signal |
|---|---|
| 1 | `aws sts get-caller-identity` returns the expected account; `docker info` doesn't error; both binaries `ls`-visible |
| 2 | `aws ecr describe-images --repository-name pinkconnect` lists the pushed tag |
| 3 | Stack status `CREATE_COMPLETE`; 5 outputs (`VpcId` + 4 subnet IDs) |
| 4 | Stack status `CREATE_COMPLETE`; `DocDbEndpoint`, `DocDbSecurityGroupId`, `DocDbClusterArn` outputs |
| 5 | 5 SSM parameters under the `SsmPrefix` exist |
| 6 | `aws acm describe-certificate` returns `Status: ISSUED` |
| 7 | Stack status `CREATE_COMPLETE`; `TaskSecurityGroupId` output |
| 7b | Stack status `CREATE_COMPLETE`; `curl -sI https://<host>/health/ready` returns `Via: ... .cloudfront.net` header |
| 7c | Stack status `CREATE_COMPLETE`; `BackupRoleArn` + `BackupVaultName` outputs |
| 8 | `curl https://<host>/health/ready` returns `200 {"status":"ready"}` |
| 9 | Proxy call through PinkConnect returns real upstream data |
| 10 | `aws backup list-recovery-points-by-backup-vault --region <dest-region>` shows the copied recovery point |

### 7. When something breaks

Read these in order:

1. The error message itself. AWS errors usually name the resource and the failure mode.
2. [`troubleshooting.md`](./troubleshooting.md). Look up the symptom.
3. CloudWatch log group `/ecs/<env>` for the structured `mcp.server.config.invalid` line — single most useful signal when `/health/ready` is stuck at 503.
4. The CFN template itself (they're short, ~100–400 lines each).
5. As a last resort, tell the human and surface a specific question.

Don't blindly retry destructive actions. If `cloudformation delete-stack` hangs, investigate (orphaned ENIs, retained snapshots) — don't `aws ec2 delete-network-interface --force` your way out without checking what holds it.

---

## License + support

License: TBD — contact Pinkfish for terms before redistributing.

Support: pf-support@pinkfish.ai. Same address for getting the two
binary artifacts.
