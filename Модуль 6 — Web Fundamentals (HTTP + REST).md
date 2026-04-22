# Module 6 — Web Fundamentals (HTTP / REST) — Questions 85–94

> Questions 85–94 from *Questions for Java Internship transfer.pdf* (client–server, HTTP/TLS, REST constraints, verbs and idempotency, status codes, cookies vs sessions, CORS, SOAP, gRPC vs REST). **Note:** questions **86** (HTTP vs HTTPS) and **92** (CORS) overlap thematically with **63** and **58** in `Модуль 4 — Web (Spring MVC + Security).md`; here the focus is **protocol and web platform** behavior, not Spring configuration.

## Table of contents

85. [What is client-server architecture?](#q85)
86. [What is the difference between HTTP and HTTPS?](#q86)
87. [What is REST and its main principles?](#q87)
88. [What is the difference between PUT and POST?](#q88)
89. [What are idempotent methods in HTTP?](#q89)
90. [What is the difference between statuses 200, 201, 400, 401, 403, 404, 500?](#q90)
91. [What are cookies and sessions?](#q91)
92. [What is CORS and why is it needed?](#q92)
93. [What is SOAP and in what cases is it used?](#q93)
94. [What is gRPC and how does it differ from REST?](#q94)

---

<a id="q85"></a>

## 85. What is client-server architecture?

### Question Restatement
What is **client–server** architecture, and how does HTTP fit into it?

### 1. Definition
**Client–server** is a structural style where **clients** initiate work and **servers** provide **resources or services**. The server **listens** on a network address; the client **connects**, sends a **request**, receives a **response**, and disconnects or reuses the connection depending on the protocol.

### 2. Roles and boundaries

| Role | Typical responsibility |
|------|-------------------------|
| **Client** | UI or calling program; builds requests; interprets responses. |
| **Server** | Authoritative data and rules; scales and protects the domain. |

**Important nuance:** “client” and “server” are **logical** roles for a conversation. A backend service calling another API is a **client** to that upstream.

### 3. Interview nuance
Contrast with **peer-to-peer** (symmetric nodes) and **monolith vs microservices** only when asked—client–server is about **who initiates** and **where stateful work** usually lives, not about deployment shape alone.

### 4. Summary (Short speaking answer)
Client–server architecture means that clients request services or data from servers over the network, and servers process those requests and return responses.

In web systems, browsers or mobile apps are typical clients, and HTTP servers expose APIs or pages. The client is responsible for presentation and user interaction, while the server holds business logic and data and enforces access rules.

The key idea is a clear separation of responsibilities: clients initiate communication; servers provide centralized, controlled access to resources.

---

<a id="q86"></a>

## 86. What is the difference between HTTP and HTTPS?

### Question Restatement
Compare **HTTP** and **HTTPS** at the **transport** level.

### 1. HTTP
**Hypertext Transfer Protocol** without TLS sends **metadata and body in cleartext** (from the perspective of anyone who can observe the link). Integrity and peer authentication are **not** guaranteed by the protocol itself.

### 2. HTTPS
**HTTPS** is **HTTP over TLS** (historically “SSL”). TLS provides **confidentiality** (encryption), **integrity** (tampering detection), and **server authentication** via **X.509** certificates (optional **mTLS** for client identity).

### 3. Production reality
Use **HTTPS everywhere** for user-facing apps; add **HSTS**, modern TLS versions, and correct **certificate chain** validation. Misconfigured TLS (weak ciphers, expired certs) breaks availability, not only security.

### 4. Summary (Short speaking answer)
HTTP transfers messages in plain form on the wire, so eavesdroppers can read or alter traffic. HTTPS wraps HTTP in TLS, encrypting traffic and verifying the server’s identity with certificates.

Browsers show warnings or block mixed content when HTTPS expectations are violated. For APIs, HTTPS protects tokens and payloads in transit.

The key idea is that HTTPS adds cryptography and trust establishment to HTTP; it is the default baseline for anything sensitive. For the same topic in a Spring Security context, see **question 63** in `Модуль 4 — Web (Spring MVC + Security).md`.

---

<a id="q87"></a>

## 87. What is REST and its main principles?

### Question Restatement
What is **REST**, and which **architectural constraints** does it emphasize?

### 1. Definition
**REST** (Representational State Transfer) is an **architectural style** for distributed hypermedia systems, described by Fielding. **RESTful HTTP APIs** apply REST ideas using **resources**, **representations** (often JSON), and **standard methods** (`GET`, `POST`, etc.).

### 2. Core constraints (speaking list)
- **Client–server** — separation of UI and storage/processing.
- **Statelessness** — each request from client to server contains **all** information needed to understand it; server does not store **client conversational state** in memory between requests (session state may live in tokens, DB, etc., but is **explicit** in the request).
- **Cacheability** — responses declare cacheability (`Cache-Control`, `ETag`, …).
- **Uniform interface** — consistent identification of resources (URLs), manipulation through representations, **self-descriptive messages**, **HATEOAS** (hypermedia as the engine of application state) in strict REST—often **weakened** in JSON CRUD APIs.
- **Layered system** — proxies, gateways, load balancers transparent to the client.

### 3. Interview nuance
Many “REST” APIs are **RPC-over-HTTP** (action URLs, verbs in path). That can be fine operationally; interviewers often check whether you know **the difference** between **style** and **marketing**.

### 4. Summary (Short speaking answer)
REST is an architectural style for networked systems, not a single framework. It favors resource-oriented thinking, stateless request processing, a uniform interface using HTTP semantics, cache-friendly responses, and layered infrastructure.

Practical REST APIs usually use nouns for resources, HTTP methods for intent, and status codes for outcomes—though not every JSON API strictly follows HATEOAS or pure hypermedia.

The key idea is predictable, scalable interaction over HTTP by constraining how clients and servers exchange representations of resources.

---

<a id="q88"></a>

## 88. What is the difference between PUT and POST?

### Question Restatement
When do you use **PUT** versus **POST**?

### 1. Conventional semantics
- **PUT** — **replace** or **create** a resource at a **known URI**; target is the **resource identity** in the URL. Intended **idempotent**: repeating the same PUT should not keep changing server state beyond the first successful application.
- **POST** — **submit data** to a **processor** or **collection**; server often **chooses** the new resource URI (`201` + `Location`). **Not** guaranteed idempotent—duplicate POSTs may create duplicates unless you add **idempotency keys** or deduplication.

### 2. Practical examples
- **PUT `/users/42`** — create or replace user 42 with the enclosed body.
- **POST `/users`** — “add a user to the collection”; server assigns id.

### 3. When it breaks
If the team uses **POST for everything** “because it is easier,” you lose **caching** and **intermediary** semantics. If you use **PUT** without stable URIs, you are really doing RPC.

### 4. Summary (Short speaking answer)
PUT targets a specific resource URI and is used to upload the full representation of that resource, with idempotent semantics in the HTTP model. POST submits an action or creates a subordinate resource when the server assigns the identity, and repeated calls may have different effects.

In real APIs, teams sometimes bend rules; what matters is documenting behavior and using idempotency where duplicates are costly.

The key idea is **who owns the resource identity** (client-known URI → PUT; server-assigned → POST) and **idempotency expectations**.

---

<a id="q89"></a>

## 89. What are idempotent methods in HTTP?

### Question Restatement
Which HTTP methods are **idempotent**, and what does that mean for **retries**?

### 1. Definition
**Idempotent** methods: **multiple identical requests** have the **same effect on server-side state** as **one** successful request (from the client’s perspective for the abstract resource model).

### 2. Standard classification (typical teaching answer)

| Method | Safe (no side effects) | Idempotent |
|--------|------------------------|------------|
| **GET** | Yes | Yes |
| **HEAD** | Yes | Yes |
| **OPTIONS** | Yes | Yes |
| **PUT** | No | Yes |
| **DELETE** | No | Yes |
| **POST** | No | Usually **no** |
| **PATCH** | No | **Not guaranteed** — depends on implementation |

### 3. Production nuance
**Network retries** (gateway, client library) are safer for **idempotent** methods. For **POST**, use **idempotency-Key** headers or dedupe business logic. **PATCH** idempotency is **not** promised by the spec—document your resource-specific behavior.

### 4. Summary (Short speaking answer)
Idempotent HTTP methods can be repeated without changing the outcome beyond the first successful application. GET, HEAD, OPTIONS are safe and idempotent; PUT and DELETE are idempotent but not safe; POST is generally not idempotent; PATCH may or may not be, depending on how the server applies partial updates.

This matters for retries, proxies, and resilient clients—the infrastructure can resend idempotent requests more safely.

The key idea is separating **read-only** operations from **state-changing** ones, and among mutating methods, knowing which repeats are **safe for automation**.

---

<a id="q90"></a>

## 90. What is the difference between statuses 200, 201, 400, 401, 403, 404, 500?

### Question Restatement
What do these **HTTP status codes** mean, and when do you return each?

### 1. Quick map

| Code | Class | Meaning (typical) |
|------|--------|-------------------|
| **200** | Success | OK — request succeeded; body often carries the result (including **successful DELETE** with no body). |
| **201** | Success | **Created** — resource created; often includes **`Location`** and representation. |
| **400** | Client error | **Bad Request** — malformed syntax or failed **validation** of input. |
| **401** | Client error | **Unauthorized** — **not authenticated** (missing/invalid credentials). |
| **403** | Client error | **Forbidden** — authenticated but **not allowed** for this action/resource. |
| **404** | Client error | **Not Found** — resource does not exist **or** you hide existence (**authorization privacy**). |
| **500** | Server error | **Internal Server Error** — unexpected failure on server. |

### 2. Interview nuance
- **401 vs 403:** “who are you?” vs “I know you, but no.”
- **404 vs 403 for hidden resources:** returning **404** avoids **resource enumeration**; pick consistently with product/security policy.
- **500 vs 4xx:** programmer bugs / downstream collapse → **5xx**; bad client input → **4xx**.

### 3. Summary (Short speaking answer)
200 means the request succeeded in a generic way. 201 means a new resource was created, commonly after POST. 400 means the client sent invalid or unprocessable input. 401 means authentication is required or failed. 403 means the user is authenticated but not authorized. 404 means the requested resource is not available at that URI. 500 means an unexpected error occurred on the server.

Using the right code helps clients, caches, and operators behave correctly and debug faster.

The key idea is that status codes communicate **outcome class** and **who should fix the problem**—client, auth layer, or server.

---

<a id="q91"></a>

## 91. What are cookies and sessions?

### Question Restatement
What are **cookies** and **sessions**, and how do they relate?

### 1. Cookies
A **cookie** is a **small piece of data** the server asks the browser to store via **`Set-Cookie`**, sent back on later requests in the **`Cookie`** header (scoped by **domain**, **path**, **`SameSite`**, **`Secure`**, **`HttpOnly`**, expiry).

### 2. Sessions
A **session** is **server-side conversational state**: the server remembers the user across requests (shopping cart, login state). The client usually holds only a **session identifier** (often in a cookie).

| Mechanism | Where state lives | Typical transport |
|-----------|-------------------|-------------------|
| **Cookie** | Client storage | HTTP headers |
| **Session** | Server store (memory, Redis, DB) | Session id in cookie or URL (avoid URL) |

### 3. Security
Prefer **`HttpOnly`** session cookies to reduce **XSS** token theft; pair with **`Secure`** and sensible **`SameSite`**. For APIs, **Bearer tokens** in memory/header trade off different XSS/CSRF risks—know both models.

### 4. Summary (Short speaking answer)
Cookies are name-value pairs stored by the browser and attached to matching HTTP requests. Sessions are server-side state keyed by an identifier; the browser often carries that id as a session cookie.

Cookies can also be used for non-session things (preferences, tracking). Sessions expire or invalidate on logout or timeout.

The key idea is **cookies are a transport/storage mechanism**; **sessions are server-side continuity** usually bound to the client via a cookie or token.

---

<a id="q92"></a>

## 92. What is CORS and why is it needed?

### Question Restatement
What is **CORS**, and which problem does it solve?

### 1. Same-Origin Policy
Browsers enforce the **Same-Origin Policy** for JavaScript-initiated requests: by default, a page at `https://a.com` cannot read responses from `https://b.com`—that limits **data theft** from random sites in the user’s browser.

### 2. CORS
**Cross-Origin Resource Sharing** lets a **server opt in** with HTTP headers (`Access-Control-Allow-Origin`, methods, headers, credentials). **Preflight** `OPTIONS` requests validate non-simple cross-origin requests before sending the real request.

### 3. Why “needed”
Without CORS, **legitimate** multi-domain frontends (SPA on CDN, API on another host) could not cooperate safely. CORS is **not** a substitute for **authentication**—it is a **browser** policy; **curl/Postman** ignore CORS.

### 4. Summary (Short speaking answer)
CORS is a browser mechanism that relaxes the same-origin policy when the target server explicitly allows another origin through response headers.

It is needed so modern web apps can call APIs on different origins while still preventing malicious sites from silently reading private responses from other domains the user is logged into.

The key idea is **CORS controls browser-readable cross-origin access**, not general API security. Spring-specific configuration overlaps with **question 58** in `Модуль 4 — Web (Spring MVC + Security).md`.

---

<a id="q93"></a>

## 93. What is SOAP and in what cases is it used?

### Question Restatement
What is **SOAP**, and where is it still relevant?

### 1. Definition
**SOAP** is a **protocol** for exchanging **XML** messages, usually over **HTTP** but not limited to it. Contracts are formal (**WSDL**), tooling generates **stubs/skeletons**, and features like **WS-Security**, **transactions**, and **reliable messaging** were standardized for enterprise integration.

### 2. When it is used
- **Enterprise / government** integrations with **strict contracts** and **XML** pipelines.
- **Legacy** systems and **ESB**-centric landscapes.
- Requirements for **machine-readable contract** + **rich standards** beyond typical JSON REST.

### 3. Trade-offs
**Pros:** strong typing, mature **interop** stacks, cross-language tooling. **Cons:** verbose XML, higher **latency** and **payload** size, steeper **DX** compared to JSON REST for many teams.

### 4. Summary (Short speaking answer)
SOAP is an XML-based messaging protocol with a formal contract, often described by WSDL, and a stack of standards for security and reliability.

It is still common where enterprises need strict contracts, auditability, or integration with legacy platforms, even though many greenfield public APIs prefer REST or gRPC.

The key idea is **SOAP optimizes for contract rigor and standards**; **REST/JSON** often optimizes for simplicity and web ergonomics.

---

<a id="q94"></a>

## 94. What is gRPC and how does it differ from REST?

### Question Restatement
What is **gRPC**, and how does it compare to **REST over HTTP/JSON**?

### 1. gRPC
**gRPC** is a **RPC framework** from Google using **HTTP/2**, **Protocol Buffers** (binary, schema-first **IDL**), **streaming** (client, server, bidirectional), and **strongly typed** stubs generated for many languages.

### 2. REST (typical JSON)
Resource-oriented **HTTP + JSON**, **human-readable**, easy **browser** consumption, broad **caching** with GET, **looser** contracts unless you add **OpenAPI**.

### 3. Comparison

| Aspect | Typical REST + JSON | gRPC |
|--------|---------------------|------|
| **Contract** | Often OpenAPI; informal | `.proto` + codegen |
| **Payload** | Text JSON | Binary protobuf |
| **Streaming** | Awkward (SSE/WebSockets separately) | First-class |
| **Browser** | Native `fetch` | needs **grpc-web** proxy |
| **Debugging** | Easy with curl | Often grpcurl / tools |

### 4. Summary (Short speaking answer)
gRPC is a high-performance RPC framework built on HTTP/2 and protobuf, with first-class streaming and generated clients and servers.

REST usually means HTTP with JSON resources, which is simpler for public web APIs and human inspection, while gRPC fits internal microservice traffic where performance, typed contracts, and streaming matter.

The key idea is **gRPC is contract-first binary RPC**; **REST is resource-oriented HTTP**—choose based on clients, observability, and ecosystem, not hype.

---
