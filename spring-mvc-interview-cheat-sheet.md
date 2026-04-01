# Spring MVC Interview Cheat Sheet (Middle/Senior)

Deep dive into Spring MVC internals, execution flow, and production pitfalls.

---

## 1) Architecture: How Spring MVC Works Under the Hood

### Complete HTTP Request Lifecycle

```
HTTP Request
    ↓
Servlet Container (Tomcat/Jetty)
    ↓
Filter Chain (CORS, Auth, Logging)
    ↓
DispatcherServlet (Front Controller)
    ↓
HandlerMapping → Finds @Controller method
    ↓
HandlerInterceptor.preHandle()
    ↓
HandlerAdapter → Invokes controller method
    ↓
Controller → @RequestMapping method
    ↓
Service Layer
    ↓
Repository/Database
    ↓
Controller returns ModelAndView or @ResponseBody
    ↓
HandlerInterceptor.postHandle()
    ↓
ViewResolver (for MVC) OR HttpMessageConverter (for REST)
    ↓
HandlerInterceptor.afterCompletion()
    ↓
HTTP Response
```

### Key Components

**DispatcherServlet**
- Central front controller — every request goes through it
- Delegates to other components (doesn't handle business logic itself)
- Manages the entire request processing workflow

**HandlerMapping**
- Maps incoming requests to handler methods
- Default: `RequestMappingHandlerMapping` (parses @RequestMapping)
- Returns HandlerExecutionChain (handler + interceptors)

**HandlerAdapter**
- Adapts different handler types (annotated methods, HttpServlet, etc.)
- Default: `RequestMappingHandlerAdapter`
- Handles argument resolution (@RequestBody, @PathVariable, etc.)

**ViewResolver**
- Resolves logical view names to actual views
- Examples: ThymeleafViewResolver, InternalResourceViewResolver
- Not used in REST APIs (bypassed when @ResponseBody present)

**Interview Answer:**
Spring MVC follows Front Controller pattern. DispatcherServlet receives all requests, asks HandlerMapping which controller method should handle it, then HandlerAdapter invokes that method. For traditional MVC, ViewResolver renders the view. For REST, HttpMessageConverter serializes the response directly, bypassing ViewResolver entirely.

---

## 2) Servlet Fundamentals (Critical)

### What is Servlet

Servlet is a Java class that handles HTTP requests. Spring MVC builds on top of Servlet API — DispatcherServlet IS a Servlet (extends HttpServlet).

### HttpServletRequest / HttpServletResponse

**Request:** Contains HTTP method, headers, parameters, body, session
**Response:** Status code, headers, output stream for body

### Filter vs Servlet vs Interceptor

| Component | Where | Scope | Spring Aware? |
|-----------|-------|-------|---------------|
| **Filter** | Servlet Container | All requests | No |
| **Servlet** | Servlet Container | Mapped URLs | Can be (Spring manages) |
| **Interceptor** | Spring MVC | DispatcherServlet only | Yes |

**Filter Chain (Critical):**
```
HTTP Request
    ↓
Filter 1 (e.g., SecurityFilter)
    ↓
Filter 2 (e.g., RequestLoggingFilter)
    ↓
DispatcherServlet
    ↓
... Spring MVC processing ...
    ↓
Response back through Filters in reverse order
```

**Where Spring Starts:**
After Filter chain completes, request reaches DispatcherServlet — this is where Spring MVC processing begins.

**Common Trap:**
Filters execute BEFORE Spring context is fully involved. You can't use @Autowired in Filter unless you register it as Spring bean via FilterRegistrationBean.

---

## 3) DispatcherServlet Deep Dive

### Central Front Controller

```java
public class DispatcherServlet extends FrameworkServlet {
    // Entry point
    protected void doService(HttpServletRequest request, HttpServletResponse response) {
        // 1. Build request attributes snapshot
        // 2. Determine HandlerExecutionChain
        // 3. Determine HandlerAdapter
        // 4. Pre-handle interceptors
        // 5. Execute handler
        // 6. Post-handle interceptors
        // 7. Process result (view or response body)
        // 8. Trigger afterCompletion
    }
}
```

### Execution Flow Step-by-Step

**1. Handler Lookup:**
```java
HandlerExecutionChain mappedHandler = getHandler(processedRequest);
// Returns: handler + list of interceptors
```

**2. HandlerAdapter Selection:**
```java
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
// Different adapters for different handler types
```

**3. Pre-handle Phase:**
```java
if (!mappedHandler.applyPreHandle(processedRequest, response)) {
    return; // Interceptor returned false, stop processing
}
```

**4. Handler Execution:**
```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
// Actually invokes the @RequestMapping method
```

**5. Post-handle Phase:**
```java
mappedHandler.applyPostHandle(processedRequest, response, mv);
// Interceptors can modify ModelAndView before rendering
```

**6. Process Dispatch Result:**
```java
processDispatchResult(processedRequest, response, mappedHandler, mv, exception);
// Renders view or writes response body
```

**7. After Completion (Always Executes):**
```java
mappedHandler.triggerAfterCompletion(request, response, null);
// Cleanup, logging, resource release
```

### Exception Handling Flow

If exception occurs during handler execution:
```java
try {
    mv = ha.handle(processedRequest, response, handler);
} catch (Exception ex) {
    // 1. Call HandlerExceptionResolver chain
    // 2. If resolved, return ModelAndView for error view
    // 3. If not resolved, rethrow
}
```

**Interview Answer:**
DispatcherServlet is the front controller. It finds the right handler via HandlerMapping, gets the adapter, runs preHandle interceptors, executes the handler method, runs postHandle, then either renders a view or writes the response body. If an exception occurs, it delegates to HandlerExceptionResolvers which can map exceptions to error views or handlers.

---

## 4) Handler Mapping & Adapter

### How @RequestMapping is Resolved

**RequestMappingHandlerMapping** processes @Controller classes at startup:
```java
@Controller
@RequestMapping("/orders")
public class OrderController {
    @GetMapping("/{id}")  // → Ant pattern: /orders/{id}
    public Order getOrder(@PathVariable Long id) { ... }
}
```

At startup, Spring builds a mapping registry: `Map<RequestCondition, HandlerMethod>`

### AntPathMatcher vs PathPattern

- **AntPathMatcher** (default pre-Spring 5.3): `?` = one char, `*` = any chars within segment, `**` = any path
- **PathPattern** (Spring 5.3+ default): More efficient, supports `{*path}` catch-all variables

```java
@GetMapping("/files/{*path}")  // PathPattern captures rest of path including slashes
```

### Method Resolution

RequestMappingInfo combines:
- URL pattern
- HTTP method (GET, POST, etc.)
- Consumes (Content-Type header)
- Produces (Accept header)
- Params (request parameters)
- Headers

**Matching priority:** More specific mappings win. `/orders/123` matches concrete path before `/orders/{id}`.

### Argument Resolvers

How @RequestBody works internally:
```java
public Order createOrder(@RequestBody OrderRequest request)
```

**Resolution flow:**
1. `RequestMappingHandlerAdapter` finds `RequestResponseBodyMethodProcessor`
2. Processor reads Content-Type header
3. Selects `HttpMessageConverter` (Jackson for application/json)
4. Jackson deserializes JSON to OrderRequest object
5. Validation triggered if @Valid present

**Deep Insight:**
Argument resolvers are pluggable. You can create custom resolvers for things like `@CurrentUser` to inject authenticated user.

---

## 5) Controller Layer

### @Controller vs @RestController

**@Controller:**
```java
@Controller
public class OrderViewController {
    @GetMapping("/orders/{id}")
    public String getOrder(@PathVariable Long id, Model model) {
        model.addAttribute("order", orderService.findById(id));
        return "orderDetail";  // View name resolved by ViewResolver
    }
}
```

**@RestController:**
```java
@RestController  // = @Controller + @ResponseBody on all methods
public class OrderApiController {
    @GetMapping("/api/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);  // Directly serialized to JSON
    }
}
```

**@ResponseBody:**
- Indicates return value should be serialized directly to response body
- Bypasses ViewResolver
- Uses HttpMessageConverter (Jackson for JSON)

### Request/Response Binding

**Request binding:**
```java
@PostMapping("/orders")
public Order create(
    @RequestBody OrderRequest request,           // JSON → Object
    @RequestHeader("X-Api-Key") String apiKey,   // Header
    @CookieValue("sessionId") String session,    // Cookie
    @RequestParam(required = false) String ref   // Query param
) { ... }
```

**Validation:**
```java
@PostMapping("/orders")
public ResponseEntity<Order> create(
    @Valid @RequestBody OrderRequest request,    // Triggers validation
    BindingResult result                         // Validation errors
) {
    if (result.hasErrors()) {
        // Handle errors
    }
}
```

**Common Trap:**
@Valid without BindingResult throws MethodArgumentNotValidException which you need to handle with @ExceptionHandler.

---

## 6) Request Lifecycle (STEP-BY-STEP)

Detailed execution flow with all stages:

```
1. HTTP Request arrives at Servlet Container (Tomcat)

2. Filter Chain executes (in order):
   a. CharacterEncodingFilter
   b. CORSFilter
   c. SecurityFilter (Spring Security)
   d. Custom Filters...

3. DispatcherServlet.doDispatch():
   
   4. HandlerMapping determines HandlerExecutionChain:
      - HandlerMethod (Controller + method)
      - List<HandlerInterceptor>
   
   5. HandlerInterceptor.preHandle():
      - Authentication checks
      - Logging
      - Can return false to abort request
   
   6. HandlerAdapter.handle():
      - Argument resolution (@RequestBody, @PathVariable)
      - Method invocation via reflection
      - Controller executes business logic
      - Return value processing
   
   7. HandlerInterceptor.postHandle():
      - Modify ModelAndView before rendering
      - Add common model attributes
   
   8. View Resolution (if ModelAndView returned):
      - ViewResolver resolves logical name to View
      - View.render() writes to response
      
      OR (if @ResponseBody):
      
   8b. HttpMessageConverter:
      - Jackson converts object to JSON
      - Writes to response body
   
   9. HandlerInterceptor.afterCompletion():
      - Always executes (even on exception)
      - Cleanup, logging, metrics

10. Response travels back through Filter chain (reverse order)

11. HTTP Response sent to client
```

**Critical sequence for REST API:**
```
Request → Filters → DispatcherServlet → preHandle → Controller 
  → @RequestBody conversion → Service → Repository 
  → Return object → @ResponseBody conversion (Jackson) 
  → postHandle → afterCompletion → Response
```

---

## 7) Interceptors vs Filters (VERY IMPORTANT)

### Key Differences

| Aspect | Filter | Interceptor |
|--------|--------|-------------|
| **Managed by** | Servlet Container | Spring MVC |
| **Access to** | HttpServletRequest/Response | HttpServletRequest/Response + Spring Model |
| **Execution** | Before/after Servlet | Before/after Controller |
| **Can modify** | Request/Response | ModelAndView |
| **Knowledge** | No Spring context | Full Spring context |

### Execution Order

```
Request
    ↓
Filter 1 (Servlet level)
    ↓
Filter 2 (Servlet level)
    ↓
DispatcherServlet
    ↓
Interceptor.preHandle() 1
    ↓
Interceptor.preHandle() 2
    ↓
Controller
    ↓
Interceptor.postHandle() 2 (reverse order)
    ↓
Interceptor.postHandle() 1
    ↓
View Rendering / ResponseBody
    ↓
Interceptor.afterCompletion() 2
    ↓
Interceptor.afterCompletion() 1
    ↓
Filter 2 (reverse)
    ↓
Filter 1 (reverse)
```

### When to Use Each

**Use Filters when:**
- Authentication/Authorization before Spring (e.g., JWT validation)
- Request/response logging at low level
- CORS handling
- GZIP compression
- Needs to wrap request/response (e.g., caching)

**Use Interceptors when:**
- Need access to Spring Model
- Adding common model attributes for views
- Post-processing controller results
- Metrics/auditing with business context
- Redirecting based on controller outcome

**Interview Answer:**
Filters work at Servlet level, before Spring MVC gets involved. They're great for things like authentication, CORS, or request logging that don't need Spring context. Interceptors work inside Spring MVC, have access to ModelAndView, and are better for business-related pre/post processing like adding user info to model or audit logging with business context.

---

## 8) Data Binding & Serialization

### How JSON → Java Object Works

```java
@PostMapping("/orders")
public Order create(@RequestBody OrderRequest request) { ... }
```

**Step-by-step:**
1. Request arrives with `Content-Type: application/json`
2. `RequestMappingHandlerAdapter` selects `RequestResponseBodyMethodProcessor`
3. Processor asks `ContentNegotiationManager` for message converter
4. `MappingJackson2HttpMessageConverter` selected (handles JSON)
5. Jackson `ObjectMapper` reads request body InputStream
6. JSON deserialized to OrderRequest object
7. If @Valid present, Bean Validation runs
8. Method invoked with populated object

### Jackson Role

Jackson is the default JSON library in Spring Boot. It:
- Deserializes request JSON to Java objects
- Serializes Java objects to response JSON
- Handles custom serializers/deserializers
- Manages date formats, property naming

### HttpMessageConverters

Spring uses converters based on Content-Type and Accept headers:

| Converter | Content-Type | Use Case |
|-----------|--------------|----------|
| MappingJackson2HttpMessageConverter | application/json | REST APIs |
| StringHttpMessageConverter | text/plain | String responses |
| ByteArrayHttpMessageConverter | application/octet-stream | File downloads |

**Common Pitfalls:**

**1. Date format issues:**
```java
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime createdAt;
```

**2. Unknown properties:**
```yaml
spring:
  jackson:
    deserialization:
      fail-on-unknown-properties: false  # Default: true
```

**3. Lazy loading in serialization:**
```java
// BAD: Triggers N+1 during JSON serialization
public class Order {
    @OneToMany(mappedBy = "order", fetch = LAZY)
    private List<Item> items;  // Jackson tries to serialize, loads lazily
}
```

**Fix:** Use DTOs or @JsonIgnoreProperties("hibernateLazyInitializer")

---

## 9) Exception Handling

### @ExceptionHandler

Handles exceptions in specific controller:
```java
@Controller
public class OrderController {
    
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }
}
```

### @ControllerAdvice

Global exception handling:
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidationErrors(
            MethodArgumentNotValidException ex) {
        
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage
            ));
        
        return ResponseEntity.badRequest().body(errors);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("Internal error"));
    }
}
```

### Exception Resolver Chain

When exception occurs:
1. HandlerExceptionResolverComposite tries multiple resolvers:
   - ExceptionHandlerExceptionResolver (@ExceptionHandler methods)
   - ResponseStatusExceptionResolver (@ResponseStatus)
   - DefaultHandlerExceptionResolver (Spring standard exceptions)
2. First resolver that can handle it wins
3. If none handle it, exception propagates up

**Deep Insight:**
ControllerAdvice with @Order can define priority when multiple exist.

---

## 10) Spring MVC vs REST APIs (CRITICAL)

### View-Based MVC vs REST

| Aspect | Traditional MVC | REST API |
|--------|----------------|----------|
| **Controller** | @Controller | @RestController |
| **Return type** | String (view name) | Object (entity/DTO) |
| **Rendering** | ViewResolver → View | HttpMessageConverter |
| **Response** | HTML page | JSON/XML |
| **ViewResolver** | Used | Bypassed |

### Why REST Controllers Don't Use ViewResolver

**@RestController = @Controller + @ResponseBody**

When @ResponseBody is present:
```java
@RestController
public class ApiController {
    @GetMapping("/api/orders/{id}")
    @ResponseBody  // ← This triggers different path
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);
    }
}
```

**Processing difference:**
```
Traditional MVC:
Controller returns "orderDetail" → ViewResolver finds orderDetail.html 
  → Thymeleaf renders → Response

