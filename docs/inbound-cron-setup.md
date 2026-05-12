# Inbound Cron Setup (cPanel)

## What this cron does

Every minute, it asks XCloud to scan the shared inbox for new verification
emails. Without it, users would have to press "چک کردن دستی" on the
verification page; the cron makes the flow feel automatic.

## Prerequisites

1. **Cron secret is set** — Inbound Settings → "Cron secret" → tick the
   "تولید خودکار" checkbox and Save. Copy the generated value (it appears
   in the page reload but you can also read it from the `settings` table).
2. **IMAP credentials work** — click "آزمایش اتصال" first and confirm green.

## cPanel steps

1. cPanel → "Cron Jobs"
2. Common Settings → **Once per minute** (`* * * * *`)
3. Command:
   ```
   curl -fsS 'https://xcloud.retrivo.ir/?action=inbound_cron&token=YOUR_SECRET_HERE' > /dev/null 2>&1
   ```
   Replace `YOUR_SECRET_HERE` with the value from step 1.
4. **Add New Cron Job**

> If you prefer not to expose the secret in the cron table, you can write a
> tiny shell wrapper in your home dir, give it 600 perms, and have the cron
> call the wrapper instead.

## Verification

After saving the cron, wait 2 minutes, then refresh
`?action=inbound_settings`. The "آخرین اجرا" timestamp should be within the
last minute, and the JSON box should show stats like
`{"scanned":0,"matched":0,...}` if there were no unread messages.

## Troubleshooting

- **403 in the response** — secret mismatch. Re-save the secret in admin UI,
  update the cron command.
- **No timestamp updates** — cron isn't actually running. Test the command
  in SSH manually: `curl -fsS '…&token=…' ` should return JSON.
- **`{"skipped":"busy"}`** repeatedly — a previous run is wedged. Check the
  lock file:
  ```
  ls -la /tmp/xcloud_inbound.lock
  ```
  If it's older than a few minutes and nothing is running, `rm` it.
- **`{"skipped":"disabled"}`** — `inbound_enabled` setting is 0; flip it on.

## Manual run for testing

Either:
- visit `?action=inbound_settings` and click "▶ اجرای دستی" (admin only), or
- POST to `?action=inbound_check_now` from the verification page (any user
  with an active pending OTP).
