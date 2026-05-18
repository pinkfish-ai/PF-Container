# PF-Container release notes

Bundle versions ship as a coordinated set. The components inside (PinkConnect image, MCPfarm image, admin-app zip, CloudFormation templates) are each independently versioned, but the bundle pins which versions go together. Use the bundle version to talk about "what release am I on" with Pinkfish support; the per-component versions matter when you're debugging a specific layer.

## How to check what you have

| Where | Command / location |
|---|---|
| Bundle version | `cat VERSION` in this directory |
| PinkConnect image version | Filename of the `pinkconnect-*.tar.gz` you loaded |
| MCPfarm image version | Filename of the `mcpfarm-*.tar.gz` you loaded |
| Admin-app version | `package.json` `version` field inside `pinkfish-connections-admin-app-main.zip` |
| Live PinkConnect | `curl https://connect.<domain>/health/live` (returns `startedAt`, plus the image SHA in CloudWatch logs) |
| Live MCPfarm | `curl https://mcp.<domain>/health/live` (same shape) |
| CloudFormation stacks | `aws cloudformation describe-stacks --query 'Stacks[?StackName==`pinkconnect-ecs`].Parameters[?ParameterKey==`ContainerImage`].ParameterValue'` |

---

## 0.2.1 — 2026-05-18

Templates-only release. **No new binaries** — the PinkConnect image, MCPfarm image, and admin-app zip are byte-for-byte the same as 0.2.0. If you are on 0.2.0, upgrading is just dropping in the two updated CloudFormation templates; nothing to re-load or re-push to ECR.

### Components

| Component | Version | Notes |
|---|---|---|
| MCPfarm | `mcpfarm-v0.2.0.tar.gz` | Unchanged from 0.2.0 |
| PinkConnect | `pinkconnect-3a0863ee1167.tar.gz` | Unchanged from 0.1.0 |
| Admin app | `pinkfish-connections-admin-app-main.zip` (internal `package.json` `version: 0.2.0`) | Unchanged from 0.2.0 |
| CloudFormation templates | `pinkconnect-ecs.yaml`, `mcpfarm-ecs.yaml` updated | `Route53HostedZoneId` is now optional |

### What changed

- **`Route53HostedZoneId` is now an optional parameter** on `pinkconnect-ecs.yaml` and `mcpfarm-ecs.yaml` (retyped from `AWS::Route53::HostedZone::Id` to `String`, `Default: ''`). Previously it was required even when `CreateDnsRecord=false`, because CloudFormation's hosted-zone parameter type rejects an empty placeholder — which blocked deployments that manage DNS outside Route 53 (and any unattended pipeline). The parameter is only consumed by conditional DNS resources, so an empty value is inert when DNS is skipped. When `CreateDnsRecord=true` the value is still required and a missing zone fails the record-set creation with a clear error — the default Route 53-managed path is unchanged. The optional `pinkconnect-cdn.yaml` stack is untouched (it always owns DNS).
- `docs/parameter-reference.md` updated for the two rows.

### Upgrade notes

- Replace `cloudformation/pinkconnect-ecs.yaml` and `cloudformation/mcpfarm-ecs.yaml` with the 0.2.1 files. No image, admin-app, SSM, or parameter changes. Existing stacks are unaffected until their next deploy; redeploying with the new templates is a no-op unless you set `CreateDnsRecord=false`.

---

## 0.2.0 — 2026-05-16

### Components

| Component | Version | Notes |
|---|---|---|
| MCPfarm | `mcpfarm-v0.2.0.tar.gz` | Built from platform `0d3760969` — includes PIN-6414 catalog endpoint + PIN-6424 remote/external surface + PIN-6425 auth fix |
| PinkConnect | `pinkconnect-3a0863ee1167.tar.gz` | Unchanged from 0.1.0 |
| Admin app | `pinkfish-connections-admin-app-main.zip` (internal `package.json` `version: 0.2.0`) | Built from `0feab7e` — PIN-6423 full rewrite |
| CloudFormation templates | Same files as 0.1.0 | No infra schema changes |

### What's new

- **`GET /catalog` on MCPfarm.** Read-only discovery endpoint. Returns every dynamic + external + remote MCP server with `id`, `serviceKey`, `parentMcpId`, `isParent`, `serverType`, and (when requested) tools with `inputSchema` + `annotations`. Resolves the 9 services where MCP server id ≠ service_key (asana→asana-api, jira→jira-cloud, netsuite→oracle-suiteql, etc.). One JWT-signed GET replaces N per-row `tools/list` JSON-RPC probes for any client doing discovery.
- **Three MCP server types surfaced.** `dynamic` (228 unique serviceKeys, 679 entries after parent/child expansion), `external` (90 SDK-wrapped — github, gmail, slack, salesforce, etc.), `remote` (43 third-party-hosted — linear, intercom, newrelic, posthog). Clients can filter via `?serverTypes=dynamic,external` to scope responses.
- **Admin app rewrite.** Operator console now shows the full MCP coverage picture: filter chip [All / Local / Remote / Connection-only], per-row MCP type badges, parent/child grouped tool sections, OAuth client setup card with the authorized-redirect-URI derived live from your install. See PR #2 in `pinkfish-connections-admin-app` for the full diff.

### Breaking / behavior changes

- `GET /catalog` now requires a valid JWT. (Earlier development builds during PIN-6414 were unauthenticated; this is closed in 0.2.0.)
- The `?serverTypes=` query parameter is new — older clients that pass it won't error (the param was silently accepted before), but they'll now see filtered responses.

### Upgrade notes

- Re-load the new MCPfarm tarball into your ECR and update the `mcpfarm-ecs` task definition's `ContainerImage` parameter to the new tag.
- Replace `pinkfish-connections-admin-app-main.zip` with the new one and re-extract. Your existing `keys/`, `.env`, and any local state are preserved if they live outside the zip's tree (they should).
- No PinkConnect change needed.
- No CloudFormation parameter changes — just the `ContainerImage` value points at the new MCPfarm tag.

---

## 0.1.0 — 2026-05-15

Initial customer self-host release.

### Components

| Component | Version | Notes |
|---|---|---|
| MCPfarm | (not bundled in 0.1.0; deferred per PIN-6389 scope) | Operators ran PinkConnect-only |
| PinkConnect | `pinkconnect-3a0863ee1167.tar.gz` | Built from platform commit `3a0863ee1167` |
| Admin app | `pinkfish-connections-admin-app-main.zip` (internal `package.json` `version: 0.1.0`) | PIN-6409 — Auth + Tools + MCP tabs |
| CloudFormation templates | `pinkconnect-networking.yaml`, `pinkconnect-ecs.yaml`, `pinkconnect-docdb.yaml`, `pinkconnect-cdn.yaml`, `pinkconnect-backup.yaml` | Smoke and production install profiles |

### What shipped

- PinkConnect ECS stack (single-AZ smoke, multi-AZ production).
- DocDB-backed credential storage.
- JWT-verified `/connect/{service_key}/{conn_id}/{path}` proxy.
- Admin app to mint JWTs, configure claims, and exercise the proxy.
- Customer-facing install docs: smoke (`install-smoke.md`) and production (`install-production.md`).
- Teardown, troubleshooting, gotchas, parameter reference, alternate-components docs.
