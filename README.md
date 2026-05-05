# 🔐 Introduction to Web Authentication

An interactive Reveal.js presentation covering web authentication — sessions, cookies, JWT, OAuth 2.0, password hashing, MFA, CSRF, CORS, RBAC, and token security.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Introduction_to_Web_Authentication/)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Web Authentication overview |
| 02 | Agenda | Topics at a glance |
| 03 | Authentication vs Authorisation | Definitions, where each applies |
| 04 | Session-Based Authentication | Cookies, server-side sessions, express-session, stores |
| 05 | Cookies Deep Dive | Set-Cookie, HttpOnly, Secure, SameSite, domain/path, expiry |
| 06 | JWT — JSON Web Tokens | Structure, header.payload.signature, signing algorithms |
| 07 | JWT in Practice | Access tokens, refresh tokens, storage strategies, Express implementation |
| 08 | OAuth 2.0 | Grant types, authorization code flow, PKCE, client credentials |
| 09 | OAuth 2.0 with Express | Passport.js, Google/GitHub strategies, callback handling |
| 10 | Password Hashing | bcrypt, scrypt, argon2 — salt, cost factor, timing attacks |
| 11 | Multi-Factor Authentication | TOTP, WebAuthn/passkeys, SMS fallbacks |
| 12 | CSRF Protection | Tokens, SameSite cookies, double-submit pattern |
| 13 | CORS & Credentials | Preflight, Access-Control headers, withCredentials |
| 14 | Role-Based Access Control | RBAC, permission models, middleware guards |
| 15 | Token Storage & Security | localStorage vs cookies, XSS vs CSRF trade-offs |
| 16 | Session Management | Invalidation, rotation, concurrent sessions, Redis stores |
| 17 | SSO & Identity Providers | SAML, OIDC, federated identity |
| 18 | Common Vulnerabilities | Session fixation, token leakage, insecure storage, brute force |
| 19 | Summary & Next Steps | Key takeaways and recommended reading |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## See also

- [Cloud_aaS_04_SaaS_Architecture](https://github.com/BrendanJamesLynskey/Cloud_aaS_04_SaaS_Architecture) — applying these primitives to multi-tenant SaaS (B2B SAML / OIDC / SCIM, Auth-as-a-Service providers).
- [Introduction to OAuth](https://github.com/BrendanJamesLynskey/Introduction_to_OAuth) — the delegated-authorisation framework in detail.
- [OAuth for MCP](https://github.com/BrendanJamesLynskey/OAuth_for_MCP) — the full Auth-as-a-Service provider tour.

## References

OWASP Foundation, *Authentication Cheat Sheet* — cheatsheetseries.owasp.org · OWASP Foundation, *Session Management Cheat Sheet* — cheatsheetseries.owasp.org · Jones et al., *RFC 7519 — JSON Web Token (JWT)*, IETF, 2015 · Hardt, *RFC 6749 — The OAuth 2.0 Authorization Framework*, IETF, 2012 · Sakimura et al., *RFC 7636 — Proof Key for Code Exchange (PKCE)*, IETF, 2015 · NIST, *SP 800-63B — Digital Identity Guidelines*, 2017 · Auth0, *Identity & Security Articles* — auth0.com/blog

## License

Educational use. Code examples provided as-is.
