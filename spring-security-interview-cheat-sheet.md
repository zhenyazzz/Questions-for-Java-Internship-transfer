# Spring Security Interview Cheat Sheet (Middle/Senior)

Deep dive into filter chain, authentication flow, SecurityContext, and internals.

---

## 1) Where Spring Security Sits in Request Flow (CRITICAL)

### The Complete Request Flow

```
HTTP Request arrives
    ↓
Servlet Container (Tomcat) receives request
    ↓
Servlet Filter Chain (first entry point)
    ↓
    ├─ CharacterEncodingFilter
    ├─ CORSFilter
    ├─ [SPRING SECURITY FILTERS START HERE]
    ├─ DelegatingFilterProxy / FilterChainProxy ← Spring Security entry
    │   ├─ SecurityContextPersistenceFilter
    │   ├─ UsernamePasswordAuthenticationFilter (if form login)
    │   ├─ BearerTokenAuthenticationFilter (if JWT)
    │   ├─ AuthorizationFilter
    │   └─ ... more security filters
    ├─ [SPRING SECURITY FILTERS END]
    ↓
DispatcherServlet (Spring MVC starts here)
    ↓
HandlerMapping, Interceptors, Controller
    ↓
Service → Repository → Database
    ↓
Response back through Filter chain (reverse order)
```

### Critical Positioning

**Spring Security runs BEFORE Spring MVC.**

By the time your `@Controller` method executes:
- Authentication is already done
- SecurityContext already contains the user (or anonymous)
- Authorization checks already passed
- Or request was rejected with 401/403

**Why this matters:**
Your controller assumes the user is authenticated. If you see `principal` as null in controller, something failed in the filter chain.

**Interview Answer:**
Spring Security sits in the Servlet Filter chain, before Spring MVC's DispatcherServlet. When a request arrives, it passes through security filters first — authentication, authorization, JWT validation. Only if all security checks pass does the request reach your controller. By then, SecurityContext already holds the authenticated user, or the request was rejected with 401/403.

---

## 2) Core Architecture

### SecurityFilterChain

A **chain of security filters** that processes requests:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .sessionManagement(session -> session.sessionCreationPolicy(STATELESS))
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
}
```

This builds a `DefaultSecurityFilterChain` containing configured filters in specific order.

### FilterChainProxy

**The central dispatcher for security filters.**

```
Request → FilterChainProxy.doFilterInternal()
    ↓
Determines which SecurityFilterChain applies (can have multiple!)
    ↓
Executes filters in that chain sequentially
    ↓
Either: Request proceeds to DispatcherServlet
Or:     Response sent (401/403)
```

**Multiple chains example:**
```java
@Bean
@Order(1)  // Higher priority
public SecurityFilterChain apiChain(HttpSecurity http) {
    return http
        .securityMatcher("/api/**")  // Only for /api paths
        .oauth2ResourceServer(oauth2 -> oauth2.jwt())
        .build();
}

@Bean
@Order(2)
public SecurityFilterChain defaultChain(HttpSecurity http) {
    return http
        .formLogin(withDefaults())  // Form login for everything else
        .build();
}
```

FilterChainProxy picks the first matching chain.

### DelegatingFilterProxy (Bridge to Servlet)

**The bridge between Servlet container and Spring.**

```java
// In web.xml or Spring Boot auto-config
<filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
```

**What it does:**
1. Servlet container calls DelegatingFilterProxy (standard Servlet Filter)
2. It looks up bean named "springSecurityFilterChain" in Spring context
3. Delegates to the actual FilterChainProxy bean

**Why needed:**
Servlet filters are created by container, not Spring. DelegatingFilterProxy bridges this gap, letting Spring manage the security filter chain.

**Spring Boot auto-config:**
```java
// SpringBootWebSecurityConfiguration automatically registers:
DelegatingFilterProxyRegistrationBean
    → DelegatingFilterProxy
        → FilterChainProxy
```

### OncePerRequestFilter

**Base class for security filters ensuring single execution.**

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) {
        // Extract and validate JWT
        // Set authentication in SecurityContext
        
        filterChain.doFilter(request, response);  // Continue chain
    }
}
```

**Why OncePerRequestFilter:**
In some dispatch scenarios (forward, include), filters might be called multiple times. This base class ensures your logic runs exactly once per request.

**Deep Insight:**
The filter chain is not a linked list of Filter objects. It's a List<Filter> in SecurityFilterChain, and FilterChainProxy iterates through it, calling each filter's doFilter() method.

---

## 3) Filter Chain Deep Dive (VERY IMPORTANT)

### Filter Order Matters

Spring Security filters execute in a specific order. Changing order changes behavior dramatically.

