# Phase 11 — Testing Checklist (Manual QA)

**Project:** XCloud
**Type:** No code changes — manual verification of Phases 1–10

## Setup

1. Set `smtp_password` in `settings` table via phpMyAdmin to the actual password of `otp@xcloud.retrivo.ir`.
2. Click "فعال کردن" in admin Users page to set `registration_enabled='1'`.
3. Have a real test email inbox available (Gmail, Yahoo, or Yandex recommended).

## Tests

### Registration — happy path
- [ ] Visit `?action=register_step1`, fill all fields correctly, submit.
- [ ] Receive OTP email within 10 seconds.
- [ ] Email arrives in Inbox (not Spam) with proper RTL Persian rendering.
- [ ] Enter correct OTP on step 2, submit.
- [ ] Auto-redirected to dashboard, logged in as new user.
- [ ] Banner does NOT appear (email is verified).
- [ ] Log out and log back in with EITHER username OR email.

### Registration — error paths
- [ ] Wrong captcha → error shown, captcha regenerates on next page.
- [ ] Invalid email format → error.
- [ ] Username with `@` or starting with digit → error.
- [ ] Weak password (no digit, or < 8 chars) → error.
- [ ] Mismatched password confirm → error.
- [ ] Username already taken → error.
- [ ] Email already registered → step 2 page still shows (no enumeration), and the existing user receives an "account already exists" notification email.
- [ ] OTP wrong code 5 times → blocked.
- [ ] OTP expires after 10 minutes → "expired" error.
- [ ] Resend OTP within 60-second cooldown → "cooldown" error.
- [ ] 6th OTP request to same email within 1 hour → "email_quota" error.
- [ ] CSRF token missing → 419 page.
- [ ] `registration_enabled='0'` → all `register_*` actions blocked.

### Password reset
- [ ] Forgot password with verified-email user → receive OTP, can reset, log in with new password.
- [ ] Forgot password with non-existent email → step 2 page shows (no enumeration), no actual email sent.
- [ ] Forgot password with admin-created user that has no email → step 2 page shows, no actual email sent.
- [ ] Wrong OTP attempts → blocked after max attempts.
- [ ] After successful reset, redirected to login with success message.

### Email-add for existing users
- [ ] Log in as admin-created user without email → yellow banner appears at top of dashboard.
- [ ] Click "بستن" → banner disappears for the session.
- [ ] Refresh page after dismiss → banner stays hidden.
- [ ] Log out and log back in → banner reappears.
- [ ] Click "افزودن ایمیل" → step 1 form.
- [ ] Complete the flow → email saved with `email_verified=1`, banner gone permanently.
- [ ] Cannot use an email already attached to another user (`email_taken` error).

### Login update (Phase 5)
- [ ] Existing username-only user can still log in with their username.
- [ ] User with verified email can log in with their email.
- [ ] Email login is case-insensitive (`USER@example.com` matches `user@example.com`).
- [ ] Wrong password → generic error (no enumeration).

### Admin toggle
- [ ] Admin sees registration card on Users page with current state.
- [ ] Clicking the button flips state and shows success message.
- [ ] When disabled: Sign-up link disappears from login page; `?action=register_step1` shows disabled message.
- [ ] When enabled: Sign-up link appears; flow works.
- [ ] Non-admin users cannot trigger the toggle endpoint.

### Security
- [ ] OTPs in `email_otps` table are SHA-256 hashed (NOT plaintext).
- [ ] OTP comparison uses `hash_equals` (constant-time).
- [ ] Captcha is single-use (cannot be reused after one verify, even if wrong).
- [ ] Sessions regenerate after registration and login.
- [ ] All POST endpoints require valid CSRF token (test with form-data tampering).
- [ ] SMTP password not exposed in any HTML output (view source on admin pages).
- [ ] Direct browser access to `/lib/PHPMailer/PHPMailer.php` returns 403.
- [ ] SQL injection attempts in email/username fail safely (PDO prepared statements).
- [ ] XSS via username (e.g. `<script>alert(1)</script>` in display) is escaped.

### Cleanup
- [ ] Wait 25 hours; verify expired `email_otps` rows are deleted automatically.
- [ ] Verify `email_send_log` rows older than 30 days are deleted.
- [ ] Cleanup runs at most once per hour (check `otp_last_cleanup` setting).

### Email deliverability
- [ ] Test against mail-tester.com → score ≥ 9/10.
- [ ] Email arrives in Gmail Inbox (not Spam, Promotions, or Updates).
- [ ] Email arrives in Outlook web Inbox.
- [ ] Email renders correctly on iOS Mail and Android Gmail.
- [ ] OTP code is clearly visible even when images are blocked.
