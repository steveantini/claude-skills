# API Security Reference

> **Version:** 1.1.0
> **Last Updated:** 2026-09-21
> **Applicability:** REST APIs, GraphQL APIs, webhook receivers
> **Dependencies:** JWT library, OAuth2 provider, rate limiting middleware

---

## JWT Validation

### Token Structure

```
Header.Payload.Signature
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxMjM0NSJ9.signature
```

### Validation Checklist (Every Request)

1. Verify signature against known public key or secret
2. Check `exp` (expiration) — reject expired tokens
3. Check `nbf` (not before) — reject tokens used too early
4. Check `iss` (issuer) — matches expected issuer
5. Check `aud` (audience) — matches your service identifier
6. Check `alg` (algorithm) — matches expected algorithm (never `none`)

### Implementation

```javascript
import jwt from 'jsonwebtoken';

const PUBLIC_KEY = fs.readFileSync('public.pem');

function verifyToken(token) {
  return jwt.verify(token, PUBLIC_KEY, {
    algorithms: ['RS256'],      // Explicit allowlist — NEVER omit
    issuer: 'https://auth.example.com',
    audience: 'api.example.com',
    clockTolerance: 30,         // 30-second clock skew tolerance
  });
}
```

### Algorithm Selection

| Algorithm | Type | Use Case |
|---|---|---|
| `RS256` | Asymmetric (RSA) | Multi-service architectures, public key distribution |
| `ES256` | Asymmetric (ECDSA) | Same as RS256, smaller keys |
| `HS256` | Symmetric (HMAC) | Single-service only (shared secret) |
| `none` | **NEVER** | Algorithm confusion attack vector |

### Critical Rules

- **Always specify allowed algorithms explicitly** — prevents algorithm confusion attacks
- Store signing keys in secrets manager, never in code
- Access tokens: short-lived (5-15 minutes)
- Refresh tokens: longer-lived, stored securely, rotated on use
- Include minimal claims in token — do not store PII
- Maintain a token revocation mechanism (blocklist or short expiry + refresh)

### Token Refresh Pattern

```
Client                              Server
  |--- Access Token (expired) ------>|
  |<-- 401 Unauthorized -------------|
  |--- Refresh Token --------------->|  (to /auth/refresh)
  |<-- New Access + Refresh Tokens --|
  |--- New Access Token ------------>|
  |<-- 200 OK ----------------------|
```

Refresh token rotation: issue new refresh token on each use, invalidate the old one. If an old refresh token is reused, invalidate all tokens for that user (potential theft).

---

## API Key Management

### Key Generation

```javascript
import crypto from 'crypto';

function generateApiKey() {
  const prefix = 'sk_live_';  // Prefix for identification and scanning
  const key = crypto.randomBytes(32).toString('hex');
  return `${prefix}${key}`;
  // Example: sk_live_a1b2c3d4e5f6...
}
```

### Key Prefixes (Convention)

| Prefix | Meaning |
|---|---|
| `sk_live_` | Secret key, production |
| `sk_test_` | Secret key, test/sandbox |
| `pk_live_` | Publishable key, production |
| `pk_test_` | Publishable key, test/sandbox |

Prefixes enable automated secret scanning (GitHub, GitGuardian) and quick identification.

### Storage

```javascript
// Store HASH of key, not plaintext
const keyHash = crypto.createHash('sha256').update(apiKey).digest('hex');
// Store: { keyHash, userId, scopes, createdAt, lastUsedAt, expiresAt }

// On validation: hash incoming key, compare to stored hash
function validateApiKey(providedKey) {
  const hash = crypto.createHash('sha256').update(providedKey).digest('hex');
  return db.apiKeys.findOne({ keyHash: hash, revokedAt: null });
}
```

### Key Lifecycle

```
Generate --> Hash + Store --> Assign Scopes --> Active Use
                                                    |
                     Rotate (generate new, overlap period, revoke old)
                                                    |
                     Revoke (set revokedAt, reject on next use)
```

### Scoping

