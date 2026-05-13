# PinkConnect — Self-Host

PinkConnect deployed to your own AWS account.

## Install

The install is orchestrated by Claude. Drop this repo into a Claude
Code session and tell it:

> *"Install PinkConnect into my AWS account following claude-setup.md."*

Claude will ask four up-front questions (AWS profile, region, domain,
binary files), then drive the AWS commands itself. Takes ~30 minutes
end-to-end; ~$150/mo to keep running.

**Before starting,** you need:

- An AWS account with admin access, and an AWS CLI profile for it.
- A domain in Route53 in that account (PinkConnect lives on a
  subdomain — e.g. `connect.example.com`).
- Two binary artifacts from Pinkfish, dropped into the repo root:
  - `pinkconnect-<version>.tar.gz` — the container image
  - `pinkfish-connections-admin-app-main.zip` — the admin app

  Email **pf-support@pinkfish.ai** to get them if you don't have
  them already.

If you'd rather drive the install yourself rather than through Claude,
[`claude-setup.md`](./claude-setup.md) is a working human-readable doc
too — it's just denser than a normal install guide.

## Verify it's running

When the install finishes, PinkConnect responds on the domain you
chose. One curl confirms everything is up:

```bash
curl -i https://connect.<your-domain>/health/ready
```

Expected:

```
HTTP/2 200
content-type: application/json; charset=utf-8

{"status":"ready"}
```

- **200 `ready`** — service is up, MongoDB reachable, all secrets
  resolved. You're done.
- **503** — the container started but a config value is missing.
  Check the `/ecs/pinkconnect` CloudWatch log group for the
  `mcp.server.config.invalid` line, which lists exactly what's wrong.
- **DNS doesn't resolve** — the ACM cert or Route53 A-alias didn't
  land. Re-check phases 6 and 7 of `claude-setup.md`, or ask Claude
  to diagnose.

## What's next

Once the smoke test passes you have a working PinkConnect at
`https://connect.<your-domain>`. To actually use it, you need to (a)
mint signed JWTs from your own app or the bundled admin app, and
(b) deploy + connect at least one service. `claude-setup.md` § 9
walks through an OpenWeather example end-to-end as a smoke test.

## Support

**pf-support@pinkfish.ai** — same address for support, binary
deliveries, and license questions.
