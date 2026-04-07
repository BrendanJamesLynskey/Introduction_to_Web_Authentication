# Introduction to Web Authentication

---

## Slide 01 — Title

# Introduction to Web Authentication

**Sessions, Tokens, OAuth & Security Best Practices**

cookies · JWT · OAuth 2.0 · bcrypt · MFA · RBAC · CSRF · CORS

---

## Slide 02 — Agenda

### Foundations
- Authentication vs Authorisation
- Session-based authentication
- Cookies deep dive
- JWT — JSON Web Tokens

### Protocols & Standards
- JWT in practice (access & refresh tokens)
- OAuth 2.0 grant types & PKCE
- OAuth 2.0 with Express & Passport.js
- SSO & Identity Providers (SAML, OIDC)

### Credentials & Access
- Password hashing (bcrypt, scrypt, argon2)
- Multi-factor authentication & passkeys
- Role-based access control (RBAC)
- Token storage & security trade-offs

### Defence & Hardening
- CSRF protection patterns
- CORS & credentials
- Session management & invalidation
- Common vulnerabilities & mitigations

---

## Slide 03 — Authentication vs Authorisation

Two distinct concerns that are often conflated. **Authentication** proves *who you are*; **authorisation** determines *what you can do*. Every secure system needs both.

### Authentication (AuthN)
- **Identity verification** — "Who is this user?"
- Occurs *before* authorisation
- Mechanisms: passwords, tokens, biometrics, certificates
- Result: a verified identity (user ID, session, JWT)
- HTTP `401 Unauthorized` = unauthenticated

### Authorisation (AuthZ)
- **Permission checking** — "Can this user do X?"
- Occurs *after* authentication
- Mechanisms: RBAC, ABAC, ACLs, policies
- Result: allow or deny the requested action
- HTTP `403 Forbidden` = unauthorised

### Where Each Applies

**AuthN** at the front door: login forms, API key validation, SSO callbacks, token verification middleware. **AuthZ** at every resource: route guards, database row-level security, feature flags, admin panels. A user can be authenticated but still unauthorised to access a resource.

```javascript
// Express: AuthN middleware runs first, then AuthZ
app.get('/admin/users', authenticate, authorise('admin'), (req, res) => {
  // authenticate() verifies the JWT/session — sets req.user
  // authorise('admin') checks req.user.role === 'admin'
  res.json(await User.findAll());
});
```

---

## Slide 04 — Session-Based Authentication

The classic approach: the server creates a **session** after login, stores it server-side, and sends a **session ID** cookie to the browser. Every subsequent request includes the cookie automatically.

**Flow:** Browser → POST /login (credentials) → Express → store session → Session Store → Set-Cookie: sid=abc123 → Browser → GET /dashboard (Cookie: sid=abc123)

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, httpOnly: true, maxAge: 1800000 } // 30 min
}));

