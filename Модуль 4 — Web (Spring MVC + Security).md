# Module 4 — Web (Spring MVC + Security) — Questions 43–64

> Questions 43–64 from *Questions for Java Internship transfer.pdf* (Spring MVC, Thymeleaf, authentication vs authorization, JWT/OAuth/OIDC, CORS, Spring Security, common web attacks). Related: `other/spring-questions-30-52.md` (43–52 overlap), `other/security-questions-53-64.md` (53–64 overlap).

## Table of contents

43. [What is the MVC pattern? Which components MVC consists of?](#q43)
44. [What is the Dispatcher Servlet?](#q44)
45. [What is ModelAndView?](#q45)
46. [How is request routing done in Spring MVC?](#q46)
47. [What is the difference between Controller and RestController annotations?](#q47)
48. [What is the difference between RequestParam and PathVariable annotations?](#q48)
49. [How to handle exceptions in Spring MVC?](#q49)
50. [What is a ViewResolver?](#q50)
51. [How to upload files using Spring MVC?](#q51)
52. [What is Thymeleaf?](#q52)
53. [What is the difference between Authorization and Authentication?](#q53)
54. [What is the difference between hashing and crypting?](#q54)
55. [What is the structure of JWT token?](#q55)
56. [How does OAuth 2.0 work? What are the roles in OAuth 2.0?](#q56)
57. [How does OpenId work?](#q57)
58. [What is CORS?](#q58)
59. [How to handle Security exceptions?](#q59)
60. [What is SQL Injection?](#q60)
61. [What is an XSS attack?](#q61)
62. [What is a CSRF attack?](#q62)
63. [What is the difference between HTTP and HTTPS?](#q63)
64. [What is a Brute Force attack?](#q64)

---

<a id="q43"></a>

## 43. What is the MVC pattern? Which components MVC consists of?

### Question Restatement
What is **Model–View–Controller**, and how does Spring MVC map those roles to HTTP?

### 1. Definition
**MVC** splits UI-related work into three roles:

| Role | Responsibility |
|------|------------------|
| **Model** | Data and domain state exposed to the view (or serialized to JSON in REST). |
| **View** | Presentation — HTML template, or “view” is skipped when returning `@ResponseBody`. |
| **Controller** | Maps HTTP requests to handlers, calls services, chooses view name or builds response. |

### 2. In Spring MVC
- **Controller** (`@Controller` / `@RestController`) receives the request after `DispatcherServlet` selects a handler.
- **Model** is often a `Map` of attributes, or domain objects returned as JSON in REST.
- **View** is a logical name resolved by `ViewResolver` (Thymeleaf, JSP) — or absent in pure REST.

### 3. Interview nuance
MVC **does not replace** a service layer: controllers should stay thin; business rules live in domain/services.

### 4. Summary (Short speaking answer)

**MVC** is an architectural pattern that separates an application into three components: **Model**, **View**, and **Controller**.

The **Model** represents the application **data** and the **business logic** (or the state you expose to the UI — often backed by services and domain objects).

The **Controller** acts as an **entry point** for HTTP: it handles incoming requests, performs **validation** and binding where appropriate, and **delegates** real processing to the **service layer**. It should **not** be the place for core business rules — keep controllers thin.

The **View** is responsible for **presenting** data to the user, typically as **HTML** pages produced by **template engines** (e.g. Thymeleaf, JSP) in classic Spring MVC.

In **modern REST** applications, the **View** layer is often **not used** at all: controllers return **data directly** (e.g. **JSON**) via `@RestController` / `@ResponseBody`, and presentation moves to a separate client (SPA, mobile app).

---

<a id="q44"></a>

## 44. What is the Dispatcher Servlet?

### Question Restatement
What is **`DispatcherServlet`**, and what does it do in the request lifecycle?

### 1. Definition

`DispatcherServlet` is the **central component** of Spring MVC and acts as the **front controller**.

It receives **all** incoming HTTP requests and is responsible for **coordinating** their processing.

It determines **which controller** should handle the request using **`HandlerMapping`**, **invokes** it via a **`HandlerAdapter`**, and then **processes the result**.

It also handles **request/response conversion**, **exception resolution**, and integrates with other components such as **interceptors** and **message converters** (`HttpMessageConverter` for bodies, `ViewResolver` for HTML views when applicable).

From there, the same pipeline runs for **every** annotated handler — whether you use **`@Controller`** (server-rendered pages) or **`@RestController`** (JSON/XML APIs). **`@RestController`** is only shorthand for **`@Controller` + `@ResponseBody` on the class**.

**What actually differs** between “API” and “MVC page” is not the first 80% of the pipeline — it is mainly **how the return value is turned into an HTTP response**:

| Style | Typical stereotype | How the response is produced |
|--------|-------------------|------------------------------|
| **REST / API** | `@RestController` or `@Controller` + `@ResponseBody` | **`HttpMessageConverter`** → JSON/XML body |
| **Server-side UI** | `@Controller` + view name / `ModelAndView` | **`ViewResolver`** → **`View.render()`** → HTML (Thymeleaf, JSP, …) |

Everything **before** the return-value handling — filters, `DispatcherServlet`, `HandlerMapping`, `HandlerAdapter`, interceptors, argument resolution, validation, controller → service — is **shared**.

Below is **one** production-oriented lifecycle with **both branches** called out where they diverge.

---

### 2. Full HTTP request lifecycle — **shared pipeline + two response paths**

#### 0. Entry: network → container

- **What:** The client sends HTTP — **browser** (HTML forms, page navigation) or **API client** (JSON). The **embedded servlet container** (typically **Tomcat**) accepts the connection and builds `HttpServletRequest` / `HttpServletResponse`.
- **Why:** Spring Boot runs **inside** the container; this section is **Servlet MVC** (not WebFlux).
- **Risks:** **Timeouts**; **worker thread pool exhaustion**; confusing container limits with “Spring slowness”.

---

#### 1. Filter chain

- **What:** **Servlet `Filter`s** — Spring Security (session/JWT, authorization), **CORS**, logging, tracing.
- **Why:** Run **before** `DispatcherServlet`; can reject or mutate the request globally.
- **Risks:** **Wrong order**; **duplicate** security logic in filter **and** interceptor; early response commit.

---

#### 2. `DispatcherServlet`

- **What:** Receives the request and runs **`doDispatch`** — single orchestration entry for **all** controller types.
- **Why:** Same **front controller** for Thymeleaf pages and REST JSON.
- **Risks:** Multipart, custom mappings, misconfigured servlet registration.

---

#### 3. `HandlerMapping`

- **What:** Resolves **path + method + constraints** → **`HandlerMethod`** on **`@Controller` or `@RestController`** + **`HandlerExecutionChain`** (interceptors).
- **Why:** Maps HTTP surface to a Java method — **identical** for UI and API controllers.
- **Risks:** Ambiguous routes, **405** method mismatch.

---

#### 4. `HandlerAdapter`

- **What:** **`RequestMappingHandlerAdapter`** (typical) **invokes** the `HandlerMethod`.
- **Why:** Same adapter family for both styles.
- **Risks:** Custom handler types → wrong adapter.

---

#### 5. `HandlerInterceptor` — `preHandle`

- **What:** Logging, metrics, tracing, correlation IDs **before** the controller runs.
- **Why:** Interceptor sees the **resolved** `HandlerMethod` (unlike filters).
- **Risks:** Business logic here; `preHandle` returns **false** → chain aborted.

---

#### 6. Argument resolution — prepare parameters

- **What:** **`HandlerMethodArgumentResolver`** builds method arguments. **Depends on the endpoint**, not on `@Controller` vs `@RestController`:

**Shared (typical for both)**

- **`@PathVariable`**, **`@RequestParam`**, **`@RequestHeader`**, **`@CookieValue`**, **`Model` / `ModelMap`**, **`Principal`**, **`Pageable`**, etc.

**Common on APIs (`@RequestBody`)**

- **`HttpMessageConverter`** (often **Jackson** for JSON) → DTO.
- **Risks:** JSON/schema drift, nulls, versioning.

**Common on MVC forms / pages**

- **`@ModelAttribute`** binding from form data, **`MultipartFile`** for upload views.
- **Risks:** binding errors, field name mismatches, mass assignment if you bind too much.

**Why:** Controllers always work with **resolved Java values**; only the **sources** differ (JSON body vs form fields vs path).

---

#### 7. Validation

- **What:** **`@Valid` / `@Validated`** (Bean Validation) on DTOs / command objects — **both** API and MVC POST handlers.
- **Why:** Fail fast before services.
- **Risks:** Forgotten annotations; inconsistent **error UI** (HTML) vs **error JSON** (API) unless you centralize handling.

---

#### 8. Controller invocation — entry to *your* system

- **What:** The handler method runs. Same layering for UI and API:

```
Controller → Service → Repository (→ DB / external systems)
```

- **Why:** Web flavor does not change where business logic belongs.
- **Risks:** Fat controllers; leaking persistence into the web layer.

---

#### 9. Business logic (service layer)

- **What:** Transactions, domain rules, integrations — **same** concerns for a “page submit” or a “JSON command”.
- **Risks:** Races, idempotency, long transactions.

---

#### 10. Return value — **fork point**

- **What:** The method returns — **depends on style**:
  - **API:** DTO, `ResponseEntity`, primitives with **`@ResponseBody`** semantics.
  - **MVC:** **String** view name, **`ModelAndView`**, or **`Model` + implicit view** from `@RequestMapping`.
- **Why:** This return object is what **`HandlerMethodReturnValueHandler`** uses to decide the next step.

---

#### 11. Turn return value into HTTP — **two branches**

**Branch A — `@ResponseBody` semantics (including `@RestController`)**

- **What:** **`HttpMessageConverter`** serializes the object to **JSON/XML** (or other negotiated type). Status/headers from `ResponseEntity` if used.
- **Why:** Return value = **response body**, not a view name.
- **Risks:** Lazy proxies, **`produces`/`Accept`** mismatch, wrong converter order.

**Branch B — view resolution (`@Controller` without `@ResponseBody` on that return path)**

- **What:** Return value is interpreted as a **logical view name** (or wrapped in **`ModelAndView`**). **`ViewResolver`** (e.g. Thymeleaf) resolves **`View`** → **`render(model)`** writes **HTML** to the response.
- **Why:** Server-rendered UI — **no** JSON body from the return type itself (unless you mix `@ResponseBody` on one method).
- **Risks:** Wrong view name → **404**-like view errors; **duplicate model keys**; **Open Session In View** hiding lazy-load issues in dev.

**Same machinery:** both branches use **`HandlerMethodReturnValueHandler`** implementations — Spring picks handlers based on return type + annotations.

---

#### 12. `HandlerInterceptor` — `postHandle` / `afterCompletion`

- **What:** **`postHandle`** can still touch **`Model`** before view render (**more relevant for Branch B**). **`afterCompletion`** for logging/cleanup — **both** branches.
- **Risks:** Ordering when exceptions occur.

---

#### 13. Exception handling

- **What:** **`HandlerExceptionResolver`** / **`@ControllerAdvice`** (or **`@RestControllerAdvice`** for JSON-heavy apps).
- **Why:** Centralized errors — **JSON problem** responses for APIs; **error view** or redirect for MVC if you configure it that way.
- **Risks:** Different UX for HTML vs JSON if you do not split advice classes or use `produces` conditions.

---

#### 14. Response to the client

- **What:** HTTP status, headers, and either **JSON/XML body** (Branch A) or **HTML page** (Branch B).
- **Why:** Same wire protocol — different content type.

---

### 3. Production differentiators (not “textbook only”)

| Topic | What to do | Why it matters |
|--------|------------|----------------|
| **Idempotency** | Idempotency keys for **POST**, deduplication, unique constraints | Retries / duplicates → **double writes** without it |
| **Timeouts & cancellation** | Client read timeouts, server timeouts, reactive/async awareness | Client may **disconnect** while server still commits work |
| **Concurrency** | Optimistic locking (`@Version`), transactional boundaries | **Lost updates** / races under parallel requests |
| **Observability** | **Tracing**, **correlationId**, metrics, structured logs | Without them, **prod is undebuggable** |
| **API contracts** | Versioning, backward-compatible DTO changes | Prevents **breaking** mobile/web clients |

---

### 4. End-to-end mental model (keep in mind)

**Same path for `@Controller` and `@RestController` until the return value is processed.** Then the pipeline **forks**:

```
Client
  → Tomcat (threads / connector)
  → Servlet Filters (Security, CORS, logging, …)
  → DispatcherServlet
  → HandlerMapping
  → HandlerAdapter
  → HandlerInterceptor.preHandle
  → Argument resolvers
       ├─ (API) HttpMessageConverter: JSON → DTO (@RequestBody)
       ├─ (MVC) form / @ModelAttribute / MultipartFile …
       └─ (both) @PathVariable, @RequestParam, …
  → Validation (@Valid)  [when used]
  → Controller → Service → Repository → DB / external systems
  ← return value
       ├─ Branch A (@ResponseBody / @RestController)
       │     → HttpMessageConverter → JSON/XML body
       └─ Branch B (@Controller + view)
             → ViewResolver → View.render(model) → HTML
  → HandlerInterceptor.postHandle / afterCompletion
  → (on failure) HandlerExceptionResolver / @ControllerAdvice / @RestControllerAdvice
  → HTTP response
```

---

### 5. Filters vs `HandlerInterceptor` (reminder)

- **Filters:** servlet-level, **before** `DispatcherServlet`, see **every** request hitting the servlet mapping.
- **Interceptors:** Spring MVC, **after** handler is known, can inspect **`HandlerMethod`**.

---

### 6. Summary

The HTTP request lifecycle in Spring MVC consists of a **shared processing pipeline** and then **splits into two response paths**: traditional MVC with **HTML** and REST with **JSON**.

First, the client sends an HTTP request, which is received by the **embedded servlet container**, typically **Apache Tomcat**. The container creates **`HttpServletRequest`** and **`HttpServletResponse`** objects and passes the request into the application.

The request then goes through the **filter chain**, where components like **Spring Security** handle authentication, authorization, **CORS**, and logging. Filters can **stop or modify** the request before it reaches Spring MVC.

After that, the request reaches the **`DispatcherServlet`**, which acts as the **front controller** and **orchestrates** the entire flow.

The `DispatcherServlet` uses **`HandlerMapping`** to find the appropriate **controller method** and then delegates execution to a **`HandlerAdapter`**. Before the controller is invoked, **`HandlerInterceptor`** implementations can run **`preHandle`** logic such as logging or tracing.

Next, Spring prepares method arguments using **`HandlerMethodArgumentResolver`** implementations. It extracts data from the request, such as **path variables**, **query parameters**, **headers**, or **request body**. If the request contains a body, it is converted into a Java object using **`HttpMessageConverter`** implementations, typically with **Jackson**. Validation using **`@Valid`** or **`@Validated`** may also happen at this stage.

Then the **controller method** is executed. The controller acts as an **entry point**, delegates processing to the **service layer**, and should **not** contain business logic. The service layer handles **transactions**, **domain logic**, and interactions with **repositories** or external systems.

After the controller returns a result, Spring processes it using **`HandlerMethodReturnValueHandler`** implementations, and at this point the flow **splits into two branches**.

In **REST** applications, where **`@ResponseBody`** or **`@RestController`** is used, the return value is **serialized** into JSON or another format using **`HttpMessageConverter`** implementations and written **directly** to the HTTP response.

In **traditional MVC** applications, the return value is interpreted as a **view name** or **`ModelAndView`**. A **`ViewResolver`** resolves it to a concrete **view**, such as a **Thymeleaf** template, which is then **rendered** into HTML using the **model** data.

After that, **`postHandle`** and **`afterCompletion`** methods of interceptors are executed for logging, metrics, or cleanup.

If any **exception** occurs during processing, it is handled by **`HandlerExceptionResolver`** implementations, typically configured via **`@ControllerAdvice`** or **`@RestControllerAdvice`**, allowing **centralized** error handling.

Finally, the **HTTP response** with status, headers, and body—either **JSON** or **HTML**—is sent back to the client.

---

<a id="q45"></a>

## 45. What is ModelAndView?

### Question Restatement
What is **`ModelAndView`**, and when is it used?

### 1. Definition

**`ModelAndView`** is a **container object** in Spring MVC that combines **model data** and a **view name**. It is used to **pass data from the controller to the view layer**, typically for **rendering HTML**. In **modern REST** applications, it is **rarely used**, since controllers return **JSON** instead of views.

Under the hood it still pairs:
- **Model** — attributes exposed to the template (`addObject` / constructor).
- **View** — logical name (`String`) or a concrete **`View`** instance.

### 2. Usage
Classic server-rendered MVC: controller returns **view name + model** in one object.

```java
return new ModelAndView("orders/list").addObject("orders", orders);
```

REST controllers typically return **DTOs** or **`ResponseEntity`** — not `ModelAndView`.

### 3. Summary (Short speaking answer)
`ModelAndView` = **model + view name** for **server-side HTML**; for **REST**, prefer **DTO / `ResponseEntity`** and `@RestController`.

---

<a id="q46"></a>

## 46. How is request routing done in Spring MVC?

### Question Restatement
How does Spring decide **which controller method** handles a request?

### 1. Definition
Routing is **declarative**: `@RequestMapping` (class/method), `@GetMapping`, `@PostMapping`, etc. At startup, Spring registers **`RequestMappingHandlerMapping`** entries (path patterns, HTTP methods, params, headers, content types).

### 2. Runtime
1. `DispatcherServlet` asks **HandlerMapping** for a **HandlerExecutionChain**.
2. Best match wins (most specific pattern, HTTP method match).
3. No handler → **404**; wrong method → **405** (if others exist for path).

### 3. Summary

In Spring MVC, request routing is handled by the **`HandlerMapping`** mechanism.

During **application startup**, Spring scans all controller beans annotated with **`@Controller`** or **`@RestController`** and analyzes their mapping annotations such as **`@RequestMapping`**, **`@GetMapping`**, or **`@PostMapping`**. It builds an **internal registry** that maps HTTP methods, URL patterns, and additional constraints like **headers** or **content types** to specific **handler methods**.

When an HTTP request arrives, the **`DispatcherServlet`** delegates to a **`HandlerMapping`** implementation, typically **`RequestMappingHandlerMapping`**, which **matches** the request against this registry and selects the **most appropriate** handler method.

The result is returned as a **`HandlerExecutionChain`**, which includes both the **handler method** and any applicable **interceptors**. The `DispatcherServlet` then passes it to a **`HandlerAdapter`** to invoke the controller method.

This approach allows **efficient** request routing because all mappings are **precomputed at startup** rather than resolved dynamically for each request.

---

<a id="q47"></a>

## 47. What is the difference between Controller and RestController annotations?

### Question Restatement
When do you use **`@Controller`** vs **`@RestController`**?

### 1. Definition
- **`@Controller`** — stereotype for **MVC**; return values are **view names** unless you add `@ResponseBody` on class or method.
- **`@RestController`** = `@Controller` + `@ResponseBody` — return values are serialized to the **HTTP body** (JSON/XML) via `HttpMessageConverter`.

### 2. Rule of thumb
- **Browser HTML** → often `@Controller` + templates.
- **REST API** → `@RestController`.

### 3. Summary (Short speaking answer)
`@RestController` always treats return values as **response body**; `@Controller` is for **views** unless `@ResponseBody` is used.

---

<a id="q48"></a>

## 48. What is the difference between RequestParam and PathVariable annotations?

### Question Restatement
What is **`@RequestParam`** vs **`@PathVariable`**?

### 1. Definition
| Annotation | Source | Example |
|------------|--------|---------|
| **`@PathVariable`** | URI **path segment** | `/orders/{id}` → `id = 42` |
| **`@RequestParam`** | **Query string** or form field | `/orders?status=NEW` |

### 2. Convention
- **Resource identity** → path (`/users/5`).
- **Filters, sort, page** → query params (`?page=0&size=20`).

### 3. Summary (Short speaking answer)
The difference between @RequestParam and @PathVariable is where the data is extracted from in the HTTP request.

@PathVariable is used to extract values from the URI path itself and is typically used to identify a specific resource, for example /users/{id}.

@RequestParam is used to extract values from query parameters, such as /users?id=1, and is commonly used for filtering, pagination, or optional parameters.

In general, @PathVariable represents required and structural parts of the URL, while @RequestParam is used for optional or additional data.

---

<a id="q49"></a>

## 49. How to handle exceptions in Spring MVC?

### Question Restatement
How do you map exceptions to HTTP responses in Spring?

### 1. Options
1. **`@ExceptionHandler`** on controller — local scope.
2. **`@ControllerAdvice` / `@RestControllerAdvice`** — global handlers for many controllers.
3. **`ResponseStatusException`** — quick mapping with status code.
4. **`@ResponseStatus`** on custom exception types.
5. Extend **`ResponseEntityExceptionHandler`** for Spring MVC/security exception types (optional base class).

### 2. REST best practice
Return a **consistent error DTO**: `timestamp`, `path`, `status`, `code`, `message`, `traceId` (no stack traces to clients in prod).

### 3. Summary (Short speaking answer)
In Spring MVC, exceptions are handled using a combination of HandlerExceptionResolver and global exception handling mechanisms.

The most common approach is to use @ControllerAdvice or @RestControllerAdvice to define centralized exception handling logic.
These classes can define methods annotated with @ExceptionHandler, which catch specific exceptions and return a structured HTTP response.

This allows separating error handling from business logic and ensures consistent error responses across the application.

---

<a id="q50"></a>

## 50. What is a ViewResolver?

### Question Restatement
What does **`ViewResolver`** do?

### 1. Definition
`ViewResolver` turns a **logical view name** (e.g. `"orders/list"`) into a concrete **`View`** (Thymeleaf template path, JSP, etc.).

### 2. Example
Controller returns `"orders/list"` → `ThymeleafViewResolver` may resolve to `templates/orders/list.html`.

### 3. REST note
If you only emit JSON, **`ViewResolver` is unused** — `HttpMessageConverter` writes the body instead.

### 4. Summary (Short speaking answer)
A ViewResolver in Spring MVC is a component that maps a logical view name returned by a controller to an actual view implementation, such as an HTML template.

After a controller returns a view name, the DispatcherServlet delegates to a ViewResolver, which resolves that name to a concrete view, for example a Thymeleaf template, and then renders it using the model data.

ViewResolvers are mainly used in traditional MVC applications, while in REST APIs they are typically not used because responses are returned directly as JSON.

---

<a id="q51"></a>

## 51. How to upload files using Spring MVC?

### Question Restatement
How do you accept **file uploads** in Spring?

### 1. Mechanism
- Client sends **`multipart/form-data`**.
- Spring exposes **`MultipartFile`** (or `Resource`, `Part` in Servlet 3+).

```java
@PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) {
    // validate, store
    return ResponseEntity.ok("ok");
}
```

### 2. Configuration
Spring Boot: **`spring.servlet.multipart.*`** (max file size, threshold). For large files, consider **streaming** and object storage (S3, MinIO).

### 3. Security
- Validate **size** and **content type**; do not trust client MIME type alone.
- **Sanitize filenames** — prevent path traversal (`../`).
- Store outside webroot or in object storage; scan if untrusted.

### 4. Summary (Short speaking answer)
In Spring MVC, file uploads are handled using multipart requests with content type multipart/form-data.
The uploaded file is typically received in a controller using the MultipartFile interface.

Spring uses a MultipartResolver to parse the incoming request and extract files before the controller is invoked.

Inside the controller, the file can be accessed, validated, and stored on disk, in a database, or in external storage like cloud services.

---

<a id="q52"></a>

## 52. What is Thymeleaf?

### Question Restatement
What is **Thymeleaf**, and where is it used?

### 1. Definition
**Thymeleaf** is a **server-side** template engine for HTML, integrated with Spring MVC (`th:each`, `th:text`, `th:if`, form binding, Spring Security extras).

### 2. Features
- **Natural templates** — HTML can be opened in a browser as static prototype.
- Layout dialects for shared layouts.

### 3. When not used
Pure **SPA + REST** backends usually **do not** need Thymeleaf — JSON only.

### 4. Summary (Short speaking answer)
Thymeleaf is a server-side Java template engine used in Spring MVC to generate dynamic HTML pages.
It processes HTML templates by injecting data from the model into the view and rendering the final HTML on the server before sending it to the client.

It integrates well with Spring because it supports natural templating, meaning the templates are valid HTML files that can be opened directly in a browser without rendering.

---

<a id="q53"></a>

## 53. What is the difference between Authorization and Authentication?

### Question Restatement
Distinguish **authentication** from **authorization**.

### 1. Definitions
- **Authentication (AuthN)** — *Who are you?* Proving identity (password, SSO, MFA, client certificate).
- **Authorization (AuthZ)** — *What may you do?* Permissions, roles, scopes, policies.

### 2. Order
Always **authenticate first**, then **authorize**. Valid token but missing role → often **403 Forbidden**; no/invalid identity → **401 Unauthorized**.

### 3. Summary (Short speaking answer)
The difference between authentication and authorization is that authentication is about verifying who the user is, while authorization is about determining what the user is allowed to do.

Authentication happens first and typically involves validating credentials such as a username and password, or a token like a JWT, to confirm the user’s identity.
Authorization happens after that and defines the user’s permissions, for example whether they can access a specific endpoint or perform certain actions based on roles or privileges.

In Spring applications, authentication is usually handled by Spring Security through mechanisms like login or token validation, while authorization is enforced using roles, authorities, and annotations such as @PreAuthorize or configuration rules.

The key idea is that authentication answers “who are you?”, and authorization answers “what are you allowed to do?”, and both are required to properly secure an application.

---

<a id="q54"></a>

## 54. What is the difference between hashing and crypting?

### Question Restatement
Compare **hashing** and **encryption** (interview wording often says “crypting”).

### 1. Hashing
- **One-way** — not meant to be reversed.
- Used for **password verification** and integrity (HMAC).
- Use **salt/pepper** and slow algorithms: **bcrypt**, **Argon2**, **scrypt**.

### 2. Encryption
- **Reversible** with a **key** (symmetric AES, asymmetric RSA/ECC).
- For data that must be **read back** (secrets at rest, field-level encryption).

### 3. Rule
- **Passwords** → hash, never “decrypt”.
- **Confidential payloads** to store and later decrypt → encrypt.

### 4. Summary (Short speaking answer)

The difference between hashing and encryption (often mistakenly called “crypting”) is that hashing is a one-way operation, while encryption is reversible.
Hashing takes input data and produces a fixed-length hash value that cannot be converted back to the original data. It is typically used for storing sensitive data like passwords. In production, hashing is done with algorithms like bcrypt or Argon2, often with salting to prevent rainbow table attacks.
Encryption, on the other hand, transforms data into an unreadable format using a key, but it can be decrypted back to the original form using the corresponding key. It is used when data needs to be stored or transmitted securely but later restored, for example in HTTPS or encrypted databases.
The key idea is that hashing is used for verification (you compare hashes), while encryption is used for confidentiality (you can recover the original data), and using encryption instead of hashing for passwords is a critical security mistake.

---

<a id="q55"></a>

## 55. What is the structure of JWT token?

### Question Restatement
What are the parts of a **JWT**?

### 1. Structure
Three **Base64URL** segments separated by `.`:

1. **Header** — `typ: JWT`, `alg` (e.g. `HS256`, `RS256`).
2. **Payload** — **claims**: `sub`, `iss`, `aud`, `exp`, `iat`, custom claims.
3. **Signature** — `sign( base64url(header) + "." + base64url(payload), key )`.

### 2. Security notes
- Default JWT is **signed**, **not encrypted** — payload is **readable** (just base64); **no secrets** in payload.
- Validate **signature**, **exp**, **iss**, **aud**, and use **strong algorithms**; beware **alg:none** attacks and key confusion.

### 3. Summary (Short speaking answer)
A JSON Web Token (JWT) consists of three parts separated by dots: header, payload, and signature.

The header contains metadata about the token, such as the signing algorithm (for example HS256 or RS256) and the token type.
The payload contains claims, which are pieces of information about the user or the token, such as user ID, roles, and standard fields like exp (expiration time) or sub (subject).
The signature is used to verify the integrity of the token. It is created by signing the encoded header and payload using a secret key or a private key, depending on the algorithm.

The final token looks like this:
header.payload.signature, where each part is Base64URL-encoded.

The key idea is that JWT is not encrypted by default, it is only signed, so the payload can be decoded by anyone, but cannot be modified without invalidating the signature.

---

<a id="q56"></a>

## 56. How does OAuth 2.0 work? What are the roles in OAuth 2.0?

### Question Restatement
Describe **OAuth 2.0** and its **roles**.

### 1. What it is
Framework for **delegated authorization** — user grants a **client** limited access to resources **without** sharing the password with the client (ideally).

### 2. Roles
| Role | Description |
|------|-------------|
| **Resource Owner** | User who owns data. |
| **Client** | App requesting access (web, mobile, SPA). |
| **Authorization Server** | Issues tokens after user consent/login. |
| **Resource Server** | API that accepts **access tokens** and serves protected resources. |

### 3. Common flow (Authorization Code + **PKCE** for public clients)
Redirect → login/consent → **authorization code** → token endpoint → **access token** → call API with `Authorization: Bearer`.

### 4. Summary (Short speaking answer)

OAuth 2.0 is an authorization framework that allows a client application to access protected resources on behalf of a user without exposing the user’s credentials.
It works by delegating authentication to an authorization server, which issues an access token that the client uses to call a resource server (API).
There are four main roles in OAuth 2.0:


Resource Owner — the user who owns the data


Client — the application requesting access


Authorization Server — the system that authenticates the user and issues tokens


Resource Server — the API that hosts protected resources and validates access tokens


The typical flow is that the client redirects the user to the authorization server, where the user authenticates and grants permissions. The authorization server then issues an access token, which the client includes in requests to the resource server to access protected data.
The key idea is that the client never sees the user’s credentials, only the token, and access is granted based on scopes and permissions defined during authorization.

---

<a id="q57"></a>

## 57. How does OpenId work?

### Question Restatement
What is **OpenID Connect** (OIDC) relative to OAuth 2.0?

### 1. Definition
**OpenID Connect** builds on OAuth 2.0 and adds an **identity layer**:
- **`id_token`** (JWT) with identity claims,
- **`openid` scope**, standard claims (`profile`, `email`),
- **UserInfo** endpoint.

### 2. Flow (simplified)
1. Client starts OAuth 2 authorization code flow with **`openid`** scope.
2. **Authorization Server** (OpenID Provider) returns **access token** + **id token**.
3. Client validates **id token** (issuer, audience, signature, `nonce`, expiry).
4. App establishes **logged-in user** session.

### 3. Summary (Short speaking answer)
**OAuth2** = delegated access; **OIDC** = **login + identity** via `id_token` and standard claims.

---

<a id="q58"></a>

## 58. What is CORS?

### Question Restatement
What is **CORS**, and why does it exist?

### 1. Definition
**Cross-Origin Resource Sharing** — browser mechanism that **relaxes** the Same-Origin Policy for **cross-origin** XHR/fetch from **JavaScript**.

### 2. How it works
- Browser sends **`Origin`** header.
- Server responds with **`Access-Control-Allow-Origin`** (and often `Methods`, `Headers`, `Credentials`).
- **Preflight**: non-simple requests trigger **`OPTIONS`** first.

### 3. Security
- Avoid `*` with **credentials**; whitelist **trusted origins**.
- CORS is **browser-only** — server-side `curl` is not restricted by CORS.

### 4. Summary (Short speaking answer)
CORS (Cross-Origin Resource Sharing) is a browser security mechanism that controls whether a web application can make requests to a different origin, where origin means a combination of scheme, domain, and port.

By default, browsers block cross-origin requests for security reasons, and CORS allows the server to explicitly specify which origins, methods, and headers are permitted using HTTP response headers.

When a cross-origin request is made, the browser may first send a preflight request (an HTTP OPTIONS request) to check if the actual request is allowed. If the server responds with the appropriate CORS headers, the browser proceeds with the request; otherwise, it blocks it.

In Spring applications, CORS can be configured globally or per controller, and it is often handled together with Spring Security to ensure proper access control.

The key idea is that CORS is enforced by the browser, not the server, and it protects users from unauthorized cross-origin interactions, not the API itself.

---

<a id="q59"></a>

## 59. How to handle Security exceptions?

### Question Restatement
How does **Spring Security** map failures to HTTP responses?

### 1. Spring Security hooks
- **Not authenticated** / auth failed → **`AuthenticationEntryPoint`** → typically **401**.
- **Authenticated but forbidden** → **`AccessDeniedHandler`** → typically **403**.

### 2. REST APIs
Configure **JSON error body** (consistent with `@ControllerAdvice` if used together); avoid leaking stack traces or internal **user enumeration** details.

### 3. Customization
`SecurityFilterChain` bean: `exceptionHandling().authenticationEntryPoint(...).accessDeniedHandler(...)`.

### 4. Summary (Short speaking answer)
In Spring applications, security exceptions are handled at the security layer, typically by Spring Security, not by standard controller exception handling.

There are two main types of security exceptions: authentication and authorization.
Authentication failures, when the user is not authenticated or provides invalid credentials, are handled by an AuthenticationEntryPoint, which usually returns a 401 Unauthorized response.
Authorization failures, when the user is authenticated but does not have sufficient permissions, are handled by an AccessDeniedHandler, which returns a 403 Forbidden response.

These handlers are configured in the security configuration and allow customizing the HTTP response, for example returning a structured JSON error instead of a default HTML error page.

The key idea is that security exceptions are intercepted before the request reaches the controller, so they must be handled within the security filter chain rather than using @ControllerAdvice.

---

<a id="q60"></a>

## 60. What is SQL Injection?

### Question Restatement
What is **SQL injection**, and how do you prevent it?

### 1. Definition
Attacker supplies input that **changes SQL structure** (e.g. `' OR '1'='1`) to bypass filters, exfiltrate, or modify data.

### 2. Prevention
- **Parameterized queries** / **bind parameters** (`PreparedStatement`, JPA `setParameter`).
- **Never** concatenate untrusted input into SQL strings.
- ORM helps but **native queries** with string concat are still vulnerable.
- **Least privilege** DB users; **input validation** as defense in depth.

### 3. Summary (Short speaking answer)
SQL Injection is a type of security vulnerability where an attacker injects malicious SQL code into an application’s query, allowing them to manipulate or access the database in unintended ways.

It typically happens when user input is directly concatenated into SQL queries without proper validation or parameterization. For example, instead of treating input as data, the database interprets it as part of the query logic.

This can allow attackers to bypass authentication, read sensitive data, modify or delete records, or even execute administrative operations on the database.

The primary way to prevent SQL Injection is to use parameterized queries or ORM frameworks that separate query structure from user input, ensuring that input is always treated as data and not executable SQL.

The key idea is that SQL Injection exploits the mixing of code and data, and proper query parameterization eliminates that risk.

---

<a id="q61"></a>

## 61. What is an XSS attack?

### Question Restatement
What is **Cross-Site Scripting (XSS)**?

### 1. Types
- **Stored** — payload saved (DB) and served to victims.
- **Reflected** — payload in URL/response once.
- **DOM-based** — unsafe client-side DOM APIs.

### 2. Impact
Steal **session cookies** / tokens, perform actions as user, deface UI.

### 3. Prevention
- **Context-appropriate output encoding** (HTML, JS, URL, CSS).
- **Content-Security-Policy** (CSP).
- Avoid `innerHTML` with untrusted data; sanitize rich text carefully.

### 4. Summary (Short speaking answer)
An XSS (Cross-Site Scripting) attack is a security vulnerability where an attacker injects malicious JavaScript into a web application, which is then executed in the browser of other users.

It typically occurs when user input is included in a page without proper escaping or sanitization, allowing the attacker’s script to run in the context of the application.

This can lead to stealing session cookies, hijacking user accounts, performing actions on behalf of the user, or redirecting users to malicious sites.

There are different types of XSS, such as stored XSS (persisted in the database), reflected XSS (returned immediately in the response), and DOM-based XSS (executed in the browser via client-side scripts).

The primary defense is proper output encoding and escaping of user input, along with using security measures like Content Security Policy (CSP) and avoiding unsafe rendering in the frontend.

The key idea is that XSS exploits the execution of untrusted input as code in the user’s browser.

---

<a id="q62"></a>

## 62. What is a CSRF attack?

### Question Restatement
What is **CSRF**?

### 1. Definition
**Cross-Site Request Forgery** — victim’s browser sends a **forged** request to a site where the user is **authenticated**, using **automatic cookie** sending.

### 2. Example
User logged into bank; malicious page submits hidden **POST** transfer; browser attaches **session cookie**.

### 3. Mitigations
- **Synchronizer token** (Spring Security CSRF token for state-changing requests).
- **`SameSite` cookies** (Lax/Strict where appropriate).
- **Double Submit Cookie** or custom headers for SPAs.
- Stateless **Bearer** APIs avoid cookie CSRF but need other protections (XSS, token storage).

### 4. Summary (Short speaking answer)
A CSRF (Cross-Site Request Forgery) attack is a security vulnerability where a malicious site tricks a user’s browser into making an unwanted authenticated request to another application where the user is already logged in.

Because browsers automatically include credentials like cookies with requests, the target application cannot distinguish between a legitimate request and a forged one.

For example, if a user is logged into a banking site and visits a malicious page, that page can trigger a request like a money transfer, and the browser will send the user’s session cookie along with it.

The main protection against CSRF is the use of CSRF tokens — unique, unpredictable values that are included in requests and validated by the server. Since an attacker cannot access these tokens, forged requests will be rejected.

Additional protections include using SameSite cookies and requiring explicit authorization for sensitive actions.

The key idea is that CSRF exploits the trust the server has in the user’s browser, not the user’s credentials themselves.

---

<a id="q63"></a>

## 63. What is the difference between HTTP and HTTPS?

### Question Restatement
Compare **HTTP** and **HTTPS**.

### 1. HTTP
Plaintext on the wire — **no** confidentiality, integrity, or server authentication by default.

### 2. HTTPS
**HTTP over TLS** — encrypts traffic, provides integrity; server proves identity via **X.509 certificate** (and optionally mutual TLS for clients).

### 3. Practice
Redirect HTTP→HTTPS, **HSTS**, modern TLS versions; certificates via Let’s Encrypt or corporate PKI.

### 4. Summary (Short speaking answer)
The difference between HTTP and HTTPS is that HTTP transmits data in plain text, while HTTPS encrypts communication using TLS (Transport Layer Security).

In HTTP, all data including headers, cookies, and request/response bodies can be intercepted and read or modified by an attacker. HTTPS, on the other hand, establishes a secure encrypted channel between the client and the server, ensuring confidentiality, integrity, and authenticity of the data.

HTTPS uses certificates issued by trusted authorities to verify the server’s identity and prevent man-in-the-middle attacks.

The key idea is that HTTP is insecure and should not be used for sensitive data, while HTTPS protects data in transit through encryption and identity verification.

---

<a id="q64"></a>

## 64. What is a Brute Force attack?

### Question Restatement
What is a **brute-force** attack, and how do you mitigate it?

### 1. Definition
Repeated **guessing** of passwords, OTPs, or tokens until success — often automated.

### 2. Related: **credential stuffing**
Reusing **leaked** username/password pairs — often more effective than random guessing.

### 3. Mitigations
- **Rate limiting** / CAPTCHA / progressive backoff.
- **MFA**, account lockout policies (careful with DoS).
- **Monitoring**, IP reputation, WAF.
- Strong **password hashing** storage (Argon2/bcrypt).

### 4. Summary (Short speaking answer)
A Brute Force attack is a type of attack where an attacker attempts to guess a user’s credentials, such as passwords or tokens, by systematically trying many possible combinations until the correct one is found.

This is usually done using automated tools that can send a large number of requests in a short time.

Brute force attacks can be mitigated by implementing rate limiting, account lockouts after multiple failed attempts, CAPTCHA, multi-factor authentication, and using strong password hashing algorithms like bcrypt.

The key idea is that brute force attacks rely on repetition and weak protections, so limiting attempts and strengthening authentication mechanisms makes them ineffective.

