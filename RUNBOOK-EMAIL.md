# Email setup — sending signup credentials (Phase 14)

Goal: when a merchant subscribes, the owner gets an email from conpass with a link to
**set their own password**. A password should never travel in an email; the current
on-screen temporary password stays as a fallback. The same setup also makes
"¿Olvidó su contraseña?" work, because that page is still a dead stub today.

conpass.cards mail is already hosted on **Hostinger**: its MX records point to
`mx1/mx2.hostinger.com` and its SPF record includes `_spf.mail.hostinger.com`.

## Recommended: Supabase Auth → Hostinger SMTP

Supabase Auth already knows how to send "invite" and "reset password" emails with a
secure one-time link. Out of the box it sends through Supabase's own mailer, which is
limited to **about 2 emails per hour** and is not meant for production. You point it at a
Hostinger mailbox instead. Then:

- the Lambdas never hold an email password: it lives only in the Supabase dashboard;
- there is no new AWS service, IAM permission or library;
- signup emails and forgot-password emails share one path.

### Step by step (owner)

1. **Create the sender mailbox.** In hPanel go to **Emails**, open conpass.cards, and
   create `no-reply@conpass.cards` with a strong password. Then go to **Emails →
   Mailboxes → View limits** to see its daily sending cap.
   - Hostinger caps each mailbox per rolling 24 hours: free/trial mailboxes 100/day,
     Business Starter 1 000/day, Business Premium 3 000/day.
   - Signups are low volume, so any of these is enough to start.
2. **Authenticate the domain** so the mail doesn't land in spam. In hPanel go to
   **Emails → conpass.cards → DNS settings / Connect domain**:
   - SPF: already present (`v=spf1 include:_spf.mail.hostinger.com ~all`). Leave it.
   - DKIM: enable it, or add the `hostingermail-*._domainkey` CNAME/TXT records hPanel shows.
   - DMARC: add a TXT record named `_dmarc` with the value
     `v=DMARC1; p=none; rua=mailto:info@conpass.cards`.
3. **Point Supabase at Hostinger.** In the Supabase dashboard go to **Authentication →
   Emails → SMTP Settings**, enable custom SMTP, and enter:
   - Host `smtp.hostinger.com`, port `465` (SSL/TLS)
   - Username `no-reply@conpass.cards`, plus that mailbox's password
   - Sender email `no-reply@conpass.cards`, sender name `conpass`
4. **Raise the auth email rate limit.** In **Authentication → Rate Limits**, set
   "emails sent per hour" to 30–100. That stays well inside Hostinger's daily cap.
5. **Allow the link to come back to the app.** In **Authentication → URL
   Configuration**:
   - Site URL: `https://console.conpass.cards`
   - Redirect URLs: add `https://console.conpass.cards/reset-password`
6. **Tell Claude it's done.** The code side is then:
   - onboarding asks Supabase to email the owner a set-password link;
   - `/reset-password` accepts that link;
   - `/forgot-password` sends one;
   - `ownerInviteSent` becomes truthful (it is hard-coded `true` today);
   - the email templates (invite and reset) are written in Spanish, with English
     variants, in conpass branding.

No secrets change in `secrets.yaml`. The Hostinger password is typed only into Supabase.

## Alternatives

| Option | Good for | What it needs | Trade-off |
|---|---|---|---|
| **Amazon SES** (directly from the Lambda, or as Supabase's SMTP) | High volume and best deliverability; already inside the AWS account | Verify conpass.cards in SES (3 DKIM CNAMEs added in Hostinger DNS), then **request production access** (an AWS review, usually about a day). The Lambda needs IAM `ses:SendEmail`. | About $0.10 per 1 000 emails, and more setup steps. Until production access is granted, SES only sends to verified addresses. |
| **Resend / Brevo** (as Supabase's SMTP, or by API) | Nice dashboards, logs and templates | An account, their DNS records in Hostinger, and an SMTP password or API key | One more vendor. Free tiers are capped per day/month (check their current pricing). |
| **Hostinger SMTP directly from the Lambda** | Sending custom emails beyond auth (receipts, campaign notices) | The mailbox password stored in `secrets.yaml` and the Lambda env | A mailbox password ends up in AWS config, and the Lambda has to build its own reset-link flow. Not worth it for credentials alone. |

**Path forward:** start with Supabase + Hostinger, which needs no new vendor and no
code-side secret. If volume grows or deliverability suffers, swap only the SMTP
credentials in step 3 to SES or Brevo. Nothing in the code changes.