app.post('/login', async (req, res) => {
  const user = await User.findByEmail(req.body.email);
  if (user && await bcrypt.compare(req.body.password, user.hash)) {
    req.session.userId = user.id;        // session stored in Redis
    return res.redirect('/dashboard');
  }
  res.status(401).json({ error: 'Invalid credentials' });
});
```

**Pros:** Simple mental model, easy revocation (delete session), cookie sent automatically, works with any framework.

**Cons:** Server-side state, requires shared store in clustered environments, CSRF-vulnerable if cookies are used.

**Stores:** `connect-redis`, `connect-mongo`, `connect-pg-simple`, `memorystore` (dev only).

---

## Slide 05 — Cookies Deep Dive

Cookies are the transport layer for session IDs, CSRF tokens, and sometimes JWTs. Understanding every attribute is critical for security.

| Attribute | Purpose | Recommended Value |
|-----------|---------|-------------------|
| `HttpOnly` | Prevents JavaScript access via `document.cookie` | Always set for auth cookies |
| `Secure` | Cookie only sent over HTTPS | Always set in production |
| `SameSite` | Controls cross-origin cookie sending | `Lax` (default) or `Strict` |
| `Domain` | Which domains receive the cookie | Omit to restrict to exact origin |
| `Path` | URL path scope | `/` for session cookies |
| `Max-Age / Expires` | Lifetime — omit for session cookies | 30 min for sessions, 7-30 days for refresh |
| `__Host-` prefix | Requires `Secure`, no `Domain`, path `/` | Use for highest security cookies |

```http
Set-Cookie: __Host-sid=abc123; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=1800
```

### SameSite=Strict
Cookie **never** sent on cross-site requests. Breaks OAuth redirects and inbound links that expect a logged-in session.

### SameSite=Lax
Sent on top-level navigations (GET) but **not** on cross-site POST/fetch. Good balance of usability and CSRF protection.

### SameSite=None
Sent on all cross-site requests. **Requires `Secure`**. Needed for cross-origin APIs and third-party embeds.

---

## Slide 06 — JWT — JSON Web Tokens

A JWT is a compact, URL-safe token with three Base64URL-encoded parts separated by dots: **header**.**payload**.**signature**. The server can verify the token without storing state.

### Header
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
Declares the signing algorithm and token type.

### Payload (Claims)
```json
{
  "sub": "user_4821",
  "email": "alice@example.com",
  "role": "admin",
  "iat": 1712505600,
  "exp": 1712509200
}
```
Registered claims: `sub`, `iss`, `aud`, `exp`, `iat`, `nbf`, `jti`.

### Signature
```
HMACSHA256(
  base64url(header) + "." + base64url(payload),
  secret
)
```
Integrity check. Tampered tokens fail verification.

### Signing Algorithms

| Algorithm | Type | Key | Use Case |
|-----------|------|-----|----------|
| `HS256` | Symmetric HMAC | Shared secret | Single-server APIs |
| `RS256` | Asymmetric RSA | Private key signs, public key verifies | Distributed systems, OIDC |
| `ES256` | Asymmetric ECDSA | Smaller keys, faster signing | Mobile, IoT, performance-critical |
| `none` | No signature | — | **NEVER use in production** |

---

## Slide 07 — JWT in Practice

A production JWT flow uses short-lived **access tokens** paired with longer-lived **refresh tokens**. The access token authorises API calls; the refresh token obtains new access tokens without re-authentication.

```javascript
const jwt = require('jsonwebtoken');
const SECRET = process.env.JWT_SECRET;

// Issue tokens after successful login
function issueTokens(user) {
  const accessToken = jwt.sign(
    { sub: user.id, role: user.role },
    SECRET,
    { expiresIn: '15m' }
  );
  const refreshToken = jwt.sign(
    { sub: user.id, type: 'refresh' },
    SECRET,
    { expiresIn: '7d' }
  );
  return { accessToken, refreshToken };
}

app.post('/login', async (req, res) => {
  const user = await authenticate(req.body);
  if (!user) return res.status(401).json({ error: 'Bad credentials' });
  const tokens = issueTokens(user);
  await RefreshToken.create({ userId: user.id,
    hash: hashToken(tokens.refreshToken) });
  res.json(tokens);
});
```

```javascript
// Middleware: verify access token
function authenticate(req, res, next) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer '))
    return res.status(401).json({ error: 'No token' });

  try {
    req.user = jwt.verify(header.slice(7), SECRET);
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError')
      return res.status(401).json({ error: 'Token expired' });
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Refresh endpoint
app.post('/refresh', async (req, res) => {
  const { refreshToken } = req.body;
  const payload = jwt.verify(refreshToken, SECRET);
  const stored = await RefreshToken.findOne({
    userId: payload.sub,
    hash: hashToken(refreshToken)
  });
  if (!stored) return res.status(403).json({ error: 'Revoked' });
  const user = await User.findById(payload.sub);
  res.json(issueTokens(user));
});
```

- **Access Token:** Short-lived (5-15 min), stateless, sent as `Authorization: Bearer <token>`.
- **Refresh Token:** Long-lived (7-30 days), stored server-side (DB/Redis), enables silent re-auth.
- **Token Rotation:** Issue a new refresh token on every use. Invalidate the old one. Detects token reuse (theft).

---

## Slide 08 — OAuth 2.0

OAuth 2.0 is an **authorisation framework** that lets a third-party application access resources on behalf of a user *without* sharing the user's password. It defines several **grant types** for different scenarios.

### Authorization Code + PKCE
- Most secure flow for web and mobile apps
- User redirected to authorisation server
- Server returns a short-lived **auth code**
- App exchanges code + **code_verifier** for tokens
- PKCE prevents code interception attacks

### Client Credentials
- Machine-to-machine (no user involved)
- App sends `client_id` + `client_secret`
- Returns access token directly
- Used for service accounts, cron jobs, microservices

### Authorization Code Flow

1. User clicks "Login with Google"
2. App redirects to /authorize
3. User consents, redirected with auth code
4. App exchanges code + PKCE for tokens
5. Auth server returns access token + ID token

### Deprecated: Implicit Grant

The implicit flow returned tokens directly in the URL fragment. It is **no longer recommended** — use Authorization Code + PKCE instead. URL fragments leak through browser history, referer headers, and proxy logs.

---

## Slide 09 — OAuth 2.0 with Express

**Passport.js** abstracts OAuth into pluggable strategies. Each strategy handles the redirect, callback, and token exchange. You just configure credentials and a verify callback.

```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
  },
  async (accessToken, refreshToken, profile, done) => {
    let user = await User.findOne({
      where: { googleId: profile.id }
    });
    if (!user) {
      user = await User.create({
        googleId: profile.id,
        email: profile.emails[0].value,
        name: profile.displayName,
        avatar: profile.photos[0].value
      });
    }
    return done(null, user);
  }
));
```

```javascript
// Serialisation for session persistence
passport.serializeUser((user, done) => done(null, user.id));
passport.deserializeUser(async (id, done) => {
  const user = await User.findByPk(id);
  done(null, user);
});

