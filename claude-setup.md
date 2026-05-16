# PinkConnect + MCPfarm — Self-Host (Claude orchestrator entry)

You (Claude) are about to install **PinkConnect + MCPfarm** into a
customer's AWS account. This file is the orchestrator entry — it tells
you how to behave and which other files to drive the install from.
The human approves AWS prompts and pastes credentials when asked; you
run the commands, watch outputs, and recover from errors.

**Default flow installs both:** PinkConnect provides the credential /
proxy layer for vendor APIs (Gmail, OpenWeather, Slack, ~50 others);
MCPfarm provides the MCP server surface that exposes ~300 tools to
callers and routes through PinkConnect for credentials. Customers who
only want the connection-proxy layer can stop after PinkConnect —
explicitly opt out at install time when you ask. Customers who want
MCPfarm always need PinkConnect first; MCPfarm's `CONNECT_URL`
parameter points at PinkConnect's ALB.

If a human is reading this directly: the human-facing overview is in
[`README.md`](./README.md). The step-by-step install runbooks are
[`install.md`](./install.md) and
[`wip/install-production.md`](./wip/install-production.md). PinkConnect lives
in §1–§8 of each runbook; MCPfarm lives in §9.

---

## For Claude — orchestration rules

### 1. Ask the human six things up front

1. **AWS profile name and account ID.** Verify with
   `aws sts get-caller-identity --profile <name>` before doing
   anything destructive. Stop if it errors.
2. **Region.** Default `us-east-1`; ARM64 Fargate is supported in all
   common regions but confirm.
3. **Domain.** Must already exist as a Route53 hosted zone in this
   account. Ask which subdomain **PinkConnect** should live on (e.g.
   `connect.example.com` for smoke, `prod.example.com` for production)
   AND which subdomain **MCPfarm** should live on (e.g. `mcp.example.com`).
   They must be different hostnames in the same hosted zone.
4. **Profile: smoke or production?** See the table below. Don't guess
   — the answer changes which install doc you follow, what
   prerequisites the human needs ready.
5. **Include MCPfarm?** Default yes. Ask: "Do you want the MCP server
   layer (MCPfarm) on top of PinkConnect, or just PinkConnect alone?"
   If they say PinkConnect-only, skip phases 10a–10d below. If they
   say both (default), drive the full flow.