**Default filter order (simplified):**
```
1.  ChannelProcessingFilter          (HTTP vs HTTPS)
2.  SecurityContextPersistenceFilter  (Load/save SecurityContext)
3.  ConcurrentSessionFilter          (Session concurrency)
4.  HeaderWriterFilter               (Security headers)
5.  CsrfFilter                       (CSRF token validation)
6.  LogoutFilter                     (Logout handling)
7.  UsernamePasswordAuthenticationFilter  (Form login)
8.  DefaultLoginPageGeneratingFilter (Default login page)
9.  BasicAuthenticationFilter        (HTTP Basic)
10. RequestCacheAwareFilter          (Save/restore request)
11. SecurityContextHolderAwareRequestFilter
12. RememberMeAuthenticationFilter   (Remember-me cookie)
13. AnonymousAuthenticationFilter    (Anonymous user)
14. ExceptionTranslationFilter       (Handle auth exceptions)
15. AuthorizationFilter              (URL-based authorization)
```

### Key Filters Explained

**SecurityContextPersistenceFilter**
```
Request arrives
    ↓
Load SecurityContext from Session (or request attribute for stateless)
    ↓
Store in SecurityContextHolder (ThreadLocal)
    ↓
Request continues...
    ↓
Request completes
    ↓
Save SecurityContext back to Session
    ↓
Clear SecurityContextHolder
```

**Critical:** Without this filter, SecurityContext wouldn't persist across request/response cycle.

**UsernamePasswordAuthenticationFilter**

For form-based login (`/login` POST):
```java
// Extract username/password from request
String username = request.getParameter("username");
String password = request.getParameter("password");

// Create Authentication token
UsernamePasswordAuthenticationToken authRequest =
    new UsernamePasswordAuthenticationToken(username, password);

// Authenticate
Authentication authentication = 
    this.getAuthenticationManager().authenticate(authRequest);

// Store in SecurityContext
SecurityContextHolder.getContext().setAuthentication(authentication);
```

**BearerTokenAuthenticationFilter (JWT/OAuth2)**

```java
String token = resolveToken(request);  // Extract from Authorization header

if (token != null) {
    // Create bearer token authentication
    BearerTokenAuthenticationToken bearerToken = 
        new BearerTokenAuthenticationToken(token);
    
    // Authenticate (will validate token)
    Authentication authentication = 
        this.getAuthenticationManager().authenticate(bearerToken);
    
    // Store context
    SecurityContextHolder.getContext().setAuthentication(authentication);
}

filterChain.doFilter(request, response);
```

**AuthorizationFilter**

Last filter in chain (before request reaches controller):
```java
// Check if current authentication has access to requested URL
AuthorizationDecision decision = 
    this.authorizationManager.check(authentication, request);

if (!decision.isGranted()) {
    throw new AccessDeniedException("Access Denied");
}

filterChain.doFilter(request, response);
```

**Interview Answer:**
Spring Security uses a chain of filters, each handling a specific concern. SecurityContextPersistenceFilter loads/stores the security context. Authentication filters like UsernamePasswordAuthenticationFilter or BearerTokenAuthenticationFilter extract credentials, authenticate via AuthenticationManager, and store the result in SecurityContext. AuthorizationFilter checks if the authenticated user has access to the requested resource. Order matters — authentication must happen before authorization.

---

## 4) Authentication vs Authorization

### Definitions

**Authentication = WHO YOU ARE**
- Verifies identity
- Examples: username/password, JWT token, OAuth2 token
- Result: `Authentication` object in SecurityContext

**Authorization = WHAT YOU CAN DO**
- Verifies permissions
- Examples: hasRole("ADMIN"), hasPermission("ORDER_READ")
- Happens after authentication

### The Authentication Object

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();  // Roles/permissions
    Object getCredentials();   // Password (usually cleared after auth)
    Object getDetails();       // Additional details (IP, session ID)
    Object getPrincipal();     // User identity (UserDetails, String, etc.)
    boolean isAuthenticated(); // true if authenticated
    void setAuthenticated(boolean isAuthenticated);
}
```

**Common implementations:**
- `UsernamePasswordAuthenticationToken` — form login
- `RememberMeAuthenticationToken` — remember-me
- `AnonymousAuthenticationToken` — anonymous user
- `BearerTokenAuthentication` — JWT/OAuth2
- `PreAuthenticatedAuthenticationToken` — pre-authenticated scenarios

### GrantedAuthority

```java
public interface GrantedAuthority extends Serializable {
    String getAuthority();  // "ROLE_ADMIN", "READ_ORDER", etc.
}
```

**SimpleGrantedAuthority** is the standard implementation.

**Interview Answer:**
Authentication verifies who you are — checking credentials like username/password or validating a JWT token. The result is an Authentication object containing the user's identity and authorities. Authorization happens next, checking if that authenticated user has permission to access the requested resource. You can be authenticated but still denied access if you lack the required role or permission.

---

## 5) Authentication Flow (STEP-BY-STEP)

### Example: Form Login

```
1. User submits /login with username=john&password=secret
   
2. UsernamePasswordAuthenticationFilter intercepts
   └─ Checks if URL matches login processing URL (default: /login POST)
   
3. Extract credentials
   └─ username = request.getParameter("username")
   └─ password = request.getParameter("password")
   
4. Create Authentication token
   └─ UsernamePasswordAuthenticationToken(unauthenticated)
      ├─ principal = username (String)
      ├─ credentials = password
      └─ authenticated = false
      
5. Pass to AuthenticationManager
   └─ AuthenticationManager.authenticate(token)
   