// Mount middleware
app.use(passport.initialize());
app.use(passport.session());

// Routes
app.get('/auth/google',
  passport.authenticate('google', { scope: ['profile', 'email'] })
);

app.get('/auth/google/callback',
  passport.authenticate('google', { failureRedirect: '/login' }),
  (req, res) => res.redirect('/dashboard')
);
```

- **Multiple Providers:** Add strategies for GitHub, Facebook, Microsoft, Apple, etc. Link accounts by matching email or storing provider IDs per user.
- **Callback Security:** Always validate the `state` parameter to prevent CSRF. Use HTTPS. Restrict redirect URIs in the provider's console.

---

## Slide 10 — Password Hashing

Never store passwords in plain text. Use a **slow, salted, one-way hash** function. The three accepted algorithms are **bcrypt**, **scrypt**, and **argon2**.

| Algorithm | CPU-Hard | Memory-Hard | Output |
|-----------|----------|-------------|--------|
| bcrypt | Yes (cost factor) | No (4 KB) | 60-char string with embedded salt |
| scrypt | Yes (N) | Yes (r, p) | Configurable, built into Node crypto |
| argon2id | Yes (iterations) | Yes (memory) | PHC-format string, OWASP recommended |

```javascript
// bcrypt — most widely used
const bcrypt = require('bcrypt');
const SALT_ROUNDS = 12; // ~250ms on modern hardware

async function register(email, password) {
  const hash = await bcrypt.hash(password, SALT_ROUNDS);
  await User.create({ email, passwordHash: hash });
}

async function login(email, password) {
  const user = await User.findByEmail(email);
  if (!user) return null;
  const valid = await bcrypt.compare(password, user.passwordHash);
  return valid ? user : null;
}
```

```javascript
// argon2 — OWASP first choice
const argon2 = require('argon2');

const hash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 65536,  // 64 MB
  timeCost: 3,
  parallelism: 4
});

const valid = await argon2.verify(hash, password);
```

```javascript
// scrypt — built into Node, no npm package needed
const { scrypt, randomBytes } = require('crypto');
const { promisify } = require('util');
const scryptAsync = promisify(scrypt);

async function hashPassword(password) {
  const salt = randomBytes(16).toString('hex');
  const derived = await scryptAsync(password, salt, 64);
  return `${salt}:${derived.toString('hex')}`;
}
```

**Timing Attacks:** String comparison (`===`) short-circuits on first mismatch. `bcrypt.compare` and `argon2.verify` use constant-time comparison internally.

---

## Slide 11 — Multi-Factor Authentication

MFA requires **two or more factors** from different categories: **something you know** (password), **something you have** (device/key), **something you are** (biometrics).

### TOTP (Time-Based OTP)
- Shared secret + current time → 6-digit code
- RFC 6238 — 30-second windows
- Google Authenticator, Authy, 1Password
- Server stores the shared secret per user

```javascript
const { authenticator } = require('otplib');
const secret = authenticator.generateSecret();
const uri = authenticator.keyuri(user.email, 'MyApp', secret);
const valid = authenticator.verify({
  token: req.body.code, secret: user.totpSecret
});
```

### WebAuthn / Passkeys
- Public-key cryptography, phishing-resistant
- Authenticator creates key pair per site
- Server stores public key only
- Browser API: `navigator.credentials`
- Passkeys sync across devices via iCloud/Google

```javascript
const { generateRegistrationOptions, verifyRegistrationResponse } = require('@simplewebauthn/server');