6. **Rate-limiter backend** *(only if §5 was "include MCPfarm")*.
   Ask: "Want rate limiting on the MCP layer? Defaults to off
   (`noop`) — fine for smoke. Say `yes` to set up Upstash Redis (free
   tier, ~2 min signup at https://upstash.com). Other backends
   available: `elasticache`, `dynamodb` — pick those only if you're
   customizing further." Their answer determines `RateLimiterBackend`
   in phase 10d's CFN deploy. If `upstash`, also drive the Upstash
   signup + SSM puts before the deploy (see install.md §9.3).

### 2. Decide which install doc to follow

| Customer answer | Follow | Use case |
|---|---|---|
| **Smoke** (default — the primary path) | [`install.md`](./install.md) | Single-AZ, no WAF, no CDN, no cross-region backup. Self-contained doc — drive the install end-to-end from it. |
| **Production** (WIP — not customer-ready yet) | [`wip/install-production.md`](./wip/install-production.md) | Multi-AZ DocDB, redundant NAT, VPC endpoints, WAF, BYOK CMK, 365-day logs, 35-day backup retention with cross-region copy, CloudFront. **Currently work-in-progress** — gaps caught during validation, hasn't been re-tested end-to-end against bundle v0.2.0. If the human asks for production, walk them through it but flag at the top that they should consult Pinkfish before relying on it. |

For 99% of installs, you'll be following `install.md`. Production is
parked in `wip/` until it's been validated end-to-end the way smoke
has.

### 3. Confirm binary artifacts are on disk

For PinkConnect (always required):

```bash
ls pinkconnect-*.tar.gz pinkfish-connections-admin-app-main.zip
```

For MCPfarm (required when phase 5 answer was "include MCPfarm"):

```bash
ls mcpfarm-*.tar.gz
```

All required artifacts must exist. If any are missing, **stop** and
tell the human to get them from Pinkfish — you can't conjure them,
and the install can't proceed without them.

### 4. Read these before deploying

- [`docs/gotchas.md`](./docs/gotchas.md) — non-obvious behaviors that bite mid-install. Read in full before starting step 1.
- [`docs/troubleshooting.md`](./docs/troubleshooting.md) — symptom → cause+fix when something breaks during/after install.
- [`docs/parameter-reference.md`](./docs/parameter-reference.md) — what every CFN parameter does. Reference when you need to know what a flag means.
- [`teardown.md`](./teardown.md) — delete-everything sequence for when the customer's done.
- [`docs/alternate-components.md`](./docs/alternate-components.md) — swap-out playbook (Atlas instead of DocDB, EKS instead of Fargate, Cloudflare instead of CloudFront, etc.). Only relevant if the human asks "can we use X instead of Y?"

### 5. Phase ordering and parallelization (both profiles)

Same shape for smoke and production; production adds two extra stacks
(cdn, backup) and uses hardened parameters throughout.

| Phase | What | Wait | Parallelizable |
|---|---|---|---|
| 1 | Verify prereqs (profile, Docker, binaries) | <1 min | — |
| 2 | Create ECR, load + push image | 2 min | yes — with §3 |
| 3 | Deploy networking stack | 3–5 min | yes |
| 4 | Deploy DocumentDB stack | 8–12 min | starts after §3, runs alongside §5–6 |
| 5a | Generate JWT keypair (`npm run keygen`) | 30 sec | runs alongside §4 |
| 5b | Populate SSM — including `mongodb-uri` | 1 min | **blocked on §4** (needs `DocDbEndpoint` to build `MONGO_URI`) |
| 6 | Request ACM cert, insert DNS validation CNAME | 2 min | runs alongside §4 |
| 7 | Deploy ECS stack with mid-deploy SG wire-up | 8 min | requires §3–6 done |
| 7b *(prod only)* | Deploy CDN stack | 10 min (CloudFront propagation) | requires §7 |
| 7c *(prod only)* | Deploy backup stack | 1 min | requires §3, §4 done |
| 8 | Verify `/health/ready` | <1 min | requires §7 (and §7b for prod, since prod's `$HOST` resolves through CloudFront) |
| 9 | Deploy openweather + create connection + proxy call (PinkConnect smoke) | 1 min | requires §8 |
| 10a *(MCPfarm)* | Push MCPfarm image to ECR | 2 min | requires §2's ECR + §9 |
| 10b *(MCPfarm)* | ACM cert for MCPfarm hostname | 2 min | runs alongside 10a |
| 10c *(MCPfarm)* | Verify JWT SSM param. If phase 6 answer was "yes, Upstash", also drive the Upstash signup + put the two SSM params; otherwise skip. | <1 min for noop, ~3 min for Upstash | requires §5b |
| 10d *(MCPfarm)* | Deploy `mcpfarm-ecs.yaml` pointing at PinkConnect ALB | 6–8 min | requires 10a, 10b, 10c |
| 10e *(MCPfarm)* | Dispatch smoke — JWT → `/dynamic/openweather` → real weather JSON | 1 min | requires 10d |
| 11 *(prod only)* | Validate cross-region backup copy via on-demand `start-copy-job` | 5 min | requires §7c |

### 6. Success criteria per phase (verify, don't ask the human)

| Phase | Success signal |
|---|---|
| 1 | `aws sts get-caller-identity` returns the expected account; `docker info` doesn't error; both binaries `ls`-visible |
| 2 | `aws ecr describe-images --repository-name pinkconnect` lists the pushed tag |
| 3 | Stack status `CREATE_COMPLETE`; 5 outputs (`VpcId` + 4 subnet IDs) |
| 4 | Stack status `CREATE_COMPLETE`; `DocDbEndpoint`, `DocDbSecurityGroupId`, `DocDbClusterArn` outputs |
| 5a | `keys/private.pem` + `keys/public.pem` exist in the admin app directory |
| 5b | 5 SSM parameters under the `SsmPrefix` exist (including `mongodb-uri`) |
| 6 | `aws acm describe-certificate` returns `Status: ISSUED` |
| 7 | Stack status `CREATE_COMPLETE`; `TaskSecurityGroupId` output |
| 7b | Stack status `CREATE_COMPLETE`; `curl -sI https://<host>/health/ready` returns `Via: ... .cloudfront.net` header |
| 7c | Stack status `CREATE_COMPLETE`; `BackupRoleArn` + `BackupVaultName` outputs |
| 8 | `curl https://<host>/health/ready` returns `200 {"status":"ready"}` |
| 9 | Proxy call through PinkConnect returns real upstream data |
| 10a | `aws ecr describe-images --repository-name mcpfarm` lists the pushed tag |
| 10b | `aws acm describe-certificate` for the MCPfarm hostname returns `Status: ISSUED` |
| 10c | `aws ssm get-parameter` returns `jwt-public-key` under `/pinkconnect/*`. Upstash params (`upstash-ratelimit-redis-url`, `upstash-ratelimit-redis-token`) are required *only* when phase 6 answer was "yes, Upstash"; skip otherwise. |
| 10d | Stack `mcpfarm-ecs` status `CREATE_COMPLETE`; `curl https://<mcpfarm-host>/health/live` returns 200. CFN deploy includes `RateLimiterBackend=<noop\|upstash\|...>` from phase 6's answer. |
| 10e | `curl -X POST https://<mcpfarm-host>/dynamic/openweather -H "Authorization: Bearer <jwt>" -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"openweather_get_current_weather","arguments":{"PCID":"<connection_id>","lat":51.5074,"lon":-0.1278}}}'` returns real OpenWeather JSON. The `PCID` argument is required — it's the connection_id from phase 9. |
| 11 | `aws backup list-recovery-points-by-backup-vault --region <dest-region>` shows the copied recovery point |

### 7. When something breaks

Read these in order:

1. The error message itself. AWS errors usually name the resource and the failure mode.
2. [`docs/troubleshooting.md`](./docs/troubleshooting.md). Look up the symptom.
3. CloudWatch log group `/ecs/<env>` for the structured `mcp.server.config.invalid` line — single most useful signal when `/health/ready` is stuck at 503.
4. The CFN template itself (they're short, ~100–400 lines each).
5. As a last resort, tell the human and surface a specific question.

Don't blindly retry destructive actions. If `cloudformation delete-stack` hangs, investigate (orphaned ENIs, retained snapshots) — don't `aws ec2 delete-network-interface --force` your way out without checking what holds it.

---

## License + support

License: TBD — contact Pinkfish for terms before redistributing.

Support: pf-support@pinkfish.ai. Same address for getting the two
binary artifacts.
