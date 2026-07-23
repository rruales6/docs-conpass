# Backlog

Deferred items — reviewed during Phase 7 planning and explicitly **out of the current
MVP scope** per product decision. Kept here so nothing is lost. Not scheduled.

## Low priority
- **Async wallet push off the cashier path.** Today the backend's post-commit
  `provider.update()` on accrue/redeem runs synchronously (best-effort). Move it to an
  async worker (SQS/EventBridge) so a slow Google Wallet call can never affect the
  in-store `<3s` operation budget. _(Deferred from Phase 7 #5 — replaced in-scope by the
  global loading animation.)_

## Medium priority
- **Lambda SnapStart.** Would cut cold-start init latency, but for this stack it's poor
  ROI right now: Python SnapStart is **x86-only** (our Lambdas are arm64/Graviton — would
  lose ~20%), it's **no longer free for Python** (per-version caching, min 3h, + per-restore
  charge), and it needs **published versions/aliases** (httpApi rewiring). Revisit if
  traffic grows and cold starts become the dominant cost. Cold starts are meanwhile reduced
  by the re-slim + memory bump done in Phase 7. _(Phase 7 #3 — deferred after investigation.)_
- **Google Wallet "pass saved" callback → `cards.wallet_installed`.** Nothing flips
  `wallet_installed` today, so the *active installed passes / installs-this-week* metrics
  (and the billing metric) are approximated as active cards. Wire the Google Wallet save
  callback to set the flag for real. _(Phase 7 #6.)_
- **Secrets → AWS Secrets Manager / SSM.** The service-account JSON and Supabase secret
  key are injected as Lambda env vars (under the 4 KB limit). Move to a managed,
  rotatable store. _(Phase 7 #9.)_
- **Least-privilege IAM + credential rotation.** Replace the temporary AWS admin with the
  scoped policies already in `docs/aws/`; rotate the seeded demo/admin passwords for prod.
  _(Phase 7 #10.)_
- **Re-enable backend CI + observability.** CI was disabled on request (`ci.yml.disabled`).
  Re-enable, add Schemathesis contract fuzzing, CloudWatch alarms, structured error
  tracking. _(Phase 7 #11.)_

## Low priority (polish)
- **PWA "new version available" prompt.** Complements the network-first service-worker
  fix — surface a "refresh for the latest" toast when a new build is detected.
  _(Phase 7 #13.)_

---
Deferred features (separate from Phase 7 hardening): **birthday automation**
(`PUT /programs/{id}/birthday-automation`, `POST /birthday-cards`) and **notifications**
(`POST /notifications/reminders`) remain `501`.
