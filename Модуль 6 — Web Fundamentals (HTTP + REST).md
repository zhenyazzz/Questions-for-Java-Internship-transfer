# Module 6 — Web Fundamentals (HTTP / REST) — Questions 85–94

> Questions 85–94 from *Questions for Java Internship transfer.pdf* (client–server, HTTP/TLS, REST constraints, verbs and idempotency, status codes, cookies vs sessions, CORS, SOAP, gRPC vs REST). **Note:** questions **86** (HTTP vs HTTPS) and **92** (CORS) overlap thematically with **63** and **58** in `Модуль 4 — Web (Spring MVC + Security).md`; here the focus is **protocol and web platform** behavior, not Spring configuration.

> Each numbered section ends with a **Middle+ answer** you can adapt on interview (monologue-style); bullets and tables above are for follow-up drilling.

## Module summary (middle+ — how I’d frame it on interview)

<a id="summary"></a>

The web platform is **client–server over HTTP**: the **client initiates** every interaction; the **server** holds authoritative **data, invariants, and access control**. HTTP is not only “a wire format”—it carries **method semantics** (safe vs unsafe, idempotent vs not), **headers** (caching, auth, content negotiation), and **status codes** that tell clients, caches, and operators **what happened** and **who should react**. **HTTPS** wraps that same HTTP in **TLS**, so you get **confidentiality**, **integrity**, and **authenticated endpoints** (certificates; optional **mTLS** for clients).

**REST** is an **architectural style** on top of HTTP (Fielding): **resources** identified by URLs, **representations** (often JSON), **stateless** processing, **cacheable** responses where possible, and a **uniform interface**. In production most “REST” APIs are **pragmatic**: CRUD-ish JSON, sometimes **RPC-shaped** URLs—acceptable if the team is explicit. What separates middle+ from junior answers is connecting this to **behavior under failure**: **PUT vs POST** and **idempotency** decide whether **retries** from gateways or clients are safe; **PATCH** is not magically idempotent; **status codes** drive **client retries**, **CDN behavior**, and **alerting**; **cookies vs sessions vs bearer tokens** is a **threat-model** choice (XSS, CSRF, storage); **CORS** is a **browser read policy**, not API security; **SOAP / gRPC / REST** is a **consumer and contract** choice (browser, tooling, streaming, governance).

Spring-specific TLS, security headers, and CORS wiring overlap with **`Модуль 4 — Web (Spring MVC + Security).md`** (**questions 63** and **58**); this module stays at **protocol + platform** level.

### Quick map (question → one line)

| # | Topic | One-line takeaway |
|---|--------|-------------------|
| **85** | Client–server | Clients initiate; servers are authoritative; roles are logical (a service is a client to its upstream). |
| **86** | HTTP vs HTTPS | HTTPS = HTTP + TLS (confidentiality, integrity, authentication); baseline for anything real. |
| **87** | REST | Architectural constraints on HTTP; statelessness scales; many “REST” APIs relax HATEOAS / purity. |
| **88** | PUT vs POST | Known resource URI + idempotent replace/create vs collection/action + server-chosen id; duplicates on POST. |
| **89** | Idempotent methods | Safe reads vs mutating idempotent vs POST; PATCH is implementation-defined; drives retry safety. |
| **90** | Status codes | Outcome + who fixes it; 401 vs 403; 404 can hide existence; 5xx for unexpected server-side failure. |
| **91** | Cookies & sessions | Cookie = transport/storage; session = server-side state keyed by id (often cookie); harden flags. |
| **92** | CORS | Browser-only cross-origin read policy; server opts in via headers; not curl/Postman security. |
| **93** | SOAP | XML + WSDL + enterprise standards; legacy / regulated / strict interop—not wrong, often contextual. |
| **94** | gRPC vs REST | gRPC: HTTP/2, protobuf, streaming, codegen—great service-to-service; REST+JSON: web/public/debuggability. |

## Table of contents