```javascript
// Define granular scopes
const keyRecord = {
  keyHash: '...',
  userId: 'user_123',
  scopes: ['read:orders', 'write:orders'],  // Least privilege
  rateLimit: { requests: 1000, window: '1h' },
  allowedIPs: ['203.0.113.0/24'],            // Optional IP restriction
  expiresAt: new Date('2026-06-01'),
};

// Check scope in middleware
function requireScope(scope) {
  return (req, res, next) => {
    if (!req.apiKey.scopes.includes(scope)) {
      return res.status(403).json({ error: 'Insufficient scope' });
    }
    next();
  };
}
```

### Rotation Process

1. Generate new key, return to user
2. Both old and new keys active during overlap period (e.g., 24 hours)
3. User updates their integration to use new key
4. Old key automatically revoked after overlap period
5. Log rotation event

---

## OAuth2 Flows

### Flow Selection

| Flow | Use Case | Client Type |
|---|---|---|
| Authorization Code + PKCE | Web apps, SPAs, mobile | Public or confidential |
| Client Credentials | Server-to-server | Confidential |
| Device Authorization | CLI tools, IoT, TV apps | Public (input-constrained) |
| **Implicit** | **Deprecated** | **Never use** |
| **Resource Owner Password** | **Deprecated** | **Never use** |

### Authorization Code + PKCE (Recommended)

```javascript
// 1. Generate PKCE challenge
const codeVerifier = crypto.randomBytes(32).toString('base64url');
const codeChallenge = crypto
  .createHash('sha256')
  .update(codeVerifier)
  .digest('base64url');

// 2. Redirect to authorization endpoint
const authUrl = new URL('https://auth.example.com/authorize');
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('client_id', CLIENT_ID);
authUrl.searchParams.set('redirect_uri', REDIRECT_URI);
authUrl.searchParams.set('scope', 'openid profile email');
authUrl.searchParams.set('state', generateRandomState());
authUrl.searchParams.set('code_challenge', codeChallenge);
authUrl.searchParams.set('code_challenge_method', 'S256');

// 3. Exchange code for tokens (server-side)
const tokenResponse = await fetch('https://auth.example.com/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    grant_type: 'authorization_code',
    code: authorizationCode,
    redirect_uri: REDIRECT_URI,
    client_id: CLIENT_ID,
    code_verifier: codeVerifier,
  }),
});
```

### OAuth2 Security Rules

- Always use PKCE (even for confidential clients)
- Validate `state` parameter to prevent CSRF
- Validate `redirect_uri` against strict allowlist (exact match, no wildcards)
- Store tokens server-side; never expose in URLs or localStorage
- Validate ID token claims (iss, aud, exp, nonce)

---

## CORS Configuration

### Restrictive Configuration (Recommended)

```javascript
const corsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://app.example.com',
      'https://admin.example.com',
    ];
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  credentials: true,
  maxAge: 86400,       // Preflight cache: 24 hours
};

app.use(cors(corsOptions));
```

### CORS Rules

| Setting | Secure Value | Notes |
|---|---|---|
| `Access-Control-Allow-Origin` | Specific origins | **Never `*` with credentials** |
| `Access-Control-Allow-Credentials` | `true` only if needed | Requires specific origin |
| `Access-Control-Allow-Methods` | Only methods you use | Not `*` |
| `Access-Control-Allow-Headers` | Specific headers | Explicit allowlist |
| `Access-Control-Expose-Headers` | Only needed headers | Limit what JS can read |
| `Access-Control-Max-Age` | `86400` (24h) | Reduce preflight requests |

### Common Mistakes

- Reflecting `Origin` header directly as `Allow-Origin` without validation (allows any origin)
- Using `*` with `credentials: true` (browser rejects this, but attempting it signals misconfiguration)
- Overly broad regex for origin matching (e.g., `/example\.com/` matches `evil-example.com`)

---

## Rate Limiting Strategy

### Tiered Approach

```
Global Rate Limit (infrastructure level, e.g., 10,000 req/min per IP)
  └── Service Rate Limit (per API key or user)
       └── Endpoint Rate Limit (per endpoint sensitivity)
```

