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

  Contact Pinkfish to get them if you don't have them already.

If you'd rather drive the install yourself rather than through Claude,
[`claude-setup.md`](./claude-setup.md) is a working human-readable doc
too — it's just denser than a normal install guide.

Want to swap in your own database, run on Kubernetes instead of
Fargate, or skip the ALB? See
[`alternate-components.md`](./alternate-components.md) — the container's
actual runtime contract is narrower than what `claude-setup.md`
provisions.

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

## Make a proxy call

Assuming you've followed `claude-setup.md` § 9 to deploy the
`openweather` service and create a connection with your API key, this
is how you hit the real API through PinkConnect:

### 1. Mint a JWT

PinkConnect verifies user JWTs signed by the keypair you generated
during install. The bundled admin app already has `jsonwebtoken`
installed, so the cheapest mint is a node one-liner from inside that
directory:

```bash
TOKEN=$(cd pinkfish-connections-admin-app-main && node -e "
const jwt = require('jsonwebtoken');
const fs = require('fs');
const key = fs.readFileSync('keys/private.pem');
console.log(jwt.sign(
  { type: 'user', providerId: 'local-dev-user', selectedOrg: 'local-dev-org' },
  key,
  { algorithm: 'RS256', expiresIn: '1h' }
));
")
echo "$TOKEN"
```

**About `providerId` and `selectedOrg`.** These are the multi-tenant
identity claims PinkConnect uses to scope every connection. Think of
them as the user identifier and the org/tenant they belong to — every
connection record in the database is owned by a `(selectedOrg,
providerId)` pair, and the proxy only returns connections that match
the claims in the JWT you present.

In production these come from your own auth system: when an end user
in org `acme-co` named user `u_42` opens your app, you mint a JWT with
`selectedOrg: "acme-co", providerId: "u_42"`, and from then on every
connection that user creates belongs to that pair. Another user in the
same org has a different `providerId` and sees a different set of
connections; a user in a different org sees nothing of acme-co's.

In this smoke test those values are hardcoded as `local-dev-user` /
`local-dev-org` (the defaults the admin app ships with in
`.env.example`), so the connection you created during § 9 belongs to
that pair. If you change them in the JWT you mint, PinkConnect treats
you as a different user and `/manage/user-connections` returns an
empty list — your connection is still there, just owned by the other
identity. Match the values you used when creating the connection.

### 2. Find the connection ID

```bash
curl -s -H "auth-token: $TOKEN" \
  https://connect.<your-domain>/manage/user-connections | jq
```

Returns an array; grab the `id` for your openweather connection (it
also shows `identifier` like `0786eb07****` so you can tell which key
it is).

### 3. Make the proxy call

PinkConnect's proxy URL pattern is:

```
GET https://connect.<your-domain>/connect/<service_key>/<connection_id>/<upstream-path>
```

For OpenWeather:

```bash
CONN_ID=<from step 2>
curl -s -H "auth-token: $TOKEN" \
  "https://connect.<your-domain>/connect/openweather/${CONN_ID}/data/2.5/weather?lat=44.34&lon=10.99" \
  | jq
```

Expected: a real OpenWeather JSON response (`{"coord":{...},"weather":[...],"main":{"temp":...}}`).
PinkConnect decrypted your stored API key, injected it as `appid` on
the upstream request per the openweather service definition, and
returned the response.

The same pattern works for every service in the catalog — swap
`openweather` for the `service_key` and the path for whatever that
provider exposes.

## Support

Contact Pinkfish for support, binary deliveries, and license questions.
