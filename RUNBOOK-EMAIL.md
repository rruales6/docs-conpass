# Email runbook — transactional email from the Lambdas (Phase 15)

conpass sends email itself through **one generic SMTP sender** in the shared layer
(**D18**). Today it sends one kind of message: **signup credentials**. When a merchant
subscribes, the owner gets a temporary password by email, and the PWA makes them replace
it at first login. Platform-admin can resend credentials from Función 04.

## How it works

```
POST /merchants ─▶ provision owner (Supabase Auth, app_metadata.must_change_password=true,
                   user_metadata.language=es|en)
               ─▶ get_email_sender().send(credentials_email(...))   ── smtp.hostinger.com:465
               ─▶ 201 {ownerInviteSent: true|false, ownerEmail}      (never the password)

First login ─▶ /me says mustChangePassword ─▶ PWA shows only /change-password
           ─▶ POST /me/password {newPassword}  (sets the password + clears the flag in one
              admin update) ─▶ PWA signs in again with the new password ─▶ role home

Función 04 ─▶ POST /admin/clients/{id}/resend-credentials
           ─▶ new temp password + flag set again ─▶ email (same template, "resend" wording)
```

| Piece | Where |
|---|---|
| Sender (`EmailSender`, `SmtpEmailSender`, `StubEmailSender`) | `backend/layers/common/python/conpass_common/providers/email.py` |
| Choosing the sender | `providers/factory.py` → `get_email_sender()`, keyed on `CONPASS_EMAIL_PROVIDER` (`stub` by default, `smtp` in the deployed Lambdas) |
| Settings | `conpass_common/config.py` (`smtp_*`, `email_from_name`, `email_reply_to`, `console_login_url`) |
| Credentials email (ES/EN) | `conpass_common/email_templates.py` → `credentials_email()` |
| Flag + temp password helpers | `conpass_common/provisioning.py` → `provision_user(must_change_password=…)`, `issue_temp_password()`, `complete_password_change()` |
| Endpoints | `POST /merchants` (merchants), `GET /me` + `POST /me/password` (identity), `POST /admin/clients/{merchantId}/resend-credentials` (admin) |
| PWA | `auth/AuthContext.tsx` (`mustChangePassword`, `changePassword`), `pages/auth/ChangePasswordPage.tsx`, signup `ActivationScreen`, admin *Gestionar cuenta* → *Reenviar credenciales* |

## Settings

| Setting | Value | Where it comes from |
|---|---|---|
| Host | `smtp.hostinger.com` | code default; override with `SMTP_HOST` or `conpass-no-reply-sender.host` |
| Port | `465` (implicit TLS). `587` switches the sender to STARTTLS | code default; override with `SMTP_PORT` or `conpass-no-reply-sender.port` |
| Username = From | `no-reply@conpass.cards` | `secrets.yaml` → `conpass.conpass-no-reply-sender.email` |
| Password | the mailbox password | `secrets.yaml` → `conpass.conpass-no-reply-sender.password` |
| From name / Reply-To | `conpass` / `info@conpass.cards` | code defaults |
| Login link in the email | `https://console.conpass.cards/login` | code default (`CONPASS_CONSOLE_LOGIN_URL`) |

`scripts/with_env.py` copies the username and password from `secrets.yaml` into the deploy
environment, and `serverless.yml` passes them to every Lambda as `SMTP_USERNAME` /
`SMTP_PASSWORD` (plain env vars, the same posture as the Supabase secret key; moving secrets
to SSM is in the backlog). Nothing is printed or logged: the sender logs only the
recipient and subject, never a body, because the body holds a password.

DNS for conpass.cards, checked 2026-11-05:
- MX: `mx1/mx2.hostinger.com`
- SPF: `v=spf1 include:_spf.mail.hostinger.com ~all`
- DKIM: `hostingermail-a._domainkey`
- DMARC: `v=DMARC1; p=none`

## Limits

- **Hostinger caps sending per mailbox** over a rolling 24 hours: about 100 per day on a
  free or trial mailbox, 1 000 on Business Starter, 3 000 on Business Premium. Check
  hPanel → Emails → View limits. Signups are low volume.
- **There is no retry queue.** A failed signup email leaves `ownerInviteSent=false`, and
  admin resends.
- **Each resend issues a new password.** Only the latest password works. The admin button
  is disabled while a resend is in flight.
- **The SMTP step has a 6 s socket timeout**, so an unreachable relay cannot use up the
  15 s Lambda timeout.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Signup screen says the email could not be sent | Relay unreachable, wrong password, or Hostinger daily cap reached | CloudWatch for the merchants Lambda: look for `owner_credentials_email_failed` / `email_send_failed` (error class only). Check the mailbox in hPanel, then resend from Función 04 |
| Resend answers **503** | `SMTP_USERNAME`/`SMTP_PASSWORD` missing in the Lambda env | Confirm both keys exist in `secrets.yaml`, then redeploy with `bash scripts/deploy.sh conpass prod --force` |
| Resend answers **502** | The password **was reset** but the email failed | Retry. Each attempt issues a new password, so the failed one never worked anyway |
| Resend answers **409** | It is the demo merchant | By design: the demo uses a shared password |
| Email lands in spam | New mailbox, DMARC at `p=none` | Mark it as not spam once. Later, move DMARC to `p=quarantine` once reports look clean (backlog) |
| Owner never sees the change-password page | They signed in before the flag was set, or with an old token | Sign out and sign in again; `/me` returns `mustChangePassword` from the token |

## Switching provider

Only settings change:
- **Amazon SES (SMTP interface):** verify conpass.cards in SES (3 DKIM CNAMEs in Hostinger
  DNS), request production access, and create SMTP credentials. Then set
  `SMTP_HOST=email-smtp.us-east-1.amazonaws.com`, `SMTP_PORT=587`, and the SES SMTP
  username/password. Cost: $0.10 per 1 000 emails.
- **Brevo:** `smtp-relay.brevo.com:587` with its SMTP key. The free plan allows 300 per day.

Add the provider's SPF include to the conpass.cards TXT record when you switch.

## Not covered yet (backlog)

- forgot-password (`/forgot-password`, `/reset-password` are still stubs)
- emails for operation users (their temporary password is still shown in the panel)
- a server-side forced-change gate
- bounce handling
- SSM for the SMTP password
