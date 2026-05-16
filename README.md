# PinkConnect + MCPfarm — Self-Host

Run the Pinkfish stack inside your own infrastructure — your AWS account, your IAM, your encryption keys, your network boundary. **15,783 MCP tools across 344 SaaS integrations**, all behind a single ECS service you control.

**Current bundle version:** see [`VERSION`](./VERSION) — currently `0.2.0`.
**What changed since 0.1.0:** [`RELEASE-NOTES.md`](./RELEASE-NOTES.md).

---

## Quick start

**1. Drop three binary artifacts from Pinkfish into this directory:**

- `pinkconnect-<version>.tar.gz` — PinkConnect container image
- `mcpfarm-<version>.tar.gz` — MCPfarm container image (skip if you don't need MCP)
- `pinkfish-connections-admin-app-main.zip` — the admin app

**2. Open this repo in a Claude Code session, then paste:**

```
install
```

That's it. Claude auto-loads [`CLAUDE.md`](./CLAUDE.md), reads its orchestrator playbook in [`claude-setup.md`](./claude-setup.md), asks you 6 things (AWS profile, region, domain, install profile, include MCPfarm, rate-limiter backend), then walks through [`install.md`](./install.md) step by step. ~30-45 minutes end-to-end.

> Want to drive it yourself without Claude? Skip step 2 — open [`install.md`](./install.md) and run the commands top-to-bottom. Same procedure, same result.

---

## What's here

| File | Purpose |
|---|---|
| [`install.md`](./install.md) | **The install runbook.** Single source of truth — Claude follows it, humans follow it. |
| [`teardown.md`](./teardown.md) | Delete everything when you're done. |
| [`CLAUDE.md`](./CLAUDE.md) | *For Claude* — auto-loaded when this repo is open in a Claude Code session. Points Claude at the right files when you ask to install / tear down. You don't need to open it. |
| [`claude-setup.md`](./claude-setup.md) | *For Claude* — orchestrator playbook (the 6 questions, phase ordering, success criteria). Claude reads it via `CLAUDE.md`; you don't. |
| [`VERSION`](./VERSION) + [`RELEASE-NOTES.md`](./RELEASE-NOTES.md) | Which bundle version this is and what's in it. |
| [`docs/`](./docs/) | Reference docs the install + teardown point at when you need detail. |
| [`cloudformation/`](./cloudformation/) | CFN templates the install deploys. |
| [`wip/`](./wip/) | Production install profile — work-in-progress, not customer-ready yet. Contact Pinkfish if you need it. |

### `docs/`

| File | Purpose |
|---|---|
| [`docs/gotchas.md`](./docs/gotchas.md) | Non-obvious behaviors. **Read before deploying**, especially "SG wire-up timing". |
| [`docs/troubleshooting.md`](./docs/troubleshooting.md) | Symptom → cause + fix when something breaks. |
| [`docs/parameter-reference.md`](./docs/parameter-reference.md) | What every CFN parameter means. |
| [`docs/alternate-components.md`](./docs/alternate-components.md) | Swap-out playbook (Atlas instead of DocDB, EKS instead of Fargate, Cloudflare instead of CloudFront). |

---

## Before starting

- **AWS account** with admin access and an AWS CLI profile for it.
- **A domain** in Route53 in that account. PinkConnect lives on a subdomain (`connect.<your-domain>`); MCPfarm on another (`mcp.<your-domain>`). They must share the same Route53 hosted zone.
- **Docker** (Desktop or engine) — used once to load the tarballs and push to ECR.
- **AWS CLI v2**, configured. Verify with `aws sts get-caller-identity --profile <name>`.

---

## Verify it's running

When the install finishes, both services respond on their domains. One curl per layer confirms everything is up:

```bash
curl -i https://connect.<your-domain>/health/ready
# HTTP/2 200  {"status":"ready"}

curl -i https://mcp.<your-domain>/health/live
# HTTP/2 200  {"status":"live", ...}
```

Listing the catalog (with a JWT minted by the admin app) returns 812 MCP servers across the local + remote split:

```bash
TOKEN=$(curl -s http://localhost:3000/api/debug/jwt | jq -r .token)
curl -sH "Authorization: Bearer $TOKEN" \
  "https://mcp.<your-domain>/catalog?includeChildren=true" | jq 'length'
# 812
```

A real upstream call through PinkConnect (after creating an OpenWeather connection per [`install.md`](./install.md) §8):

```bash
curl "http://localhost:3000/api/proxy/openweather/${CONN_ID}/data/2.5/weather?lat=44.34&lon=10.99" | jq
# 200 with real OpenWeather JSON wrapped in {"output": {...}}
```

---

## Support

Contact Pinkfish for binary deliveries, install help, and license questions.
