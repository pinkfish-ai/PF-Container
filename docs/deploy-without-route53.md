# Deploy PinkConnect without Route 53

The default `install.md` flow assumes Route 53 owns the target
domain — `aws acm request-certificate` validates via DNS records in
the hosted zone, and the ECS stack creates an A-alias from
`CustomDomainName` to the ALB.

Some organizations can't use Route 53 because of central DNS policy
(e.g. `accenture.com` endpoints live in an internal IPAM / external
DNS provider). This walkthrough covers the alternative install path:

- DNS managed externally — you bring your own CNAME pointing at the
  ALB's AWS-managed hostname.
- TLS cert issued by your organization's CA — imported into ACM
  instead of provisioned by AWS.

Everything else (networking, DocumentDB, SSM, ECS, JWT, smoke test)
runs the same as the main install. The only differences are §1, §5,
and §6 below.

---

## Prerequisites

Same as `install.md` **except**:

- Skip the "domain in Route 53" prerequisite.
- You need a TLS certificate covering `<host>` (the public name you
  want PinkConnect served at), as three files:
  - `<host>.crt` — server certificate
  - `<host>.key` — matching private key
  - `<host>-chain.crt` — intermediate chain (optional but recommended)
- You need write access to your organization's external DNS to add a
  single CNAME record once the deploy completes.

---

## 1. Push the container image to ECR

No change from `install.md` §1.

---

## 2. Deploy networking

No change from `install.md` §2.

---

## 3. Deploy DocumentDB

No change from `install.md` §3.

---

## 4. Generate JWT keypair and populate SSM

No change from `install.md` §4.

---

## 5. Import the TLS certificate into ACM

Replaces `install.md` §5 (which does `aws acm request-certificate`).

```bash
export CERT_ARN=$(aws acm import-certificate \
  --certificate fileb://<host>.crt \
  --private-key fileb://<host>.key \
  --certificate-chain fileb://<host>-chain.crt \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query CertificateArn --output text)
```

Notes:

- The cert must be in the same region as the ALB (i.e. matches
  `AWS_REGION`).
- Cert subject (CN) or one of its SANs must match `<host>` exactly.
  Wildcards work (e.g. `*.accenture.com` covers
  `pinkconnect.accenture.com`).
- `--certificate-chain` is optional; provide it if your CA isn't
  publicly trusted (most internal CAs aren't).
- ACM doesn't auto-renew imported certs. Track expiry in your
  organization's cert lifecycle tooling. Renewal is the same command
  with new files; the ARN stays stable.

Verify:

```bash
aws acm describe-certificate --certificate-arn "$CERT_ARN" \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Certificate.{Type:Type,Status:Status,Domain:DomainName,Expires:NotAfter}'
# Type should be IMPORTED, Status ISSUED
```

---

## 6. Deploy ECS without the Route 53 alias

Replaces `install.md` §6.

The ECS template has a `CreateDnsRecord` parameter; set it to
`false`. The ALB still gets its AWS-managed hostname (returned via
the `AlbDnsName` output) — you'll point an external CNAME at it in
§6b.

```bash
# Same task SG / docdb SG race as the default install — see install.md §6.
# Kick off the deploy in the background:
aws cloudformation deploy \
  --stack-name pinkconnect-ecs \
  --template-file cloudformation/pinkconnect-ecs.yaml \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    VpcId=$VPC_ID \
    PublicSubnetAId=$PUB_A PublicSubnetBId=$PUB_B \
    PrivateSubnetAId=$PRIV_A PrivateSubnetBId=$PRIV_B \
    ContainerImage="$ECR_URI" \
    CustomDomainName="$HOST" \
    CallbackUrl="https://${HOST}/connect/callback" \
    CertificateArn="$CERT_ARN" \
    CreateDnsRecord=false &

# Authorize the task SG on the docdb SG as soon as it exists
until TASK_SG=$(aws cloudformation describe-stack-resources \
       --stack-name pinkconnect-ecs --region "$AWS_REGION" --profile "$AWS_PROFILE" \
       --query 'StackResources[?LogicalResourceId==`TaskSecurityGroup`].PhysicalResourceId' \
       --output text 2>/dev/null) && [ -n "$TASK_SG" ]; do sleep 5; done

aws ec2 authorize-security-group-ingress \
  --group-id "$DOCDB_SG" --source-group "$TASK_SG" \
  --protocol tcp --port 27017 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"

wait
```

Notice what's missing compared to `install.md` §6:

- No `Route53HostedZoneId=...` parameter (we set `CreateDnsRecord=false`,
  so the hosted-zone ID is unused — leave empty).
- No `HOSTED_ZONE_ID` lookup before the deploy.

The `DnsRecord` resource is skipped entirely when
`CreateDnsRecord=false`. The stack reaches `CREATE_COMPLETE` without
ever touching Route 53.

### 6a. Capture the ALB hostname

```bash
ecs_out() { aws cloudformation describe-stacks --stack-name pinkconnect-ecs \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue" --output text; }

export ALB_DNS=$(ecs_out AlbDnsName)
echo "ALB hostname: $ALB_DNS"
# e.g. pinkconnect-1706009482.eu-central-1.elb.amazonaws.com
```

### 6b. Create the CNAME in your external DNS provider

In your organization's DNS (Akamai, NS1, Cloudflare, internal IPAM,
etc.) create a single CNAME record:

```
Type:  CNAME
Name:  <host>             (e.g. pinkconnect.accenture.com)
Value: $ALB_DNS            (e.g. pinkconnect-1706009482.eu-central-1.elb.amazonaws.com)
TTL:   300                 (anything reasonable; lower while testing)
```

Wait for propagation (usually under TTL, often seconds for modern
providers):

```bash
dig +short CNAME "$HOST"
# Should print the ALB hostname.
```

> **Why CNAME, not A?** ALB IPs rotate. A-records get stale. CNAME
> follows the ALB's own DNS, which AWS keeps current. (Route 53
> A-aliases work too because they're a Route-53-only special-case
> that internally resolves the ALB DNS at query time; external DNS
> providers don't have that primitive.)

> **Apex (`accenture.com` itself, not a subdomain)?** A CNAME at the
> zone apex isn't valid per RFC 1035. Either deploy on a subdomain
> (recommended) or use your DNS provider's apex alias / ALIAS / ANAME
> record type, which most modern providers support.

---

## 7. Verify

No change from `install.md` §7. Hit `https://${HOST}/health/live`
and `https://${HOST}/health/ready` — both return 200.

---

## 8. Smoke test with a real connection

No change from `install.md` §8.

---

## Teardown

No change from `teardown.md`, except:

- There's no Route 53 hosted zone to keep — the only thing that
  persists outside the stack is the imported ACM certificate. Delete
  it after teardown if you don't plan to redeploy:
  ```bash
  aws acm delete-certificate --certificate-arn "$CERT_ARN" \
    --region "$AWS_REGION" --profile "$AWS_PROFILE"
  ```
- Remember to also remove the CNAME you created in §6b from your
  external DNS provider.

---

## Troubleshooting

| Symptom | Cause + fix |
|---------|------------|
| ECS stack fails with `Parameter validation failed: 'Route53HostedZoneId'` | You're on a template build that still types `Route53HostedZoneId` as `AWS::Route53::HostedZone::Id`. Pull the latest `cloudformation/pinkconnect-ecs.yaml` — recent versions type it as `String` with `Default: ''`. |
| Browser shows TLS warning ("Not Secure" / unknown issuer) on `https://<host>` | Imported cert's chain is missing or doesn't link back to a CA your browser trusts. Re-run `aws acm import-certificate` with `--certificate-chain` pointing at the full intermediate bundle. |
| `curl https://<host>/health/live` works from inside the VPC but external clients time out | The CNAME isn't propagated yet (or doesn't exist). Verify with `dig +short CNAME <host>`. |
| `curl: (60) SSL: no alternative certificate subject name matches target host name` | The imported cert's CN/SAN doesn't cover `<host>`. Re-issue with a SAN for the exact hostname (or a covering wildcard) and re-import. |
| Health checks 200 but the ALB target group is unhealthy | Default health-check path is `/health/ready`. Container's `validateStartupConfig` returns 503 on `/health/ready` until every required SSM param is reachable. Check CloudWatch log group `/ecs/pinkconnect` for the `mcp.server.config.invalid` line — it lists what's missing. |
