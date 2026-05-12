# XCloud Inbound Email Verification — Deployment Runbook

This document covers the post-deploy steps and verification scenarios for the
reverse-OTP / inbound-email verification system that replaces the outbound
SMTP flow.

## Overview

Users prove email ownership by sending a message from their address to a
shared inbox (`xcloud@retrivo.ir`) with a token of the form `XV-XXXXXXXX` in
the **Subject** line. A cron job + on-demand manual button scan the inbox,
validate the message (SPF/DKIM stamped by our MTA), match the token against
pending `email_otps`, consume it, and apply the payload (create user / grant
reset authorization).

## Deployment order

1. **Backup** the current `index.php` and DB (admin → DB Backup).
2. **Pull** the new `index.php`. `lib/PHPMailer/` no longer exists — only
   `lib/.htaccess` remains.
3. **Load** the site once as the admin. Migrations apply automatically:
   - new table `email_inbound_processed`
   - new `inbound_*` settings rows
   - SMTP rows are removed from `settings`
4. **Configure inbound** (admin → "📨 دریافت ایمیل" in the nav):
   - Host (`mail.retrivo.ir`), port (`993`), encryption (`ssl`)
   - Username (`xcloud@retrivo.ir`) + password
   - MTA hostname (the hostname your provider stamps in `Authentication-Results:` —
     send a test mail to the inbox, open its source, copy the hostname before
     the first `;`)
   - Folder names: keep defaults unless you know what you're doing
   - Cron secret: tick "تولید خودکار" and save
5. **Test connection**: click "آزمایش اتصال". Should show `ext/imap` status,
   confirm credentials, and list unread count.
6. **Add the cPanel cron job**:
   ```
   * * * * * curl -fsS 'https://xcloud.retrivo.ir/?action=inbound_cron&token=YOUR_SECRET' > /dev/null 2>&1
   ```
7. **Wait 2 minutes**, then refresh the Inbound Settings page. The "آخرین اجرا"
   timestamp should be fresh.
8. **Test register** with a real email (see TC1 below).
9. **Lock** the setup wizard (`?action=setup_wizard` → step 4).

## Architecture notes

- **Token format**: `XV-` + 8 chars from `ABCDEFGHJKMNPQRSTUVWXYZ23456789`
  (Base32-ish, no ambiguous chars). Stored as `sha256(pepper|UPPERCASE_TOKEN)`.
- **TTL**: 30 min (controlled by `otp_ttl_minutes`).
- **Detection layers**:
  1. **Cron** (per-minute) — primary path, processes up to 50 messages/run
  2. **Manual button** on step2 page — rate-limited 1/15s per session
  3. **JS poller** on step2 page — every 5s, just reads OTP row, doesn't scan IMAP
- **Concurrency**: file lock at `sys_get_temp_dir()/xcloud_inbound.lock`.
  Second concurrent run returns `{skipped: 'busy'}` (HTTP 200).
- **Auth gate**: only `Authentication-Results:` lines stamped by the configured
  MTA hostname are trusted. SPF=pass OR DKIM=pass required (toggleable).
- **Dedup**: every processed Message-ID is recorded in
  `email_inbound_processed` with a unique key — replays are no-ops.
- **From normalization**: bare email is extracted from `From:` header, lowered,
  compared case-insensitively against `email_otps.email`.

## Test scenarios (TC)

| # | Scenario | Expected |
|---|---|---|
| TC1 | Happy path register from Gmail (SPF/DKIM pass) | User created within 60s, auto-login on poll |
| TC2 | Register from a provider without SPF/DKIM | Message moved to Rejected, `auth_fail` in history |
| TC3 | Send mail with no `XV-…` token in subject | Moved to Rejected, `no_token` |
| TC4 | Right token but `From:` differs from registered address | Moved to Rejected, `wrong_from` |
| TC5 | Two messages with same Message-ID | Second is `skipped_dup`, no double-create |
| TC6 | Token expired (>30 min) | `expired` outcome, user sees "توکن منقضی شد" |
| TC7 | Full password-reset flow | step1 → step2 → cron matches → poller redirects to step3 → new password set → can log in |
| TC8 | Cron secret wrong/missing | HTTP 403, no body |
| TC9 | Manual "Check now" pressed twice within 15s | Second click hits rate-limit redirect |
| TC10 | Two concurrent manual checks (two tabs) | One returns `busy`, other does the work |
| TC11 | Admin sets email for a user via 📧 modal | Email saved, `email_verified=1`, banner disappears, user can use password reset |
| TC12 | imap extension absent on server | Socket fallback works; test-connection card shows "ext/imap غایب" |

## Rollback

- Stop the cron.
- Restore previous `index.php` from backup.
- The new table `email_inbound_processed` is harmless to leave behind.
- New `inbound_*` settings rows can stay (they don't conflict with anything).
- SMTP settings will need to be re-seeded if you restore the old code (the
  new code's first load deletes them).

## Common issues

| Symptom | Likely cause |
|---|---|
| `imap_open failed: AUTHENTICATIONFAILED` | Wrong username/password; cPanel sometimes wants `xcloud@retrivo.ir` not just `xcloud` |
| `STARTTLS rejected` / TLS errors | Port mismatch (993=SSL, 143=STARTTLS); flip encryption setting |
| Cron runs but no matches | Check `inbound_mta_hostname` matches exactly the hostname before `;` in `Authentication-Results:` of a real message |
| All inbound messages → `auth_fail` | Same as above, or your provider doesn't add `Authentication-Results:` at all — temporarily disable `inbound_require_spf_or_dkim` to confirm |
| `socket connect failed: Connection refused` | Wrong port; or hosting blocks IMAP egress to remote (use cPanel-local hostname) |

## File touch summary

- `index.php` — all logic
- `docs/inbound-verification-runbook.md` — this file
- `lib/PHPMailer/` — **deleted**
- `lib/.htaccess` — kept (deny direct access)