6. AuthenticationManager delegates to ProviderManager
   └─ Tries each AuthenticationProvider until one supports the token
   
7. DaoAuthenticationProvider processes it
   └─ Calls UserDetailsService.loadUserByUsername(username)
   └─ UserDetailsService queries database: SELECT * FROM users WHERE username=?
   
8. Password comparison
   └─ DaoAuthenticationProvider compares submitted password 
      with UserDetails.password using PasswordEncoder
   └─ If match: success. If not: BadCredentialsException
   
9. Create authenticated Authentication
   └─ UsernamePasswordAuthenticationToken(authenticated)
      ├─ principal = UserDetails object (or custom User)
      ├─ credentials = null (erased)
      ├─ authorities = List<GrantedAuthority> from UserDetails
      └─ authenticated = true
      
10. Store in SecurityContext
    └─ SecurityContextHolder.getContext().setAuthentication(authenticated)
    
11. SecurityContextPersistenceFilter saves context to session
    └─ Session now contains the authenticated user
    
12. Success handler redirects (default: saved request or root)
    └─ Or returns JSON for REST APIs
```

### Example: JWT Authentication

```
1. Request arrives with header: Authorization: Bearer eyJhbGciOiJ...

2. BearerTokenAuthenticationFilter (or custom JwtAuthFilter) intercepts

3. Extract token
   └─ Remove "Bearer " prefix
   └─ Validate JWT signature (check signing key)
   └─ Validate expiration (not expired)
   └─ Validate issuer/audience if configured

4. Extract claims from JWT payload
   └─ sub (subject/username)
   └─ roles/authorities
   └─ exp (expiration)

5. Create Authentication
   └─ Could create UsernamePasswordAuthenticationToken or custom
   └─ principal = username from JWT
   └─ authorities = roles from JWT claims
   └─ details = token claims

6. OR: Pass to AuthenticationManager for validation
   └─ JwtAuthenticationProvider validates token structure
   
7. Store in SecurityContext
   └─ SecurityContextHolder.getContext().setAuthentication(jwtAuth)
   
8. Request proceeds to controller
   └─ Controller can access: SecurityContextHolder.getContext().getAuthentication()
   
9. Response returns
   └─ SecurityContextPersistenceFilter clears context (stateless — no session)
```

**Key difference:**
- Form login: Session-based, SecurityContext stored in HTTP Session
- JWT: Stateless, SecurityContext exists only for this request, reconstructed from token each time

**Interview Answer:**
For form login, UsernamePasswordAuthenticationFilter extracts credentials, creates an unauthenticated token, and passes it to AuthenticationManager which uses DaoAuthenticationProvider. This loads UserDetails via UserDetailsService, compares passwords with PasswordEncoder, and if valid, creates an authenticated token stored in SecurityContext. For JWT, BearerTokenAuthenticationFilter extracts the token, validates it (signature, expiration), extracts user info from claims, creates an Authentication object, and stores it in SecurityContext — all stateless, no session involved.

---

## 6) SecurityContext (CRITICAL)

### What It Is

SecurityContext holds the **currently authenticated user** for the duration of a request:

```java
public interface SecurityContext extends Serializable {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}
```

### Where It's Stored: ThreadLocal

```java
// SecurityContextHolder uses ThreadLocal
private static final ThreadLocal<SecurityContext> contextHolder = 
    new ThreadLocal<>();
```

**Per-request storage:**
- Each request gets its own thread (Servlet model)
- ThreadLocal ensures isolation between concurrent requests
- SecurityContext is request-scoped, not global

**Access patterns:**
```java
// In controller
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();

// In service
public void processOrder(Long orderId) {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    String userId = auth.getName();
    // Check if user can access this order...
}

// Better: inject Authentication as parameter
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id, Authentication auth) {
    // Spring injects from SecurityContext
}
```

### How It's Managed

**SecurityContextPersistenceFilter lifecycle:**

```
Request arrives
    ↓
MODE_THREADLOCAL (default):
    - Check if SecurityContext already in ThreadLocal (forwards)
    - If not, check HttpSession for context
    - If found in session, copy to ThreadLocal
    - If not, create empty context (anonymous)
    
Request processing
    ↓
Authentication filters may set context
    ↓
Request completes
    ↓
Save context back to session (if MODE_THREADLOCAL and not stateless)
    ↓