REST API:
Controller returns Order object → MappingJackson2HttpMessageConverter 
  → Jackson serializes to JSON → Response
```

**Interview Answer:**
@RestController adds @ResponseBody to all methods, which tells Spring to skip ViewResolver entirely. Instead of looking for a view template, Spring uses HttpMessageConverters to serialize the return object directly to the response body — Jackson for JSON by default.

### When ViewResolver is Used

- Traditional server-rendered web apps
- Spring with Thymeleaf, JSP, FreeMarker
- Error pages
- PDF/Excel generation views

---

## 11) Performance & Pitfalls

### Blocking Nature of Spring MVC

Spring MVC is **synchronous and blocking**:
- One thread per request
- Thread waits during I/O (database, external API calls)
- Thread pool size limits concurrent requests

```
Request 1 → Thread-1 allocated → Processing → DB query (Thread-1 blocked, waiting) 
  → Response → Thread-1 released
```

**Impact:** Under high load with slow I/O, thread pool exhausts and requests queue.

### Thread Per Request Model

Default Tomcat: ~200 threads
If 200 requests are waiting for database, 201st request waits in queue.

**Mitigation:**
- Increase thread pool (but memory cost)
- Use async processing (Servlet 3.0 async)
- Use WebFlux for fully reactive (different programming model)

### WebFlux Difference (Brief)

WebFlux:
- Event-loop (like Node.js)
- Few threads handle many concurrent requests
- Non-blocking I/O throughout
- Functional programming with Mono/Flux

**When to use WebFlux:** High concurrency, streaming, low latency requirements.
**When to stick with MVC:** Most business apps, complex transactions, team expertise.

### Common Mistakes

**1. Heavy logic in controllers:**
```java
// BAD
@GetMapping("/report")
public Report generateReport() {
    // Complex calculation, data aggregation, file generation
    // Blocks thread for seconds
}
```

**Fix:** Move to async processing or dedicated service with caching.

**2. Misuse of @RequestBody with large payloads:**
```java
// BAD: No size limit
@PostMapping("/upload")
public void upload(@RequestBody String data) { }
// Could be 100MB, out of memory
```

**Fix:** Configure `spring.servlet.multipart.max-file-size` and `max-request-size`.

**3. Jackson serialization of lazy associations:**
```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", fetch = LAZY)
    private List<Item> items;
}