const options = await generateRegistrationOptions({
  rpName: 'MyApp',
  rpID: 'example.com',
  userID: user.id,
  userName: user.email
});
```

### Recovery & Fallbacks
- **Recovery codes** — one-time use, store hashed
- **SMS OTP** — weakest factor (SIM swap attacks)
- Email magic links — acceptable as second step
- Hardware keys (YubiKey) — strongest physical factor

**SMS Risks:** NIST SP 800-63B discourages SMS as sole second factor. SIM swapping, SS7 interception, and social engineering make SMS vulnerable.

---

## Slide 12 — CSRF Protection

**Cross-Site Request Forgery** tricks a user's browser into making an authenticated request to your site from a malicious page. The browser automatically includes cookies, so the request appears legitimate.

### Synchronizer Token Pattern

- Server generates a unique CSRF token per session
- Token embedded in forms as hidden field
- Server validates token on every state-changing request
- Token not in cookies — attacker can't read it

```javascript
const csrf = require('csurf');
app.use(csrf({ cookie: false })); // session-based

app.get('/form', (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});
// Template: <input type="hidden" name="_csrf" value="{{csrfToken}}">
```

### SameSite Cookies
- `SameSite=Lax` blocks cross-site POST
- No extra tokens needed for simple cases
- Supported in all modern browsers

```javascript
app.use(session({
  cookie: { sameSite: 'lax', secure: true, httpOnly: true }
}));
```

### Double-Submit Cookie
- Set a random token in a cookie *and* a header
- Server compares cookie value vs header value
- Attacker can send cookie but can't read/set the header
- Stateless — no server-side token storage

```javascript
app.use((req, res, next) => {
  if (['POST','PUT','DELETE'].includes(req.method)) {
    if (req.cookies.csrf !== req.headers['x-csrf-token'])
      return res.status(403).json({ error: 'CSRF' });
  }
  next();
});
```

---

## Slide 13 — CORS & Credentials

**Cross-Origin Resource Sharing** controls which domains can call your API from a browser. Without CORS headers, the **same-origin policy** blocks cross-origin fetch/XHR requests.

```javascript
const cors = require('cors');

app.use(cors({
  origin: 'https://app.example.com',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400
}));

// Dynamic origin (multiple allowed origins)
const allowedOrigins = ['https://app.example.com', 'https://admin.example.com'];

app.use(cors({
  origin: (origin, cb) => {
    if (!origin || allowedOrigins.includes(origin)) cb(null, true);
    else cb(new Error('Not allowed by CORS'));
  },
  credentials: true
}));
```

| Header | Purpose |
|--------|---------|
| `Access-Control-Allow-Origin` | Which origin(s) can access the response |
| `Access-Control-Allow-Credentials` | Allow cookies / Authorization header |
| `Access-Control-Allow-Methods` | Permitted HTTP methods (preflight) |
| `Access-Control-Allow-Headers` | Permitted request headers (preflight) |
| `Access-Control-Max-Age` | Cache preflight response (seconds) |

### Common Pitfalls
- `Access-Control-Allow-Origin: *` **cannot** be used with `credentials: true`
- Preflight (OPTIONS) must return 2xx with CORS headers
- `withCredentials: true` needed on fetch/XHR for cookies
- CORS is browser-enforced — curl/Postman bypass it entirely

### Preflight Request
Browser sends an `OPTIONS` request before any non-simple request (custom headers, PUT/DELETE, JSON content-type). If CORS headers are missing, the actual request is blocked.

---

## Slide 14 — Role-Based Access Control (RBAC)

RBAC assigns **permissions** to **roles**, then assigns roles to users. It simplifies permission management — you update the role definition, not individual user records.

| Role | Permissions |
|------|-------------|
| viewer | `read:articles`, `read:comments` |
| editor | viewer + `write:articles`, `write:comments` |
| admin | editor + `manage:users`, `manage:settings` |
| superadmin | All permissions, can manage roles themselves |

```javascript
const ROLES = {
  viewer:  ['read:articles', 'read:comments'],
  editor:  ['read:articles', 'read:comments', 'write:articles', 'write:comments'],
  admin:   ['read:articles', 'read:comments', 'write:articles', 'write:comments',
            'manage:users', 'manage:settings'],
};