Clear ThreadLocal (critical!)
```

**Stateless mode (JWT):**
```java
http.sessionManagement(session -> 
    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
);
```

- No session created
- SecurityContext not persisted between requests
- Each request must re-authenticate from token
- SecurityContextPersistenceFilter still clears ThreadLocal

**Common Trap:**
Accessing SecurityContextHolder in async code (CompletableFuture, @Async):
```java
@Async
public void asyncTask() {
    // WRONG: SecurityContext lost!
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    // auth is null or anonymous
}
```

**Fix:**
```java
// Propagate context to async thread
SecurityContext context = SecurityContextHolder.getContext();
CompletableFuture.runAsync(() -> {
    SecurityContextHolder.setContext(context);
    // Now auth is available
});
```

Or use `DelegatingSecurityContextExecutor`.

**Interview Answer:**
SecurityContext holds the current Authentication and is stored in ThreadLocal, making it per-thread and effectively per-request in the Servlet model. SecurityContextPersistenceFilter manages it — loading from session at request start and saving back at end. For stateless JWT, it's not persisted; each request builds a fresh context from the token. Access it via SecurityContextHolder, but be careful in async code where the context doesn't automatically propagate to new threads.

---

## 7) JWT Authentication Flow

### Complete Stateless Flow

```
┌─────────────────────────────────────────────────────────┐
│ CLIENT                                                    │
│ 1. POST /login with credentials                          │
│ 2. Receive JWT token in response                         │
│ 3. Store token (localStorage/cookie)                      │
│ 4. Subsequent requests: Authorization: Bearer <token>    │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│ SERVER (Per Request)                                      │
│                                                           │
│ Request with Bearer token                                │
│     ↓                                                     │
│ JwtAuthenticationFilter (or BearerTokenAuthenticationFilter)│
│     ↓                                                     │
│ 1. Extract token from Authorization header                │
│     String token = header.replace("Bearer ", "");         │
│     ↓                                                     │
│ 2. Validate token                                         │
│     - Parse JWT (extract claims)                        │
│     - Verify signature (check signing key)               │
│     - Check expiration (exp claim)                       │
│     - Optional: validate issuer, audience                │
│     ↓                                                     │
│ 3. If valid, extract user info                           │
│     String username = claims.getSubject();                │
│     List<String> roles = claims.get("roles", List.class); │
│     ↓                                                     │
│ 4. Create Authentication object                          │
│     UsernamePasswordAuthenticationToken auth =            │
│         new UsernamePasswordAuthenticationToken(          │
│             username, null,                              │
│             roles.stream().map(SimpleGrantedAuthority::new)│
│                 .collect(Collectors.toList())             │
│         );                                                │
│     ↓                                                     │
│ 5. Store in SecurityContext                               │
│     SecurityContextHolder.getContext().setAuthentication(auth)│
│     ↓                                                     │
│ 6. Request proceeds to controller                         │
│     Controller checks authentication/authorization         │
│     via @PreAuthorize or SecurityContextHolder            │
│     ↓                                                     │
│ 7. Response returns                                        │
│     SecurityContextPersistenceFilter clears ThreadLocal   │
│     (No session stored — completely stateless)           │
└─────────────────────────────────────────────────────────┘
```

### JWT Filter Implementation Example

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtTokenProvider tokenProvider;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                      HttpServletResponse response,
                                      FilterChain filterChain) 
            throws ServletException, IOException {
        
        try {
            String jwt = getJwtFromRequest(request);
            
            if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)) {
                String username = tokenProvider.getUsernameFromJWT(jwt);
                List<String> roles = tokenProvider.getRolesFromJWT(jwt);
                
                // Create Authentication
                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        username, null,
                        roles.stream()
                            .map(SimpleGrantedAuthority::new)
                            .collect(Collectors.toList())
                    );
                
                authentication.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                );
                
                // Set in context
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            logger.error("Could not set user authentication in security context", ex);
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### Stateless Nature

**No server-side session:**
- Server doesn't store token state
- Token contains all needed info (user, roles, expiration)
- Server only validates signature and expiration
- Horizontal scaling is trivial (no session replication)

**Trade-offs:**
- Token can't be revoked easily (need blacklist)
- Token size adds to every request
- Token refresh logic needed

**Interview Answer:**
JWT authentication is completely stateless. Each request includes the token in the Authorization header. The server validates the signature and expiration, extracts user info from the token claims, creates an Authentication object, and stores it in SecurityContext for that request only. No session is created or stored server-side, making horizontal scaling trivial. The trade-off is token revocation — since the server doesn't track tokens, you need a blacklist or short expiration times to handle logout.

---

## 8) UserDetailsService

### Role in Authentication

**UserDetailsService is the bridge to your user database.**

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

**Responsibilities:**
1. Load user by identifier (username, email)
2. Return UserDetails with credentials and authorities
3. Throw UsernameNotFoundException if user doesn't exist

### Standard Implementation

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    private final UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));
        
        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),  // Must be encoded (BCrypt)
            user.isEnabled(),
            true, true, true,
            getAuthorities(user.getRoles())
        );
    }
    
    private Collection<? extends GrantedAuthority> getAuthorities(Set<Role> roles) {
        return roles.stream()
            .flatMap(role -> role.getPermissions().stream())
            .map(permission -> new SimpleGrantedAuthority(permission.getName()))
            .collect(Collectors.toList());
    }
}
```

### Custom UserDetails

```java
public class CustomUserDetails implements UserDetails {
    private final Long id;
    private final String username;
    private final String password;
    private final boolean enabled;
    private final Collection<? extends GrantedAuthority> authorities;
    private final String department;  // Custom field
    
    // Constructor, getters, UserDetails methods
    
    public Long getId() { return id; }
    public String getDepartment() { return department; }
}
```

**Benefit:** Can store additional user info beyond what Spring Security needs.

