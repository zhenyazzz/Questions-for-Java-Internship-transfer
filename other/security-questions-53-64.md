# Security — Questions 53–64

Interview-ready answers in English for authentication, authorization, web security, and common attacks.

## Table of Contents

53. [What is the difference between Authorization and Authentication?](#q53)
54. [What is the difference between hashing and crypting?](#q54)
55. [What is the structure of JWT token?](#q55)
56. [How does OAuth 2.0 work? What are the roles in OAuth 2.0?](#q56)
57. [How does OpenId work?](#q57)
58. [What is CORS?](#q58)
59. [How to handle Security exceptions?](#q59)
60. [What is SQL Injection?](#q60)
61. [What is an XSS attack?](#q61)
62. [What is an CSRF attack?](#q62)
63. [What is the difference between HTTP and HTTPS?](#q63)
64. [What is a Brute Force attack?](#q64)

---

<a id="q53"></a>
## 53. What is the difference between Authorization and Authentication?

- **Authentication** = proving identity (“Who are you?”).  
  Example: login with password, MFA, OAuth login.
- **Authorization** = checking permissions (“What are you allowed to do?”).  
  Example: user can read orders but cannot delete them.

Order matters: first authenticate, then authorize.

Common interview example:
- AuthN succeeds (valid token), but AuthZ fails (missing role) -> `403 Forbidden`.

---

<a id="q54"></a>
## 54. What is the difference between hashing and crypting?

You should usually say **hashing vs encryption**.

- **Hashing**
  - One-way function (cannot be reversed practically).
  - Used for password storage and integrity checks.
  - Same input -> same hash.
  - Use salt + strong algorithms (`bcrypt`, `scrypt`, `Argon2`).
- **Encryption**
  - Two-way (encrypt and decrypt with key).
  - Used to protect confidential data in transit or at rest.
  - Symmetric (AES) or asymmetric (RSA/ECC).

Rule:
- Passwords -> hash (never decrypt).
- Secrets that must be read later -> encrypt.

---

<a id="q55"></a>
## 55. What is the structure of JWT token?

JWT has 3 Base64Url-encoded parts:
1. **Header**
   - token type (`typ`: `JWT`)
   - algorithm (`alg`: e.g. `HS256`, `RS256`)
2. **Payload**
   - claims (`sub`, `iss`, `aud`, `exp`, `iat`, custom claims)
3. **Signature**
   - computed from `base64Url(header) + "." + base64Url(payload)` and key

Format:
`header.payload.signature`

Important:
- JWT is usually **signed**, not encrypted.
- Anyone can decode header/payload; do not put sensitive plaintext data there.
- Always validate signature + expiration + issuer/audience.

---

<a id="q56"></a>
## 56. How does OAuth 2.0 work? What are the roles in OAuth 2.0?

OAuth 2.0 is a framework for **delegated authorization**.

### Roles
- **Resource Owner**: user who owns data.
- **Client**: application requesting access.
- **Authorization Server**: issues tokens after user consent/authentication.
- **Resource Server**: API that validates token and serves protected data.

### Typical flow (Authorization Code + PKCE)
1. Client redirects user to Authorization Server.
2. User authenticates and consents.
3. Client gets authorization code.
4. Client exchanges code for access token (and optionally refresh token).
5. Client calls Resource Server with bearer token.

Important:
- OAuth is about authorization, not identity itself.
- For user identity, add OpenID Connect.

---

<a id="q57"></a>
## 57. How does OpenId work?

Usually interview means **OpenID Connect (OIDC)** on top of OAuth 2.0.

OIDC adds identity layer:
- introduces **ID Token** (JWT with user identity claims),
- standard scopes (`openid`, `profile`, `email`),
- user info endpoint.

Flow:
1. App starts OAuth2 Authorization Code flow with `openid` scope.
2. Authorization server (now OpenID Provider) returns access token + ID token.
3. App verifies ID token (signature, issuer, audience, expiration, nonce).
4. App establishes authenticated user session.

Short: OAuth2 gives delegated access; OIDC gives standardized login/identity.

---

<a id="q58"></a>
## 58. What is CORS?

**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism controlling whether a frontend from Origin A can call backend from Origin B.

Why needed:
- Browsers enforce Same-Origin Policy by default.

How it works:
- Browser sends request with `Origin` header.
- Server responds with headers like:
  - `Access-Control-Allow-Origin`
  - `Access-Control-Allow-Methods`
  - `Access-Control-Allow-Headers`
  - `Access-Control-Allow-Credentials` (if cookies/auth credentials allowed)
- For non-simple requests browser sends preflight `OPTIONS`.

Security best practice:
- avoid wildcard `*` for sensitive APIs,
- explicitly allow trusted origins.

---

<a id="q59"></a>
## 59. How to handle Security exceptions?

In Spring Security (typical Java interview):
- **Authentication failures** (not logged in / bad token) -> handled by `AuthenticationEntryPoint` -> usually `401`.
- **Access denied** (logged in but insufficient permissions) -> handled by `AccessDeniedHandler` -> usually `403`.

Best practice:
- return consistent JSON error format,
- do not leak internal security details,
- log with trace/correlation ID.

For MVC/global errors also use:
- `@ControllerAdvice` / `@RestControllerAdvice`,
- custom exception mapping and unified error DTO.

---

<a id="q60"></a>
## 60. What is SQL Injection?

SQL Injection is an attack where malicious input changes SQL query logic.

Bad example:
```sql
SELECT * FROM users WHERE login = '" + input + "' AND password = '" + pass + "'
```
Input like `' OR '1'='1` can bypass checks.

Prevention:
- always use parameterized queries (`PreparedStatement`, ORM bind parameters),
- never concatenate untrusted input into SQL,
- use least-privilege DB accounts,
- validate/whitelist inputs.

---

<a id="q61"></a>
## 61. What is an XSS attack?

**XSS (Cross-Site Scripting)** happens when attacker injects script into pages viewed by other users.

Types:
- Stored XSS (saved in DB, served later),
- Reflected XSS (immediate response),
- DOM XSS (client-side JS vulnerability).

Impact:
- session/token theft,
- actions on behalf of user,
- UI defacement/phishing.

Prevention:
- output encoding/escaping by context (HTML/JS/URL),
- sanitize rich text input,
- Content Security Policy (CSP),
- avoid unsafe `innerHTML`.

---

<a id="q62"></a>
## 62. What is an CSRF attack?

**CSRF (Cross-Site Request Forgery)** tricks authenticated browser into sending unwanted request to trusted site.

Condition:
- browser automatically sends cookies/session.

Example:
- user logged in bank app,
- visits malicious page,
- hidden form triggers money transfer request.

Prevention:
- CSRF tokens (synchronizer token pattern),
- `SameSite` cookies,
- check Origin/Referer where suitable,
- for APIs prefer stateless bearer tokens in `Authorization` header instead of cookie auth (still design carefully).

---

<a id="q63"></a>
## 63. What is the difference between HTTP and HTTPS?

- **HTTP**
  - plain text,
  - no encryption/integrity/authentication.
- **HTTPS**
  - HTTP over TLS,
  - provides confidentiality, integrity, and server authentication via certificate.

HTTPS protects against eavesdropping and many MITM attacks.

Best practice:
- enforce HTTPS everywhere,
- redirect HTTP -> HTTPS,
- use HSTS.

---

<a id="q64"></a>
## 64. What is a Brute Force attack?

Brute force is repeated guessing of passwords/OTP/keys until one works.

Typical targets:
- login forms,
- API tokens,
- SSH/RDP endpoints.

Mitigations:
- rate limiting / throttling,
- account lockout or progressive delays,
- MFA,
- CAPTCHA (careful UX),
- monitoring and IP reputation,
- strong password policy and hashed password storage.

Interview point: credential stuffing (using leaked passwords) is related and often more realistic than pure random brute force.