// Middleware guard factory
function requirePermission(...perms) {
  return (req, res, next) => {
    const userPerms = ROLES[req.user.role] || [];
    const hasAll = perms.every(p => userPerms.includes(p));
    if (!hasAll) return res.status(403).json({ error: 'Insufficient permissions' });
    next();
  };
}

// Usage in routes
app.get('/articles', authenticate, requirePermission('read:articles'), articleController.list);
app.delete('/users/:id', authenticate, requirePermission('manage:users'), userController.delete);
```

**Beyond RBAC:** ABAC (attribute-based) uses policies with user attributes, resource properties, and environment context. Libraries: `casl`, `casbin`, OPA.

---

## Slide 15 — Token Storage & Security

Where you store tokens in the browser determines which attacks you're vulnerable to. There is **no perfect solution** — only trade-offs between **XSS** and **CSRF**.

| Storage | XSS Risk | CSRF Risk | Auto-Sent | Recommendation |
|---------|----------|-----------|-----------|----------------|
| `localStorage` | HIGH — JS can read it | None | No (manual header) | Avoid for auth tokens |
| `sessionStorage` | HIGH — JS can read it | None | No | Avoid for auth tokens |
| `HttpOnly cookie` | LOW — JS can't read it | Medium | Yes (automatic) | Best for session/refresh tokens |
| In-memory (JS variable) | Medium — XSS can access | None | No | OK for short-lived access tokens |

### Recommended Pattern
- **Refresh token** in `HttpOnly`, `Secure`, `SameSite=Strict` cookie
- **Access token** in memory (JS variable) — never persisted
- On page load: call `/refresh` to get a new access token
- Access token attached to API calls via `Authorization` header
- Short expiry (5-15 min) limits the window of XSS exploitation

### XSS vs CSRF Trade-off
- **XSS** — attacker injects JS that reads tokens from `localStorage` or makes API calls
- **CSRF** — attacker tricks browser into sending cookies to your API from a malicious page
- HttpOnly cookies mitigate XSS token theft but open CSRF surface
- CSRF is easier to defend against than XSS — prefer cookie storage
- **Defence in depth:** CSP headers + SameSite cookies + CSRF tokens

---

## Slide 16 — Session Management

Sessions must be actively managed: **rotated** to prevent fixation, **invalidated** on logout, and **expired** after inactivity. A session store like Redis provides fast lookups and TTL-based expiry.

```javascript
// Session rotation after login (prevent fixation)
app.post('/login', async (req, res) => {
  const user = await authenticate(req.body);
  if (!user) return res.status(401).end();

  req.session.regenerate((err) => {
    req.session.userId = user.id;
    req.session.role = user.role;
    req.session.loginAt = Date.now();
    req.session.save(() => res.redirect('/dashboard'));
  });
});

// Logout: destroy session
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    res.clearCookie('connect.sid');
    res.redirect('/login');
  });
});
```

```javascript
// Redis session store with TTL
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

app.use(session({
  store: new RedisStore({
    client: redisClient,
    prefix: 'sess:',
    ttl: 1800
  }),
  rolling: true,
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, httpOnly: true, maxAge: 1800000 }
}));

