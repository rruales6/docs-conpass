# Deploy runbook

## Prerequisites (done)
- Serverless Framework 3 + plugins installed in `backend/` (`npm install`). ✓
- Docker running (for the ARM64 dependency layer build). ✓
- Secrets in `secrets.yaml` under `conpass.*`; injected at deploy time by
  `scripts/with_env.py` (Supabase + Google values never touch shell history). ✓
- Migrations already applied to Supabase. ✓

## AWS auth (you do this)
Serverless reads credentials from `~/.aws` (profile or SSO cache) or the environment —
it does **not** need the `aws` binary on this shell's PATH. Authenticate with the admin
identity however you normally do:

- **Access keys:** `aws configure --profile conpass` → paste key/secret, region `us-east-1`.
- **SSO:** `aws sso login --profile conpass`.

Then tell me the **profile name** (or say "default").

## Deploy (I run this once you've logged in)
```bash
cd backend
scripts/deploy.sh conpass prod --force     # exports temp creds + injects Supabase secrets
```
Anything after the stage is forwarded to `serverless deploy`. Use **`--force`** whenever the
change touches `serverless.yml` only (new route, bucket, env var, IAM) — serverless otherwise
decides nothing changed and silently no-ops. This has bitten CORS, Phase 2, and Phase 8.

**Do not trust the deploy's exit code or its last log line.** `serverless deploy | tail` makes
`$?` the exit of `tail` (always 0), and the log can end mid-stream while CloudFormation is still
finishing. Verify against the stack instead — and pass the region explicitly, because exported
credentials fall back to the `~/.aws` **default** region (us-east-2 here), which reports the
stack as nonexistent:

```bash
AWS_REGION=us-east-1 aws cloudformation describe-stacks \
  --stack-name conpass-api-prod --query 'Stacks[0].[StackStatus,LastUpdatedTime]' --output text
```
Note: the `conpass` profile is SSO/`login_session`-based, which Serverless's AWS SDK v2
can't resolve directly (`AWS profile "conpass" doesn't seem to be configured`).
`scripts/deploy.sh` works around this by running `aws configure export-credentials` and
passing the resolved temporary credentials as env vars. (Plain `npm run deploy` only works
with a static-key profile.) SSO sessions expire — re-run `aws sso login`/`aws login` if a
deploy fails with an expired-token error.
Outputs the HTTP API base URL. I then:
1. Smoke-test `GET /health` and a signed `GET /me`.
2. Point `api.conpass.cards` at it (ACM cert in us-east-1 + custom domain + DNS).
3. Build & upload the PWA to S3 + CloudFront for `conpass.cards`, with
   `VITE_API_BASE_URL=https://api.conpass.cards`.

## S3 buckets (created by `serverless.yml`, not by hand)

| Bucket | Access | Contents | If you break it |
|---|---|---|---|
| `conpass-program-assets-${stage}` | **public-read** (bucket policy) | program icon/background, payment QR under `platform/` | wallet passes and the signup QR stop loading |
| `conpass-payment-proofs-${stage}` | **private** — all four public-access blocks `true` | transfer receipts under `payment-proofs/` | receipts become world-readable; see D11 |

Both carry a `CorsConfiguration` allowlist for browser uploads (presigned PUT). **Adding a new
frontend origin means updating both**, plus the API's `httpApi.cors.allowedOrigins` and
`config.cors_origins` — a missing origin fails only in the browser, as an opaque CORS error.
Sanity check after a deploy that touches them:

```bash
AWS_REGION=us-east-1 aws s3api get-public-access-block --bucket conpass-payment-proofs-prod
```
All four flags must be `true`.

## Rollback
`AWS_PROFILE=conpass npm run remove` tears the stack down cleanly (nothing outside the
`conpass-api-*` prefix is touched).