// Controller
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderRepository.findById(id).get();  // Jackson triggers N+1
}
```

**Fix:** Use DTOs, @JsonIgnore, or OpenEntityManagerInView (with caution).

**4. Not using @Valid:**
```java
// BAD: No validation
@PostMapping("/orders")
public Order create(@RequestBody OrderRequest request) { ... }
```

**Fix:** Always validate request bodies in production.

---

## 12) Interview Questions with Strong Answers

### Q: Explain how a request flows through Spring MVC

**Strong Answer:**
> Request hits the Servlet Container first, goes through the Filter chain (security, CORS), then reaches DispatcherServlet. DispatcherServlet asks HandlerMapping which controller method should handle it, gets the HandlerAdapter, runs preHandle interceptors, invokes the controller method. The controller returns either a view name (traditional MVC) or an object (REST). For REST, HttpMessageConverter serializes to JSON. postHandle and afterCompletion interceptors run, then the response goes back through the Filter chain.

---

### Q: Difference between Filter and Interceptor?

**Strong Answer:**
> Filters work at Servlet level, before Spring MVC is involved. They don't have access to Spring context or Model. Use for authentication, CORS, low-level logging. Interceptors work inside Spring MVC, have access to ModelAndView, and can modify the model before view rendering. Use for business-level pre/post processing like adding user info to model or audit logging.

---

### Q: What does DispatcherServlet actually do?

**Strong Answer:**
> It's the front controller. Receives all requests, delegates to HandlerMapping to find the right handler, gets HandlerAdapter to invoke it, manages the interceptor chain (preHandle, postHandle, afterCompletion), and orchestrates the response — either through ViewResolver for MVC or HttpMessageConverter for REST. It doesn't contain business logic; it coordinates the workflow.

---

### Q: How does @RequestBody work internally?

**Strong Answer:**
> RequestMappingHandlerAdapter detects @RequestBody and uses RequestResponseBodyMethodProcessor. It looks at Content-Type header, selects the appropriate HttpMessageConverter (Jackson for JSON), and the converter deserializes the request body InputStream into the Java object using ObjectMapper. If @Valid is present, Bean Validation runs before the method is invoked.

---

### Q: Why doesn't @RestController use ViewResolver?

**Strong Answer:**
> @RestController includes @ResponseBody on all methods. This tells Spring that the return value should be written directly to the response body, not interpreted as a view name. So instead of ViewResolver, Spring uses HttpMessageConverters — Jackson by default for JSON — to serialize the object. ViewResolver is completely bypassed in the REST flow.

---

### Q: What happens when an exception is thrown in a controller?

**Strong Answer:**
> DispatcherServlet catches it and delegates to HandlerExceptionResolver chain. First it checks for @ExceptionHandler methods in the controller, then @ControllerAdvice classes, then @ResponseStatus annotations. The first resolver that can handle the exception generates a response — either a ModelAndView for error page or ResponseEntity for REST. If no resolver handles it, the exception propagates and becomes a 500 error.

---

### Q: How would you handle 10,000 concurrent requests in Spring MVC?

**Strong Answer:**
> With default blocking MVC, I'd hit thread pool limits. Options: 1) Scale horizontally with load balancer. 2) Use async Servlet support with CompletableFuture to release threads during I/O. 3) For fully reactive, consider WebFlux. But most importantly, ensure external calls and database queries are optimized — the real bottleneck is usually I/O latency, not the framework.

---

## 13) Interview Speaking Mode

### Short Spoken Answers

#### DispatcherServlet
> DispatcherServlet is the front controller. Every request goes through it. It finds which controller method should handle the request, runs interceptors, invokes the method, and either renders a view or serializes JSON. It coordinates everything but doesn't do business logic.

#### Filter vs Interceptor
> Filters run before Spring gets involved — good for auth, CORS, low-level stuff. Interceptors run inside Spring MVC and can access the model. I use filters for things that don't need Spring context, interceptors when I need to modify what the controller returns.

#### RequestBody internals
> Spring sees @RequestBody, picks the right message converter based on Content-Type — usually Jackson for JSON. Jackson reads the request body and deserializes it to your Java object. If you have @Valid, validation runs before your method is called.

#### REST vs MVC
> Traditional MVC returns view names that get resolved to HTML templates. REST controllers use @ResponseBody — the object goes straight to Jackson, serialized to JSON, no view resolution at all. That's why @RestController doesn't touch ViewResolver.

#### Exception handling
> DispatcherServlet catches exceptions and asks HandlerExceptionResolvers what to do. @ExceptionHandler methods get checked first, then @ControllerAdvice. The resolver either returns an error view or ResponseEntity. If nothing handles it, you get a 500.

#### Performance concerns
> Spring MVC is blocking — one thread per request. If you have slow database queries or external API calls, threads wait. Under heavy load, you can exhaust the thread pool. Solutions: optimize I/O, use caching, or for extreme cases, async processing or WebFlux.

---

### Bad vs Good Answers

| Question | Weak Answer | Strong Answer |
|----------|-------------|---------------|
| DispatcherServlet? | "It handles requests" | "It's the front controller that coordinates the entire flow — finds handlers, manages interceptors, handles both view-based and REST responses through different paths" |
| Filter vs Interceptor? | "Filter runs first" | "Filters are Servlet-level, before Spring context exists. Interceptors are Spring MVC, have ModelAndView access. Different use cases: auth vs business logic" |
| @RequestBody? | "It converts JSON to object" | "HandlerAdapter detects it, uses RequestResponseBodyMethodProcessor, selects HttpMessageConverter based on Content-Type, Jackson deserializes. @Valid triggers Bean Validation" |
| REST vs MVC? | "REST returns JSON, MVC returns HTML" | "@RestController includes @ResponseBody which bypasses ViewResolver entirely. Return value goes through HttpMessageConverter instead of view resolution" |
| Exceptions? | "Use @ExceptionHandler" | "DispatcherServlet delegates to HandlerExceptionResolver chain — @ExceptionHandler, @ControllerAdvice, @ResponseStatus, in that order" |

---

### Common Interviewer Follow-ups

After your answer, expect these:

**On DispatcherServlet:**
- "What's the difference between HandlerMapping and HandlerAdapter?"
- "How does DispatcherServlet know which interceptor to run?"
- "Can you have multiple DispatcherServlets?"

**On Filters/Interceptors:**
- "Can a Filter access Spring beans?"
- "What happens if preHandle returns false?"
- "How do you order interceptors?"

**On Data Binding:**
- "How would you customize JSON serialization?"
- "What happens if JSON has unknown properties?"
- "How do you handle date format in JSON?"

**On Exception Handling:**
- "What if multiple @ControllerAdvice can handle the same exception?"
- "How do you handle validation errors uniformly?"
- "Can you redirect from an exception handler?"

**On Performance:**
- "How many threads does Tomcat use by default?"
- "What's the alternative to blocking I/O?"
- "How would you handle long-running requests?"

---

### 1-Minute Summary Answers

**"Explain Spring MVC architecture"**
> Spring MVC uses Front Controller pattern. DispatcherServlet receives all requests, delegates to HandlerMapping to find the controller method, HandlerAdapter to invoke it. Interceptors run before and after. For traditional MVC, ViewResolver renders HTML. For REST, @ResponseBody bypasses views — Jackson serializes directly to JSON.

**"Filter vs Interceptor"**
> Filters run at Servlet level before Spring context, good for auth and CORS. Interceptors run inside Spring MVC, can access and modify ModelAndView. Use filters for low-level concerns, interceptors for business-related processing.

**"How @RequestBody works"**
> HandlerAdapter detects @RequestBody annotation, uses RequestResponseBodyMethodProcessor which selects HttpMessageConverter based on Content-Type. For JSON, Jackson deserializes request body to the parameter object. @Valid triggers Bean Validation.

**"Why @RestController doesn't use ViewResolver"**
> @RestController adds @ResponseBody to all methods, telling Spring to serialize return values directly rather than interpreting them as view names. HttpMessageConverter handles the serialization, ViewResolver is skipped entirely.

**"Exception handling flow"**
> DispatcherServlet catches exceptions and delegates to HandlerExceptionResolver chain. Priority: @ExceptionHandler in controller, @ControllerAdvice, @ResponseStatus annotation, DefaultHandlerExceptionResolver. First match handles it.

---

### Quick Confidence Builders

**Show deep knowledge:**
- "The implementation detail that matters is..."
- "In production, I've seen this cause..."
- "The trade-off between X and Y is..."

**Show practical experience:**
- "When we migrated from X to Y, we learned..."
- "I typically configure this as..."
- "The pitfall we hit was..."

**Signal awareness:**
- "This works well until you scale to..."
- "The default behavior changes in..."
- "This interacts with Spring Security by..."

---

## Summary

Spring MVC implements Front Controller pattern with DispatcherServlet as the central coordinator. Requests flow through Filter chain (Servlet level), then DispatcherServlet which uses HandlerMapping to find controllers and HandlerAdapter to invoke them. Interceptors provide pre/post processing within Spring context. Traditional MVC uses ViewResolver for HTML rendering; REST APIs with @ResponseBody bypass views and use HttpMessageConverter (Jackson) for direct JSON serialization. Filters operate before Spring context exists; Interceptors have full Spring and ModelAndView access. @RequestBody triggers HttpMessageConverter selection based on Content-Type. Exception handling flows through HandlerExceptionResolver chain: @ExceptionHandler, @ControllerAdvice, @ResponseStatus. Spring MVC is blocking by design — one thread per request — which can exhaust thread pools under I/O-heavy loads. Use DTOs to avoid Jackson serialization triggering lazy loading. Understand the request lifecycle to debug issues effectively.