**Interview Answer:**
UserDetailsService loads user data from your database during authentication. DaoAuthenticationProvider calls it to get UserDetails containing the encoded password and authorities. It then compares the submitted password using PasswordEncoder. I typically implement it to load from JPA repository, map my User entity to UserDetails, and include any custom fields like department or tenant ID that I need in the security context.

---

## 9) Authorization Mechanisms

### URL-Based (HttpSecurity)

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            // Public endpoints
            .requestMatchers("/", "/login", "/register", "/public/**").permitAll()
            
            // Static resources
            .requestMatchers("/css/**", "/js/**", "/images/**").permitAll()
            
            // Role-based access
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .requestMatchers("/api/orders/**").hasAnyRole("USER", "ADMIN")
            
            // Authority-based (more granular than roles)
            .requestMatchers("/api/reports/**").hasAuthority("REPORT_READ")
            
            // Pattern matching
            .requestMatchers("/user/{id}/orders/**").access(
                new WebExpressionAuthorizationManager(
                    "hasRole('USER') and #id == authentication.name"
                )
            )
            
            // Everything else requires authentication
            .anyRequest().authenticated()
        );
    return http.build();
}
```

### Method-Level Security

**@PreAuthorize (most powerful):**
```java
@Service
public class OrderService {
    
    @PreAuthorize("hasRole('USER')")
    public Order createOrder(OrderRequest request) {
        // Only authenticated users with USER role
    }
    
    @PreAuthorize("hasRole('ADMIN') or #orderId == principal.username")
    public Order getOrder(Long orderId) {
        // Admin can view any order, user can view own orders
    }
    
    @PreAuthorize("@orderSecurityService.canEditOrder(#orderId, authentication)")
    public void updateOrder(Long orderId, OrderUpdateRequest request) {
        // Delegate to custom security service for complex logic
    }
}
```

**@PostAuthorize (filter results):**
```java
@PostAuthorize("returnObject.owner == authentication.name")
public Order getOrder(Long orderId) {
    // Method executes, but if returned order isn't owned by current user,
    // AccessDeniedException is thrown
    return orderRepository.findById(orderId).orElseThrow();
}
```

**@Secured (simpler, older):**
```java
@Secured("ROLE_ADMIN")  // Note: must include ROLE_ prefix
public void deleteUser(Long userId) { }
```

**Enable method security:**
```java
@Configuration
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class SecurityConfig { }
```

### Custom Permission Evaluator

For complex authorization logic:

```java
@Component
public class OrderPermissionEvaluator implements PermissionEvaluator {
    
    @Override
    public boolean hasPermission(Authentication auth, Object targetDomain, Object permission) {
        if ((auth == null) || (targetDomain == null) || !(permission instanceof String)) {
            return false;
        }
        
        Order order = (Order) targetDomain;
        String currentUser = auth.getName();
        
        // Owner can do anything
        if (order.getOwner().equals(currentUser)) {
            return true;
        }
        
        // Check specific permission
        return switch (permission.toString()) {
            case "READ" -> hasAuthority(auth, "ORDER_READ");
            case "DELETE" -> hasAuthority(auth, "ORDER_DELETE");
            default -> false;
        };
    }
    
    private boolean hasAuthority(Authentication auth, String authority) {
        return auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals(authority));
    }
}
```

**Interview Answer:**
Spring Security provides URL-based authorization in the security filter chain, checking if the authenticated user has required roles before allowing access to specific paths. For more granular control, I use method-level security with @PreAuthorize which supports SpEL expressions — hasRole, hasAuthority, or even referencing method parameters like #orderId. For complex business rules, I implement custom PermissionEvaluator or delegate to a security service that checks ownership, department permissions, or other business logic.

---

## 10) Filters vs Interceptors (IMPORTANT)

### Where They Sit

```
Request
    ↓
Servlet Filter Chain (Spring Security lives here)
    ↓
DispatcherServlet
    ↓
HandlerInterceptor.preHandle()  ← Spring MVC Interceptor
    ↓
Controller
    ↓