### Configuration by Tier

| Tier | Key | Window | Limit | Response |
|---|---|---|---|---|
| Free | API key | 1 hour | 100 | 429 + Retry-After |
| Pro | API key | 1 minute | 100 | 429 + Retry-After |
| Enterprise | API key | 1 minute | 1000 | 429 + Retry-After |
| Auth endpoints | IP | 15 min | 10 | 429 + Retry-After |

### Implementation Pattern

```javascript
// Token bucket or sliding window in Redis
async function checkRateLimit(key, limit, windowSec) {
  const current = await redis.incr(key);
  if (current === 1) {
    await redis.expire(key, windowSec);
  }
  return {
    allowed: current <= limit,
    remaining: Math.max(0, limit - current),
    resetAt: await redis.ttl(key),
  };
}
```

### Rate Limit Headers

```
RateLimit-Limit: 100
RateLimit-Remaining: 57
RateLimit-Reset: 1620000060
```

---

## Request Size Limits and Timeouts

### Size Limits

```javascript
// Express
app.use(express.json({ limit: '100kb' }));          // JSON bodies
app.use(express.urlencoded({ limit: '100kb', extended: true }));
app.use(express.raw({ limit: '10mb', type: 'application/octet-stream' }));  // File uploads

// Reverse proxy (nginx)
// client_max_body_size 10m;
```

### Timeout Configuration

| Timeout | Value | Purpose |
|---|---|---|
| Server request timeout | 30s | Kill long-running requests |
| Database query timeout | 10s | Prevent slow query resource exhaustion |
| External API calls | 5-10s | Prevent cascading failures |
| WebSocket idle timeout | 5 min | Free idle connections |
| Keep-alive timeout | 65s | Slightly above load balancer timeout |

```javascript
// Request timeout middleware
function requestTimeout(ms) {
  return (req, res, next) => {
    const timer = setTimeout(() => {
      if (!res.headersSent) {
        res.status(408).json({ error: 'Request timeout' });
      }
    }, ms);
    res.on('finish', () => clearTimeout(timer));
    next();
  };
}

app.use(requestTimeout(30000));
```

---

## API Versioning Security

### Versioning Strategy

```
/api/v1/resource    # URL path versioning (most common)
Accept: application/vnd.api+json;version=1   # Header versioning
```

### Deprecation Security

- Maintain security patches on all supported versions
- Set hard sunset dates — do not maintain insecure old versions indefinitely
- Return deprecation headers on old versions:

```
Deprecation: true
Sunset: Sat, 01 Jan 2027 00:00:00 GMT
Link: <https://api.example.com/v2/docs>; rel="successor-version"
```

- Block requests to deprecated/unsupported versions with `410 Gone`
- Apply the same security middleware (auth, rate limiting, validation) to all versions

---

## Webhook Signature Verification

### Signing Outbound Webhooks

```javascript
import crypto from 'crypto';

function signWebhookPayload(payload, secret) {
  const timestamp = Math.floor(Date.now() / 1000).toString();
  const signedContent = `${timestamp}.${JSON.stringify(payload)}`;
  const signature = crypto
    .createHmac('sha256', secret)
    .update(signedContent)
    .digest('hex');

  return {
    'webhook-id': crypto.randomUUID(),
    'webhook-timestamp': timestamp,
    'webhook-signature': `v1=${signature}`,
  };
}
```

### Verifying Inbound Webhooks

```javascript
function verifyWebhookSignature(req, secret) {
  const signature = req.headers['webhook-signature'];
  const timestamp = req.headers['webhook-timestamp'];

  // 1. Check timestamp freshness (prevent replay attacks)
  const age = Math.floor(Date.now() / 1000) - parseInt(timestamp, 10);
  if (Math.abs(age) > 300) {  // 5-minute tolerance
    throw new Error('Webhook timestamp too old');
  }

  // 2. Compute expected signature
  const signedContent = `${timestamp}.${req.rawBody}`;
  const expected = crypto
    .createHmac('sha256', secret)
    .update(signedContent)
    .digest('hex');

  // 3. Constant-time comparison (prevent timing attacks)
  const expectedBuf = Buffer.from(`v1=${expected}`);
  const receivedBuf = Buffer.from(signature);
  if (expectedBuf.length !== receivedBuf.length ||
      !crypto.timingSafeEqual(expectedBuf, receivedBuf)) {
    throw new Error('Invalid webhook signature');
  }

  return true;
}
```

