# AWS setup (you apply this — I do not touch AWS until it's in place)

Two least-privilege policies, so the deploy identity can do **only** what deployment needs.

## 1. Create the deploy user
1. IAM → Users → **Create user** `conpass-deployer` (no console access needed).
2. Create two customer-managed policies from the JSON in this folder, replacing
   `ACCOUNT_ID` with your 12-digit account id (region is pinned to `us-east-1`):
   - [iam-backend-deploy-policy.json](iam-backend-deploy-policy.json) → `conpass-backend-deploy`
   - [iam-frontend-deploy-policy.json](iam-frontend-deploy-policy.json) → `conpass-frontend-deploy`
3. Attach both to `conpass-deployer`.
4. Create an **access key** (Application running outside AWS) and give me the pair, or add
   to `secrets.yaml` under `conpass.aws`:
   ```yaml
   conpass:
     aws:
       region: us-east-1
       access_key_id: AKIA...
       secret_access_key: ...
   ```

## Scope of each policy
- **backend-deploy** — Serverless Framework v3 deploy of the API: CloudFormation for the
  `conpass-api-*` stack, its S3 deployment bucket (`conpass-api-*`), Lambda functions +
  the `conpass-*` layers, HTTP API Gateway, function IAM roles (`conpass-api-*` only,
  incl. `PassRole`), and `/aws/lambda/conpass-api-*` log groups. Nothing outside the
  `conpass-api-*` name prefix.
- **frontend-deploy** — the `conpass-cards*` site bucket, CloudFront (no resource-level
  ARNs exist for CloudFront/ACM, so `*` there), ACM cert, and an optional Route53
  statement (delete it if `conpass.cards` DNS is on Cloudflare/registrar).

## Notes / caveats
- Serverless Framework is broad by nature; if a deploy fails on a missing action, tell me
  the exact `AccessDenied` and I'll add that one action (keeping the prefix scoping).
- Prefer this over `AdministratorAccess`. If you want to unblock a first deploy fast, you
  *can* attach `AdministratorAccess` temporarily and swap to these after — your call.
- ACM certs for CloudFront **must be in us-east-1** (they are here).
- Deploy prerequisites on the build machine: Docker running (✓), and
  `npm i` in `backend/` for the Serverless plugins.