HandlerInterceptor.postHandle()
```

### Differences

| Aspect | Filter (Spring Security) | Interceptor (Spring MVC) |
|--------|---------------------------|--------------------------|
| **Level** | Servlet API | Spring MVC |
| **Access to** | HttpServletRequest/Response | Request/Response + Controller info |
| **Runs before** | Spring MVC | Controller execution |
| **Can stop request** | Yes, send response | Yes (return false) |
| **Knows controller** | No | Yes (HandlerMethod) |
| **Knows Spring Security** | Is Spring Security | Can access SecurityContext |

### When to Use Each

**Use Filters when:**
- Authentication/verification before Spring MVC
- Token validation (JWT)
- Request/response modification at low level
- Security concerns (Spring Security itself)

**Use Interceptors when:**
- Need controller metadata (which method was called)
- Logging with method information
- Post-processing model before view rendering
- MVC-specific concerns

**Interview Answer:**
Filters work at Servlet level, before Spring MVC gets involved. Spring Security uses filters for authentication and authorization because it needs to protect the application before any Spring processing happens. Interceptors work inside Spring MVC and have access to which controller method is being called — useful for logging or modifying the model. Filters for security and cross-cutting concerns before Spring, interceptors for MVC-specific post-processing.

---

## 11) Common Pitfalls

### SecurityContext Not Set

**Symptom:** User shows as "anonymous" in controller despite sending credentials.

**Causes:**
1. **Filter not firing:** Missing `.addFilterBefore()` in SecurityFilterChain
2. **Wrong filter order:** JWT filter after AuthorizationFilter
3. **Token validation failing silently:** Catch block swallows exception
4. **URL pattern mismatch:** Filter configured for `/api/**` but hitting `/api/v2/**`

**Debug:**
```java
// Add logging filter
@Component
public class DebugFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                      HttpServletResponse response, 
                                      FilterChain chain) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        System.out.println("Auth: " + auth);  // Should not be null
        chain.doFilter(request, response);
    }
}
```

### Wrong Filter Order

**Common mistake:**
```java
http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
```

But UsernamePasswordAuthenticationFilter may not even be in your chain if you're stateless!

**Correct approach:**
```java
http.addFilterBefore(jwtAuthFilter, AuthorizationFilter.class);
// AuthorizationFilter is always present, check happens after auth
```

### Stateless vs Session Confusion

**Mistake:** Mixing session and JWT:
```java
http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
// But then:
SecurityContextPersistenceFilter still tries to save to session!
```

**Solution:**
For true stateless JWT:
```java
http
    .sessionManagement(session -> 
        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    .securityContext(context -> 
        context.securityContextRepository(NoOpSecurityContextRepository.getInstance()));
```

### Missing Exception Handling

**Problem:**
```java
@Override
protected void doFilterInternal(...) {
    String token = extractToken(request);
    // If extraction throws, filter chain breaks!
    validateToken(token);
    // If validation fails, no response sent, chain continues with null auth
}
```

**Fix:**
```java
@Override
protected void doFilterInternal(...) {
    try {
        String token = extractToken(request);
        if (token != null) {
            validateAndSetAuth(token);
        }
    } catch (JwtException ex) {
        // Send 401 and don't continue chain
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.getWriter().write("Invalid token");
        return;
    }
    filterChain.doFilter(request, response);
}
```

### JWT Not Validated Properly

**Bad:**
```java
// Only parsing, not validating signature!
Jwts.parser().parseClaimsJws(token);  // No signature validation
```

**Good:**
```java
Jwts.parser()
    .verifyWith(secretKey)
    .build()
    .parseSignedClaims(token);  // Validates signature
```

---

## 12) Debugging Security (VERY IMPORTANT)

### Enable Debug Logging

```yaml
logging:
  level:
    org.springframework.security: DEBUG
```

**What you'll see:**
```
Securing POST /api/orders
Created security context: SecurityContextImpl [Null authentication]
Checking authorization for '/api/orders' using org.springframework.security...
Authenticated user, setting security context: SecurityContextImpl[Authentication=...]
Secured POST /api/orders
```

### Filter Chain Debugging

```java
@Configuration
public class DebugConfig {
    
    @Bean
    public CommandLineRunner printFilters(FilterChainProxy filterChainProxy) {
        return args -> {
            filterChainProxy.getFilterChains().forEach(chain -> {
                System.out.println("Filter chain: " + chain);
                chain.getFilters().forEach(filter -> 
                    System.out.println("  - " + filter.getClass().getSimpleName())
                );
            });
        };
    }
}
```

### Understand Rejections

**401 Unauthorized:**
- No credentials provided
- Credentials invalid (bad password, expired token)
- Authentication filter failed

**403 Forbidden:**
- Authenticated but not authorized (wrong role/permission)
- AuthorizationFilter rejection

**Common causes checklist:**
1. Is filter registered? Check SecurityFilterChain
2. Is filter order correct? Should be before AuthorizationFilter
3. Is SecurityContext being set? Add debug logging
4. Is token valid? Check signature and expiration
5. Is user principal correct? Verify UserDetailsService returns expected authorities

**Interview Answer:**
To debug Spring Security, I enable DEBUG logging for org.springframework.security — it shows which filters run and the authentication flow. If a request is rejected, I check if the filter is registered and in the right order. For 401, the issue is usually authentication — token validation or UserDetails lookup. For 403, it's authorization — the user is authenticated but lacks the required role. I also verify the SecurityContext is being populated by adding a debug filter that logs the authentication state.

---

## 13) Spring Security vs Custom Security

### Why Not Write Your Own

**Attempting custom JWT filter:**
```java
// Easy to get wrong:
- Signature validation bugs (timing attacks)
- Missing expiration checks
- Not handling token refresh
- No protection against replay attacks
- Edge cases in error handling
```

**Spring Security provides:**
- Battle-tested crypto (PasswordEncoder, JWT libraries)
- Protection against common attacks (CSRF, session fixation)
- Standard patterns (OAuth2, SAML)
- Comprehensive testing
- Active security community

### When Custom Makes Sense

- Very specific requirements not covered by standards
- Proprietary authentication protocols
- Learning/educational projects

**Rule:** Use Spring Security for production. Build custom only for learning.

**Interview Answer:**
I use Spring Security in production because it handles the hard security problems — proper password encoding, protection against timing attacks, CSRF, session fixation. Rolling your own JWT validation is risky; it's easy to miss signature verification steps or expiration checks. Spring Security has been battle-tested and follows security best practices. I'd only write custom security code for learning or if there was a very specific protocol requirement not covered by existing standards.

---

## 14) Interview Questions with Strong Answers

### Q: How does Spring Security work internally?

**Strong Answer:**
> Spring Security operates as a chain of Servlet Filters. DelegatingFilterProxy bridges from the Servlet container to Spring's FilterChainProxy, which manages one or more SecurityFilterChains. Each request passes through security filters — authentication filters extract and validate credentials, create an Authentication object, and store it in SecurityContext (ThreadLocal). AuthorizationFilter checks if the authenticated user has access to the requested resource. If all checks pass, the request reaches your controller. If any check fails, an early response (401/403) is sent.

---

### Q: What is the filter chain?

**Strong Answer:**
> The security filter chain is an ordered list of filters that process every request. Each filter has a specific responsibility — SecurityContextPersistenceFilter loads and saves the security context, UsernamePasswordAuthenticationFilter or BearerTokenAuthenticationFilter handle authentication, AuthorizationFilter performs access checks. Filter order is critical; authentication must happen before authorization. You can have multiple chains for different URL patterns (e.g., one for API with JWT, another for web with form login), and FilterChainProxy selects the first matching chain.

---

### Q: How does JWT authentication work?

**Strong Answer:**
> JWT authentication is stateless. Each request includes the token in the Authorization header. A filter — BearerTokenAuthenticationFilter or a custom JwtAuthenticationFilter — extracts the token, validates the signature against a secret key, checks expiration, and extracts claims. If valid, it creates an Authentication object with the user details and authorities from the token claims, and stores it in SecurityContext. No session is created. The SecurityContext exists only for that request and is cleared afterward. This makes scaling horizontally trivial since no server-side session state needs to be shared.

---

### Q: What is SecurityContext?

**Strong Answer:**
> SecurityContext holds the currently authenticated user's Authentication object. It's stored in ThreadLocal, making it per-thread and effectively per-request in the Servlet model. SecurityContextPersistenceFilter manages its lifecycle — loading from the HTTP session at request start and saving back at the end for session-based auth. For stateless JWT, it's not persisted. You access it via SecurityContextHolder.getContext().getAuthentication(). In async code, it doesn't automatically propagate to new threads, so you need to explicitly copy it or use DelegatingSecurityContextExecutor.

---

### Q: Authentication vs Authorization?

**Strong Answer:**
> Authentication verifies who you are — checking credentials like username/password or validating a JWT token. The result is an Authentication object containing the principal (user identity) and authorities (roles/permissions). Authorization happens after authentication and decides what you're allowed to do — checking if you have ROLE_ADMIN for an admin endpoint, or if you own the resource you're trying to access. You can pass authentication but fail authorization, which results in 403 Forbidden instead of 401 Unauthorized.

---

### Q: Where does Spring Security sit in request lifecycle?

**Strong Answer:**
> Spring Security sits in the Servlet Filter chain, before Spring MVC's DispatcherServlet. The Servlet container calls DelegatingFilterProxy which delegates to FilterChainProxy. Security filters process the request before it ever reaches your controller. By the time DispatcherServlet starts its MVC processing, authentication is already complete and SecurityContext holds the user, or the request was rejected. This positioning is essential because security decisions must happen before any business logic executes.

---

## 15) Interview Speaking Mode

### Short Spoken Answers

#### Filter chain explained
> Spring Security uses a chain of Servlet Filters. Each filter does one thing — load the security context, authenticate the user, or check authorization. They execute in order. Authentication filters extract credentials from the request, validate them via AuthenticationManager, and store the result in SecurityContext. AuthorizationFilter is near the end, checking if the user can access the URL. If any filter rejects the request, it sends 401 or 403 and stops the chain.

#### JWT flow
> Each request has a Bearer token in the header. My JWT filter extracts it, validates the signature and expiration, parses the claims for username and roles, creates an Authentication object, and puts it in SecurityContext. No session involved — completely stateless. The controller then gets the authenticated user from SecurityContextHolder. Response returns and the context is cleared.

#### SecurityContext
> It holds the current user's Authentication and lives in ThreadLocal. One per request thread. SecurityContextPersistenceFilter loads it from session at request start and saves at end. For JWT I don't persist it — build fresh each request. Access via SecurityContextHolder. Watch out in async code — new threads don't inherit the context automatically.

#### UserDetailsService
> Bridge to your user database. DaoAuthenticationProvider calls it during form login to load the user by username. Returns UserDetails with the encoded password and authorities. Spring compares the submitted password using PasswordEncoder. If match, authentication succeeds. I implement it to query my JPA repository and map to UserDetails with any custom fields I need.

#### Debugging security
> Enable DEBUG logging for org.springframework.security. It shows every filter that runs and the authentication state. If 401, check if token is being extracted and validated — signature, expiration. If 403, check roles and authorities. I add a debug filter to log SecurityContext state. Most issues are filter order wrong or token validation not working as expected.

---

### Bad vs Good Answers

| Question | Weak Answer | Strong Answer |
|----------|-------------|---------------|
| **Filter chain?** | "It filters requests" | "Ordered chain of Servlet Filters — SecurityContextPersistenceFilter loads context, authentication filters validate credentials, AuthorizationFilter checks access. Order matters. Multiple chains can target different URL patterns" |
| **JWT flow?** | "It validates the token" | "Filter extracts Bearer token, validates signature and exp, parses claims, creates Authentication with user/roles from JWT, stores in SecurityContext. Stateless — no session, context cleared after request" |
| **SecurityContext?** | "It holds the user" | "ThreadLocal storage for Authentication. Loaded from session by SecurityContextPersistenceFilter for stateful, reconstructed from JWT for stateless. Per-request isolation. Doesn't propagate to async threads automatically" |
| **Auth vs AuthZ?** | "One is login, one is permissions" | "Authentication verifies identity — credentials, tokens, produces Authentication object. Authorization checks permissions — hasRole, hasAuthority, happens after auth. 401 vs 403" |
| **Position in lifecycle?** | "Before controller" | "Servlet Filter chain, before DispatcherServlet. DelegatingFilterProxy → FilterChainProxy → security filters → MVC. Security decisions complete before any business logic" |

---

### Common Interviewer Follow-ups

**On filter chain:**
- "Can you have multiple security filter chains?"
- "What happens if two filters both try to send a response?"
- "How do you add a custom filter in specific position?"

**On authentication:**
- "What if UserDetailsService is slow? How do you optimize?"
- "How would you implement remember-me?"
- "What is the AuthenticationManager exactly?"

**On JWT:**
- "How do you handle token refresh?"
- "What if you need to revoke tokens?"
- "How long should tokens live?"

**On SecurityContext:**
- "How do you propagate context to @Async methods?"
- "What about reactive/WebFlux?"
- "How do you access it in a non-Spring class?"

**On authorization:**
- "How would you implement row-level security?"
- "Can you use method security with SpEL?"
- "What if authorization depends on business data?"

---

### 1-Minute Summary Answers

**"How does Spring Security work?"**
> Spring Security operates as a chain of Servlet Filters registered via DelegatingFilterProxy. SecurityContextPersistenceFilter manages the SecurityContext in ThreadLocal. Authentication filters like UsernamePasswordAuthenticationFilter or BearerTokenAuthenticationFilter extract credentials, validate via AuthenticationManager, and store the Authentication result. AuthorizationFilter checks access permissions. If all pass, the request reaches your controller; otherwise, 401/403 is sent early.

**"JWT authentication flow?"**
> Stateless authentication where each request includes a JWT in the Authorization header. A filter extracts the token, validates signature and expiration against a secret key, parses claims for user identity and roles, creates an Authentication object, and stores it in ThreadLocal SecurityContext. No server-side session is created. The SecurityContext is request-scoped and cleared after processing, making horizontal scaling trivial.

**"Authentication vs Authorization?"**
> Authentication verifies identity through credentials or tokens, resulting in an Authentication object containing the principal and authorities. Authorization happens afterward, checking if that authenticated user has permission to access the requested resource via roles, authorities, or custom business rules. Authentication failures return 401; authorization failures return 403.

**"Where does Spring Security fit?"**
> It sits in the Servlet Filter chain before Spring MVC. DelegatingFilterProxy bridges from the Servlet container to FilterChainProxy which orchestrates security filters. By the time DispatcherServlet starts, security processing is complete — the user is authenticated in SecurityContext or the request was rejected. This positioning ensures security decisions happen before any business logic executes.

---

### Quick Confidence Builders

**Show deep knowledge:**
- "The key implementation detail is ThreadLocal storage in SecurityContextHolder"
- "Filter order matters because..."
- "The way Spring bridges Servlet filters to Spring-managed beans is through DelegatingFilterProxy"

**Signal production experience:**
- "In production, I always enable security debug logging when troubleshooting"
- "The common pitfall I've seen is..."
- "For performance, I cache UserDetails lookups because..."

---

## Summary

Spring Security operates as a chain of Servlet Filters positioned before Spring MVC's DispatcherServlet. DelegatingFilterProxy bridges the Servlet container to Spring's FilterChainProxy, which manages one or more SecurityFilterChains. SecurityContextPersistenceFilter loads and saves the SecurityContext (stored in ThreadLocal) per request. Authentication filters extract and validate credentials, create an Authentication object via AuthenticationManager and AuthenticationProviders like DaoAuthenticationProvider, and store the result in SecurityContext. AuthorizationFilter performs access control checks. For JWT, the flow is stateless — each request presents a token, which is validated, parsed for claims, converted to Authentication, and stored in context without session persistence. UserDetailsService bridges to the user database during authentication. Method-level security with @PreAuthorize provides granular authorization using SpEL. Filters work at Servlet level before Spring MVC; interceptors work inside Spring MVC and have access to controller metadata. Common pitfalls include wrong filter order, mixing session and stateless modes, missing exception handling in custom filters, and async code where SecurityContext doesn't propagate automatically. Enable DEBUG logging for org.springframework.security to trace filter execution and authentication decisions.