- [Module summary (middle+)](#summary)

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

### 4. Middle+ answer (as on interview)

Client–server architecture is a model where a client sends requests and a server processes them and returns responses.

The client is typically responsible for the user interface or initiating actions, while the server handles business logic, data storage, and security.

The interaction is request–response based:

the client sends a request
the server processes it
the server returns a response

An important detail is that “client” and “server” are logical roles. A backend service can act as a server for one request and as a client when calling another service.

HTTP is the most common protocol used in this model. It defines how requests and responses are structured and exchanged over the network.

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

### 4. Middle+ answer (as on interview)

The main difference between HTTP and HTTPS is security.

HTTP sends data in plain text, meaning it can be intercepted or modified by anyone who has access to the network
HTTPS is HTTP over TLS, which provides encryption, data integrity, and authentication

With HTTPS:

data is encrypted → cannot be easily read
integrity is protected → tampering is detected
the server is authenticated using certificates

In practice, HTTPS is the standard for all modern applications, especially when handling sensitive data like authentication tokens or personal information.

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

### 4. Middle+ answer (as on interview)

REST (Representational State Transfer) is an architectural style for designing network APIs, typically using HTTP.

It is based on working with resources, which are identified by URLs, and manipulated using standard HTTP methods like GET, POST, PUT, and DELETE.

The main principles of REST are:

Client–server — separation between client and server responsibilities
Statelessness — each request contains all necessary information; the server does not store client state between requests
Cacheability — responses can be cached to improve performance
Uniform interface — consistent use of URLs, HTTP methods, and status codes
Layered system — intermediaries like proxies or gateways can exist between client and server

In practice, many APIs follow REST principles loosely and behave more like RPC over HTTP, which is acceptable as long as the design remains consistent.

There is also an optional constraint called code-on-demand, where the server can send executable code to the client.

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

### 4. Middle+ answer (as on interview)

The main difference between PUT and POST is how they handle resource creation and idempotency.

PUT is used to create or replace a resource at a known URI. The client specifies the resource identifier, and the operation is idempotent, meaning repeating the same request does not change the result
POST is used to create a new resource or trigger processing. The server usually generates the resource identifier, and the operation is not idempotent, meaning repeated requests may create multiple resources

Examples:

PUT /users/42 → create or replace user with ID 42
POST /users → create a new user, server assigns ID

In practice, PUT is used when the client knows the resource identity, while POST is used when the server controls resource creation.

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

### 4. Middle+ answer (as on interview)

Idempotent methods are HTTP methods where repeating the same request multiple times has the same effect as executing it once.

The main idempotent methods are:

GET, HEAD, OPTIONS — safe and idempotent
PUT — idempotent, as it replaces a resource
DELETE — idempotent, as deleting the same resource multiple times does not change the result

Non-idempotent methods:

POST — typically creates new resources, so repeated requests may produce duplicates
PATCH — not guaranteed to be idempotent, depends on implementation

In practice, idempotency is important for retries. Idempotent methods can be safely retried without causing unintended side effects, while non-idempotent methods require additional mechanisms like idempotency keys.

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

### 3. Middle+ answer (as on interview)

These HTTP status codes indicate the result of a request and are divided into success, client errors, and server errors.

200 OK — the request succeeded, and the response contains the result
201 Created — a new resource was successfully created, often with a Location header
400 Bad Request — the request is invalid, for example due to malformed input or validation errors
401 Unauthorized — the request is not authenticated (missing or invalid credentials)
403 Forbidden — the user is authenticated but does not have permission
404 Not Found — the requested resource does not exist
500 Internal Server Error — an unexpected error occurred on the server

In practice:

4xx errors indicate problems with the client request
5xx errors indicate problems on the server side

Status codes are important because they guide client behavior, such as retries and error handling.

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

### 4. Middle+ answer (as on interview)

A cookie is a small piece of data stored in the client’s browser and sent with each request to the server.

A session is server-side data that stores user-specific information, such as authentication state or a shopping cart.

They are typically used together:

the server creates a session and stores data on the server
the client receives a session ID in a cookie
on each request, the browser sends the cookie back
the server uses the session ID to retrieve the session data

So:

cookie = stored on the client
session = stored on the server

Sessions introduce server-side state, which can complicate horizontal scaling.

In practice, cookies are just a transport mechanism, while sessions hold the actual state.

In modern APIs, server-side sessions are often replaced with stateless authentication (e.g., JWT), but sessions are still used in some systems.

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

### 4. Middle+ answer (as on interview)

CORS (Cross-Origin Resource Sharing) is a browser mechanism that allows or restricts requests between different origins.

By default, browsers enforce the Same-Origin Policy, which means a web page can only access resources from the same domain, protocol, and port.

CORS allows a server to explicitly permit requests from other origins using HTTP headers like:

Access-Control-Allow-Origin
Access-Control-Allow-Methods
Access-Control-Allow-Headers

For more complex requests, the browser sends a preflight (OPTIONS) request to check if the server allows the operation.

CORS is needed because modern applications often have a frontend and backend on different domains, and they must communicate securely.

CORS does not prevent sending requests, it prevents a malicious website from reading responses from another origin in the browser.

Important: CORS is enforced only by browsers and does not replace authentication or security on the server.
CORS protects users by preventing malicious websites from reading sensitive data from other domains.

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

### 4. Middle+ answer (as on interview)

SOAP (Simple Object Access Protocol) is a protocol for exchanging structured messages using XML, typically over HTTP.

It uses a strict contract defined by WSDL, which describes available operations and data formats. This allows automatic code generation and strong typing.

SOAP is mainly used in:

enterprise systems
banking and government integrations
legacy systems that require strict contracts and standards

Advantages:

strong contracts and formal standards
built-in support for security and reliability

Disadvantages:

verbose XML format
higher complexity and slower development compared to REST

In practice, SOAP is still used in regulated or legacy environments, while most modern systems prefer REST or gRPC.

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

### 4. Middle+ answer (as on interview)

gRPC is a high-performance RPC framework that uses HTTP/2 and Protocol Buffers (protobuf) for communication.

It defines APIs using .proto files and generates strongly typed client and server code.

REST, on the other hand, is an architectural style that typically uses HTTP + JSON and works with resources.

Key differences:

Data format — REST uses text-based JSON, while gRPC uses compact binary protobuf
Performance — gRPC is faster and more efficient due to binary format and HTTP/2
Streaming — gRPC supports streaming natively; REST requires additional mechanisms
Contract — gRPC uses strict schema (.proto), while REST is more flexible
Browser support — REST works directly in browsers; gRPC requires additional tools (e.g., grpc-web)

When to use:

use gRPC for internal microservice communication where performance and strict contracts matter
use REST for public APIs and browser clients where simplicity and compatibility are important

gRPC is a contract-first RPC framework where APIs are defined using protobuf. It provides strong typing, efficient binary communication, and supports streaming, making it suitable for internal microservices.

---
