# CyLab Security Academy — Writeups

Personal writeups from CyLab Security Academy web exploitation and authentication labs, documented with Burp Suite Professional. Each writeup includes the approach, screenshots of key steps, and lessons learned — including mistakes made and corrected along the way, since those are often as instructive as the solution itself.

> ⚠️ Flags are included as submitted on CyLab's own platform for personal reference. If you're working through the same challenges, consider solving them yourself first — the value is in the process.

## Index

| Challenge | Category | Difficulty | Writeup |
|---|---|---|---|
| Crack the Gate 1 | Web Exploitation | Easy | [`web-exploitation/crack-the-gate-1`](./web-exploitation/crack-the-gate-1/README.md) |
| login (Mars) | Authentication | Medium | [`authentication/cylab-login-mars`](./authentication/cylab-login-mars/README.md) |
| PortSwigger: 2FA Simple Bypass | Authentication | Apprentice | [`authentication/portswigger-2fa-simple-bypass`](./authentication/portswigger-2fa-simple-bypass/README.md) |

## Repo Structure

```
.
├── web-exploitation/
│   └── crack-the-gate-1/
└── authentication/
    ├── cylab-login-mars/
    └── portswigger-2fa-simple-bypass/
```

Each challenge folder contains:
- `README.md` — the full writeup
- `screenshots/` — numbered screenshots referenced inline in the writeup

## Tools Used

- **Burp Suite Professional** — Proxy, Repeater, Decoder
- **FoxyProxy** — browser proxy switching
- Target platforms: CyLab Security Academy (picoCTF/picoMini-hosted instances), PortSwigger Web Security Academy

## Recurring Lessons Across These Challenges

A few patterns that showed up more than once, worth keeping in mind for future labs:

1. **Check raw responses, not rendered ones.** Browser "View Source" and dev tools can normalize or hide content that Burp's raw Proxy history shows plainly — HTML comments, exact whitespace, and headers included.
2. **Client-side checks are not security.** Any validation (login, 2FA gating) that happens only in the browser, or that the server fails to re-verify server-side, can be bypassed by directly crafting or navigating to the request that skips it.
3. **Raw HTTP is strict.** A stray leading space on a header line or a missing blank line before the body will produce confusing errors (`400`, `408`) that look like logic bugs but are actually just malformed requests.
4. **Encoded ≠ encrypted, and not everything is base64.** Both ROT13 and base64 showed up across these challenges — if one decoding attempt produces garbage, try the next.