// Force logout all sessions for a user
async function invalidateAllSessions(userId) {
  const keys = await redisClient.keys('sess:*');
  for (const key of keys) {
    const data = JSON.parse(await redisClient.get(key));
    if (data.userId === userId) await redisClient.del(key);
  }
}
```

- **Rotation:** Regenerate session ID after login, privilege escalation, and password change. Prevents session fixation attacks.
- **Concurrent Sessions:** Track active sessions per user. Let users view and revoke sessions (like GitHub/Google "active sessions" page).
- **Absolute Timeout:** Even with rolling TTL, enforce a maximum session lifetime (e.g., 24h). Re-authentication required after absolute expiry.

---

## Slide 17 — SSO & Identity Providers

**Single Sign-On** lets users authenticate once and access multiple applications. The identity is managed by a central **Identity Provider (IdP)**, and applications (Service Providers) trust the IdP's assertions.

### SAML 2.0
- XML-based, enterprise-focused
- IdP sends signed **SAML Assertion** via POST
- Assertions contain identity + attributes
- Used by Okta, Azure AD, ADFS
- Complex but well-established (2005)

```javascript
// passport-saml
passport.use(new SamlStrategy({
  entryPoint: 'https://idp.example.com/sso',
  issuer: 'my-app',
  cert: IDP_PUBLIC_CERT,
  callbackUrl: '/auth/saml/callback'
}, (profile, done) => {
  return done(null, profile);
}));
```

### OpenID Connect (OIDC)
- Identity layer on top of OAuth 2.0
- Returns **ID Token** (JWT) with user claims
- `/.well-known/openid-configuration` for discovery
- Standard scopes: `openid profile email`
- Used by Google, Auth0, Keycloak, Cognito

```json
{
  "iss": "https://accounts.google.com",
  "sub": "1104327891",
  "aud": "my-client-id",
  "email": "alice@example.com",
  "name": "Alice Smith",
  "iat": 1712505600,
  "exp": 1712509200
}
```

### Federated Identity
- Trust relationships between domains
- User authenticates at home IdP
- SP accepts tokens from trusted IdPs
- Enterprise: Azure AD B2B, Shibboleth
- Consumer: "Login with Google/Apple"

| | SAML | OIDC |
|---|------|------|
| Format | XML | JSON/JWT |
| Transport | POST/Redirect | HTTP APIs |
| Best for | Enterprise SSO | Modern apps |

---

## Slide 18 — Common Vulnerabilities

Understanding attack vectors is essential for building secure authentication. These are the most frequently exploited weaknesses in web auth systems.

### Session Fixation
- Attacker sets victim's session ID before login
- After victim logs in, attacker uses the same session
- **Fix:** regenerate session ID on login (`req.session.regenerate()`)

### Token Leakage
- Tokens in URLs, referrer headers, or browser history
- Logging middleware accidentally logs Authorization headers
- **Fix:** tokens in headers only, scrub logs, use POST for token exchange

### Insecure Storage
- Passwords stored as MD5/SHA-256 (fast hashes)
- Tokens in `localStorage` exposed to XSS
- **Fix:** bcrypt/argon2 for passwords, HttpOnly cookies for tokens

### Brute Force & Credential Stuffing
- Automated login attempts with leaked credential lists
- No rate limiting = unlimited attempts
- **Fix:** rate limiting, account lockout, CAPTCHA, MFA

```javascript
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: { error: 'Too many login attempts' },
  standardHeaders: true,
  keyGenerator: (req) => req.body.email || req.ip
});

app.post('/login', loginLimiter, loginHandler);
```

### Defence Checklist
- Rotate session IDs on login and privilege changes
- Set `HttpOnly`, `Secure`, `SameSite` on all auth cookies
- Rate-limit login and token endpoints
- Use `helmet` for security headers (CSP, HSTS, X-Frame)
- Implement account lockout with exponential backoff
- Log failed authentication attempts for monitoring
- Require MFA for privileged accounts

---

## Slide 19 — Summary & Next Steps

### Key Takeaways
- **Authentication** proves identity; **authorisation** grants access
- Sessions are stateful (server-side); JWTs are stateless (client-side)
- OAuth 2.0 + PKCE is the standard for third-party auth
- Always hash passwords with bcrypt, scrypt, or argon2
- MFA significantly reduces account compromise risk
- CSRF and CORS are distinct but interconnected concerns
- Token storage is a trade-off between XSS and CSRF
- Defence in depth — no single control is sufficient

### Recommended Reading
- OWASP Authentication Cheat Sheet
- OWASP Session Management Cheat Sheet
- RFC 7519 — JSON Web Tokens
- RFC 6749 — OAuth 2.0 Framework
- RFC 7636 — PKCE for OAuth
- NIST SP 800-63B — Digital Identity Guidelines
- Auth0 Blog — identity architecture articles

### Next Steps
- Build a login system with sessions + bcrypt
- Add JWT access/refresh token rotation
- Integrate Google OAuth via Passport.js
- Implement TOTP-based MFA with `otplib`
- Set up RBAC middleware with permission checks
- Explore WebAuthn/passkeys for passwordless auth

### Essential npm Packages
`bcrypt` · `jsonwebtoken` · `passport` · `express-session` · `connect-redis` · `cors` · `helmet` · `express-rate-limit` · `otplib` · `@simplewebauthn/server`