### Webhook Security Rules

- Always use HMAC-SHA256 or stronger
- Include timestamp in signed content to prevent replay
- Use constant-time comparison for signature verification
- Store webhook secrets in secrets manager
- Log webhook receipt (ID, timestamp, status) for audit
- Process webhooks idempotently (same event ID = same result)
- Respond with 200 quickly, process asynchronously
- Validate source IP if provider publishes IP ranges

---

## Testing Authorization: Prove the Test Bites

### The gate lives in the handler, not in the page around it

A layout, a route group, or a proxy that checks the role protects the PAGE. It
does not protect a server action or a route handler rendered inside that page:
those are separate endpoints, callable directly. Moving a control from a
tightly gated area into a widely gated one (say, from an owner-only console into
an admin area that also admits lesser admin roles) silently widens who can call
its action unless the action checks the role itself. Every privileged action
verifies the caller on every call, first, before it reads or writes anything.

### Test the real predicate, over a fake data layer

Mocking `isAdmin()` to return `false` and asserting a refusal tests the mock.
Instead, mock one layer down (the database client) with a small in-memory fake
that returns the caller's role row, and let the real role predicates run. A
refusal in that test is the production decision.

- Cover every tier, not just "admin" and "user": the top role allowed; each
  lesser admin role, and a plain member, refused.
- Assert the refusal happened **before side effects**: the fetch, the write,
  and the audit call were never invoked.
- Make the fake **throw on any table it was not told about**. That turns "the
  handler never looks up X" from a hope into an assertion, which is how you
  prove a handler targets the caller's own tenant rather than looking one up.

### Mutate the gate once, and watch a test fail

A permission test that cannot fail is decoration. After writing it, change the
gate to the next-wider predicate (super admin to any admin, say), run the
tests, and confirm the case for the newly admitted role fails. Then restore the
gate and confirm green. It takes a minute and it is the only evidence that the
test guards the line it claims to guard. Do the restore from a saved copy, not
from memory, and check the diff afterward.

### Open endpoints: the limits are the whole defense

An endpoint with no session and no role has only its limits: per-caller rate,
a global daily cap, message and history size, and an output cap on any model
call. Drive the real route and the real limiter in a test as an anonymous
caller: accepted within limits, refused past each one, with the expensive call
never made on a refusal. Say plainly in the limiter's comment when counters are
in memory per server instance: concurrent instances each count separately and a
cold start resets them, so the bound is a small multiple, not a hard ceiling.

---

## Quick Checklist

- [ ] JWTs validated: signature, exp, nbf, iss, aud, algorithm whitelist
- [ ] Access tokens short-lived (5-15 min), refresh tokens rotated on use
- [ ] API keys: hashed in storage, prefixed, scoped, expiring, revocable
- [ ] OAuth2 using Authorization Code + PKCE; no implicit or password grants
- [ ] CORS: specific origin allowlist, no wildcard with credentials
- [ ] Rate limiting at global, service, and endpoint tiers
- [ ] Every privileged action checks the role itself; the surrounding page's gate does not count
- [ ] Authorization tests run the real predicate, cover every tier, and were seen to fail under a mutated gate
- [ ] Request size limits and timeouts configured
- [ ] Deprecated API versions still patched or blocked
- [ ] Webhook signatures verified with HMAC + timestamp + constant-time compare
- [ ] All API errors return consistent format without internal details
- [ ] API responses include security headers (see backend-security.md)
- [ ] GraphQL: query depth/complexity limits, introspection disabled in production
