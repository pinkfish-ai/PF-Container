# PinkConnect — Troubleshooting

Symptom → cause + fix. Read this when something breaks during or
after install. For the things that are *predictable* failure modes
worth knowing in advance, see `gotchas.md`.

| Symptom | Cause + fix |
|---|---|
| Stack stuck `CREATE_IN_PROGRESS` on `Service` for ~15 min, then rolls back | The SG wire-up didn't happen during the ECS deploy window. See the SG step in `../install.md` / `../wip/install-production.md` — authorize while the deploy is still running, not after. |
| `ResourceInitializationError: invalid ssm parameters: /pinkconnect/...` | Missing SSM param or task exec role can't decrypt. Re-run the matching `put-parameter` step from the install doc; ECS auto-recovers on its next launch attempt. |
| Container restart loop, log shows `Missing required environment variables` or `mcp.server.config.invalid` | Same root cause as above; the structured `mcp.server.config.invalid` line lists every missing env var. |
| `MongoServerSelectionError` / connect timeout from container logs | Task SG isn't authorized on the DocDB SG. Re-run the `authorize-security-group-ingress` step. |
| `MongoNetworkError` with TLS handshake failure | Mongo URI is missing `tlsCAFile=/app/global-bundle.pem`. The bundle is baked at that exact path in the image; the URI option must match verbatim. |
| ALB target health check failing but container looks fine in logs | Health check is `GET /health/ready`. Returns 503 until env vars resolve. Look for `mcp.server.config.invalid` in CloudWatch. |
| OAuth callback fails with `invalid_redirect` | The OAuth app's registered redirect URI doesn't match `https://<host>/connect/callback`. Update in the provider's developer console. |
| OAuth consent succeeds but the connection lands in `error`; logs show `ERR_CRYPTO_INVALID_KEYLEN` / `Error storing access token` | `/pinkconnect/token-encryption-key` is the wrong format. The container decodes it as **hex** (`Buffer.from(key,'hex')`) for an AES-256-GCM key, so it must be 64 hex chars — generate it with `openssl rand -hex 32`, **not** `-base64`. API-key connections are unaffected (they use `oauth-encryption-key`, which is SHA-256-hashed and accepts any string). Update the SSM param and `aws ecs update-service --force-new-deployment` so the task reloads it. |
| `core_services` collection empty after first deploy | Auto-seed runs idempotently at boot. Restart the task or `POST /admin/services/sync`. |
| `cloudformation delete-stack` hangs on networking | NAT gateway / EIP releases can take ~5 min. If it stays stuck longer, check for orphaned ENIs in the VPC. |
| CloudFront returns 502 from CloudFront origin | Most common cause: CloudFront → ALB TLS verification failed because the ALB cert doesn't cover the host CloudFront is sending. The CDN stack uses `AllViewerExceptHostHeader` so CloudFront resets Host to `OriginHostname` — confirm the ALB cert covers `OriginHostname`. Second most common: the ECS stack was deployed without `OriginHostname` set, so no A-alias exists for it and DNS resolution fails. |
| `aws backup start-backup-job` succeeds but no recovery point in destination vault | `start-backup-job` creates an ad-hoc recovery point that bypasses the plan's `CopyAction`. To force cross-region copy on-demand, run `start-copy-job` explicitly with `--source-backup-vault-name` and `--destination-backup-vault-arn`. The plan's `CopyAction` only fires on scheduled runs. |
| `aws backup` job fails with `AccessDenied` even though the IAM role exists | The role's trust policy must allow `backup.amazonaws.com`. The `pinkconnect-backup` stack's `BackupServiceRole` sets this; if you're using a different role, verify the trust policy. |
