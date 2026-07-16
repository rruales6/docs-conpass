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
scripts/deploy.sh conpass prod             # exports SSO temp creds + injects Supabase secrets
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

## Rollback
`AWS_PROFILE=conpass npm run remove` tears the stack down cleanly (nothing outside the
`conpass-api-*` prefix is touched).
