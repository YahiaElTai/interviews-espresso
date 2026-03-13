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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What is the difference between hashing and encryption, and between symmetric and asymmetric cryptography — why is hashing irreversible (and why that matters for passwords), when would you use symmetric vs asymmetric encryption, and how do digital signatures use asymmetric keys to prove authenticity and integrity?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What are the OWASP Top 10 vulnerabilities most relevant to backend engineers — why do categories like injection, broken access control, and SSRF keep appearing on this list year after year, what common development patterns cause them, and what is the overarching defense philosophy (input validation, least privilege, output encoding) that prevents entire classes of vulnerabilities rather than individual attacks?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. Why do JWTs exist and when should you use them vs avoid them — what are the three parts (header, payload, signature), how does signing with HS256 (symmetric) differ from RS256 (asymmetric) in key distribution and verification, and what are the real tradeoffs of stateless tokens that make teams regret choosing JWTs for certain use cases?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How do JWTs compare to session-based authentication — what are the tradeoffs in scalability (stateless vs server-side state), how does revocation work differently (token blocklisting vs session deletion), and why do some teams regret choosing JWTs for use cases where sessions would have been simpler?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What makes session management secure — why are the HttpOnly, Secure, and SameSite cookie flags each necessary, what is session fixation and how does session regeneration after login prevent it, and what other session attacks exist (session hijacking, session replay)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How does token rotation work with refresh tokens — why do access tokens need short lifetimes while refresh tokens are long-lived, what is refresh token reuse detection and how does token family tracking catch stolen refresh tokens, and what happens when a client legitimately retries a refresh request?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How does OAuth 2.0 work — walk through the Authorization Code flow step by step (who talks to whom and why), what is PKCE and why is it required for public clients (SPAs, mobile), and why was the Implicit flow deprecated (what vulnerability did it create)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What does OpenID Connect (OIDC) add on top of OAuth 2.0 — what is the difference between an ID token and an access token (who is each for), when should you use OIDC vs plain OAuth 2.0, and why has OIDC replaced SAML for most modern applications?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How should API key authentication be designed — how should you generate keys (randomness, entropy), why must keys be hashed in storage (not stored in plaintext), how do you handle rotation without breaking existing clients, and what are the security risks of API keys compared to OAuth tokens?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why should you use bcrypt, scrypt, or argon2 for password hashing instead of SHA-256 — what makes these algorithms deliberately slow (cost factors, memory-hardness), what do salt and pepper each protect against, and how do you choose the right cost factor that balances security with login latency?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. How does the browser's same-origin policy work and why does CORS exist to relax it — what triggers a preflight OPTIONS request vs a simple request, how does Access-Control-Allow-Origin interact with credentialed requests (why can't you use a wildcard with credentials), and what are the most common CORS misconfigurations that create security vulnerabilities?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why is role-based access control (RBAC) the dominant authorization model for backend APIs — what problem does it solve that simple user-level checks don't, how does the user → roles → permissions hierarchy work, what's the critical difference between checking roles vs checking permissions (and why checking roles directly is a common mistake), and when does RBAC fall short requiring ABAC or policy engines like OPA?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Why is rate limiting essential specifically for authentication endpoints beyond general API rate limiting — what threat models does it address (credential stuffing, brute force, account enumeration), how do per-IP vs per-account vs combined strategies differ in effectiveness, and what are the tradeoffs between blocking, throttling, and CAPTCHA escalation?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Authentication Implementation

<details>
<summary>15. Implement JWT-based authentication with RS256 in a Node.js API — show the token generation (signing with a private key), verification middleware (verifying with a public key), and explain why RS256 is preferred over HS256 for microservices, how to handle token expiration, and what claims to include in the payload</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Implement session-based authentication with secure cookie configuration in a Node.js application — show the session creation on login, the cookie settings (HttpOnly, Secure, SameSite, path, maxAge), session regeneration after authentication, and explain what each flag prevents</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Implement refresh token rotation with reuse detection — show the flow where a refresh token is single-use (exchanged for a new access + refresh token pair), how to detect when a previously-used refresh token is resubmitted (indicating theft), and how token family tracking invalidates all tokens in the compromised family</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Implement the OAuth 2.0 Authorization Code flow with PKCE for a single-page application — show the authorization request (with code_challenge), the callback that exchanges the code for tokens (with code_verifier), and explain why PKCE prevents the authorization code interception attack that made the Implicit flow insecure</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Build an API key authentication system — show how to generate cryptographically random keys, store them hashed (with a prefix for identification), verify incoming keys efficiently, implement key rotation (issuing a new key while the old one remains valid for a grace period), and explain the tradeoffs vs bearer tokens</summary>

<!-- Answer will be added later -->

</details>

## Practical — Security Hardening

<details>
<summary>20. Demonstrate SQL injection on a vulnerable query and fix it with parameterized queries — show the vulnerable code (string concatenation), the attack payload, the fix using parameterized/prepared statements (with Prisma, raw SQL, or a query builder), and explain why parameterized queries are immune to injection while escaping is not reliable</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Implement input validation at an API boundary using Zod or Joi in a Node.js API — show how schema validation prevents mass assignment and injection attacks, demonstrate output encoding that prevents stored XSS when user input is rendered, explain the defense-in-depth principle (why you need both input validation AND output encoding), and show how a poorly written validation regex can create a ReDoS vulnerability</summary>

<!-- Answer will be added later -->

</details>

## Practical — Access Control & Production

<details>
<summary>22. Implement RBAC middleware for a Node.js API — show the role and permission model (user → roles → permissions), the middleware that checks permissions on each route, how to handle hierarchical roles (admin inherits editor permissions), and what the common mistakes are (checking roles instead of permissions, hardcoding role names)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Implement rate limiting and brute-force protection for a login endpoint — show the middleware that limits login attempts per IP and per account (using Redis for distributed state), implement progressive delays or temporary lockouts after repeated failures, configure request payload size limits to prevent oversized-body DoS attacks, and explain how attackers bypass naive rate limiting (distributed attacks, credential stuffing)</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you designed or significantly improved an authentication system — what approach did you choose (JWT, sessions, OAuth), what tradeoffs did you face, and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Describe a security vulnerability you discovered or fixed in a production application — how did you find it, what was the impact, and what changes did you make to prevent similar issues?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>26. Describe a time you had to balance security requirements with developer experience or business needs — what was the tradeoff, how did you make the decision, and what was the outcome?</summary>

<!-- Answer framework will be added later -->

</details>
