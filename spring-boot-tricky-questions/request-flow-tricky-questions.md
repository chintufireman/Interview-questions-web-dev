#### What happens when you hit the request to spring boot server
**Ans**
##### Flow Chart:-
    ```
        +--------------------+
        |   Client (Browser) |
        +--------------------+
                |
                | 1. HTTP Request (GET /api/users)
                v
        +--------------------+
        |  Embedded Server   |  (Tomcat / Jetty / Undertow)
        +--------------------+
        | - Accepts socket   |
        | - Parses HTTP req  |
        | - Creates HttpServletRequest/Response |
        | - Assigns worker thread (from pool)   |
        +--------------------+
                |
                | 2. Pass request to Servlet Filter Chain
                v
        +--------------------------------------+
        |  Filter Chain                        |
        |--------------------------------------|
        |  (a) Custom Filters                  |
        |  (b) Spring Security Filter Chain    |
        |  (c) HiddenHttpMethodFilter, etc.    |
        |  (d) DispatcherServlet (last filter) |
        +--------------------------------------+
                |
                | 3. DispatcherServlet receives request
                v
        +------------------------------------+
        | DispatcherServlet                  |
        |------------------------------------|
        | doDispatch()                       |
        |  ├─ Lookup HandlerMapping           |
        |  ├─ Get HandlerExecutionChain       |
        |  ├─ Run preHandle() of Interceptors |
        |  ├─ Invoke HandlerAdapter           |
        |  └─ Run postHandle() + view render  |
        +------------------------------------+
                |
                | 4. HandlerMapping finds Controller
                v
        +------------------------------------------+
        |  RequestMappingHandlerMapping            |
        |------------------------------------------|
        | Matches URL, method, headers, etc.       |
        | Returns HandlerMethod (Controller method)|
        +------------------------------------------+
                |
                | 5. Call HandlerAdapter
                v
        +---------------------------------------------+
        | RequestMappingHandlerAdapter                |
        |---------------------------------------------|
        | - Resolves arguments via ArgumentResolvers  |
        | - Applies Data Binding, @Valid validation   |
        | - Executes @Controller method               |
        | - Handles return value via ReturnValueHandlers|
        +---------------------------------------------+
                |
                | 6. Controller method executes
                v
        +------------------------------------------+
        | @RestController / @Controller Bean       |
        |------------------------------------------|
        | - May call Service Layer (via AOP proxy) |
        | - Service Layer may start @Transactional |
        | - Accesses Repositories, DB, etc.        |
        | - Returns domain or DTO object           |
        +------------------------------------------+
                |
                | 7. Return value converted
                v
        +------------------------------------------+
        | HttpMessageConverter (e.g. Jackson)      |
        |------------------------------------------|
        | Converts Java object -> JSON/response body|
        +------------------------------------------+
                |
                | 8. DispatcherServlet post-handle
                v
        +------------------------------------------+
        | Interceptors postHandle/afterCompletion  |
        +------------------------------------------+
                |
                | 9. Response written to socket
                v
        +--------------------+
        | Embedded Server    |
        | Writes bytes to TCP|
        | Flushes response   |
        +--------------------+
                |
                | 10. HTTP Response sent back
                v
        +--------------------+
        |   Client receives  |
        |   JSON / HTML / etc|
        +--------------------+
    ```

1. Embedded Server (Tomcat/Jetty)

    - Spring Boot auto-configures TomcatServletWebServerFactory.
    - Tomcat starts with N acceptor and worker threads.
    - When HTTP request arrives → accepted socket passed to worker thread.
    - Parses bytes → creates HttpServletRequest & HttpServletResponse.
    - 🔹 Key thread pool: tomcat-exec-<n> (or similar).

2. Filter Chain:
    - All Filter beans registered with @Order or FilterRegistrationBean.
    - Runs sequentially, wrapping the request.

    - Common built-ins:
        |Filter|Purpose|
        |---|---|
        |CharacterEncodingFilter|Sets UTF-8 encoding|
        HiddenHttpMethodFilter|	Enables PUT/DELETE via hidden form field
        CorsFilter|	Handles CORS headers
        Spring Security Filter Chain|	Auth/authz logic
        OncePerRequestFilter|	Prevents double execution

At the end → DispatcherServlet (Spring MVC entry point).

3. DispatcherServlet (Spring MVC Heart): 
Core method: doDispatch(): 
    - Gets handler from HandlerMapping (usually RequestMappingHandlerMapping)
    - Applies HandlerInterceptor.preHandle().
    - Chooses correct HandlerAdapter (for controller invocation).
    - Invokes controller → gets return value.
    - Uses HandlerMethodReturnValueHandler + HttpMessageConverter to write response.
    - Calls postHandle() and afterCompletion() on interceptors.

4. HandlerMapping:

    - Matches incoming request to a controller method based on:
    - Path (@RequestMapping("/api/users"))
    - HTTP method (GET, POST)
    - consumes / produces
    - Params / headers
    - Produces a HandlerExecutionChain (controller + interceptors).

5. HandlerAdapter

    - Responsible for invoking the handler (controller method).
        - Resolves arguments via HandlerMethodArgumentResolvers.
    - e.g., @PathVariable, @RequestBody, @RequestParam, HttpServletRequest, etc.
    - Applies validators for @Valid.
    - Calls controller using reflection.

🧠 6. Controller & Service Layer

When your controller calls a service method:

- The service bean might be wrapped by an AOP proxy (due to @Transactional, @Async, or @Aspect).
- Proxy handles cross-cutting concerns:

    - Begin transaction → execute → commit/rollback.
    - Switch thread (async).
    - Log method calls.

Important: self-calls inside the same bean skip proxy → annotation ignored.

7. Return Value & Response Conversion

    - Return type may be:
        - ResponseEntity<\T> → status + headers + body.
        - String (view name).
        - Object → JSON/XML body via HttpMessageConverter.

    - Spring picks correct converter:
        - MappingJackson2HttpMessageConverter → JSON
        - MappingJackson2XmlHttpMessageConverter → XML

Applies ResponseBodyAdvice if defined.

8. Interceptors Post-Processing 
    - HandlerInterceptor.postHandle() runs before view rendering.
    - afterCompletion() runs after response commit (even if exception occurred).

9. Response Finalization
    - Response written to servlet output stream.
    - Servlet container sends bytes to client socket.
    - Connection reused or closed (depending on keep-alive).

10. Exception Path (If error occurs)
    - If controller or service throws an exception:
        - DispatcherServlet catches it.
        - Passes to registered HandlerExceptionResolvers:
            - @ControllerAdvice / @ExceptionHandler
            - ResponseStatusExceptionResolver
            - DefaultHandlerExceptionResolver

Produces structured JSON or error page (via BasicErrorController).


### Q1. At the OS/network layer, what’s the first thing before Spring even sees the request?
**Ans**:
1. TCP connection (or reuse via keep-alive/HTTP2 multiplex). The OS hands bytes to the JVM process bound to a socket. An embedded server (Tomcat/Jetty/Undertow) accepts the socket, assigns a worker thread from its acceptor/pool, parses HTTP bytes into an HttpServletRequest (for Servlet) or the reactive HTTP server request (for WebFlux), then dispatches to servlet/filter chain
    
2. Note: If worker threads exhaust, requests queue or are dropped → client timeouts.

### Q2. In a Spring Boot (Spring MVC) app with embedded Tomcat, what receives the request first in the JVM?
**Ans:**    The Servlet container (Tomcat). It invokes configured Servlet filters in order, then the DispatcherServlet (mapped to / by default). Spring’s own filters (e.g., Spring Security filter chain) are regular servlet filters and are invoked as part of the chain before the DispatcherServlet.

### Q3: What’s the order: Filters, Interceptors, Controller?
**Ans:**
1. Servlet Filters (Servlet container order + @Order / FilterRegistrationBean ordering).
2. DispatcherServlet entry
3. HandlerMapping finds handler
4. HandlerInterceptor.preHandle(...)
5. Controller method (@Controller/@RestController)
6. HandlerInterceptor.postHandle(...)
7. View rendering / HttpMessageConverters write response
8. HandlerInterceptor.afterCompletion(...)
9. Servlet Filters finally logic (return through chain)

HandlerExceptionResolvers and exception handling can short-circuit this flow

### Q4 How does DispatcherServlet map a request to a controller method?
**Ans:** DispatcherServlet asks HandlerMapping beans (e.g., RequestMappingHandlerMapping) to find the best HandlerExecutionChain for the request URL, HTTP method, headers, params, consumes/produces. It uses RequestMappingInfo which considers path patterns, HTTP verb, content negotiation (consumes/produces) and @RequestMapping attributes.

Gotcha: If produces doesn't match client Accept header, you can get 406 Not Acceptable

### Q5: How are controller method parameters resolved?
**Ans:** HandlerMethodArgumentResolvers inspect method signature and resolve parameters (path vars, query params, request body via @RequestBody, HttpServletRequest, Principal, @RequestHeader, @SessionAttribute, @Valid + BindingResult etc.). Custom resolvers can be registered.

Edge: @RequestBody uses HttpMessageConverter to deserialize; for large bodies the InputStream may be streamed or buffered depending on converter.

### Q6. When @RequestBody fails (malformed JSON), what happens?
**Ans:** The HttpMessageConverter (e.g., Jackson) throws HttpMessageNotReadableException. DispatcherServlet routes this to HandlerExceptionResolvers — Spring MVC’s ResponseStatusExceptionResolver, ExceptionHandlerExceptionResolver (for @ExceptionHandler), and DefaultHandlerExceptionResolver can produce a proper error response (400). @ControllerAdvice can intercept and customize.

### 7 Q: Where are @Controller beans created and when?

**Ans:** By ApplicationContext during context initialization, unless annotated @Lazy. Beans are created and processed by BeanPostProcessors. @PostConstruct runs after dependencies injected. Scopes: default singleton. request scope beans are created per HTTP request.

### 8 Q: How does @Transactional work when you call a service from a controller?

**Ans:** Spring creates a proxy around the transactional bean (JDK proxy or CGLIB). On method entry, the proxy creates/joins a transaction using PlatformTransactionManager according to propagation rules (REQUIRED, REQUIRES_NEW, etc.). On exit, it commits/rolls-back depending on exceptions. Important gotcha: @Transactional on private methods or internal self-invocation won’t work because proxies are bypassed.

### 9 Q: What’s the difference in request handling for Spring MVC vs WebFlux?
**Ans:**
Spring MVC (Servlet): thread-per-request model. The servlet container assigns a thread to the request and it blocks while waiting on IO or DB.

WebFlux (reactive): non-blocking, event-loop style. Uses Netty or reactive wrapper. It uses fewer threads and backpressure; controller returns Mono/Flux. Blocking calls in WebFlux thread can starve the event loop — run them on bounded elastic/scheduler.

### 10 Q: What happens if you send a request that triggers file upload (multipart)?

**Ans:** MultipartResolver parses request before controller (filter or DispatcherServlet uses it). Files are hashed into MultipartFile objects; by default they may be stored in disk temp or in-memory depending on size and config. If you stream upload, ensure controller uses StreamingResponseBody or reactive multipart handling; otherwise entire file may buffer.

### 11 Q: How is response conversion handled?

**Ans:** After controller returns, HandlerAdapter and HandlerMethodReturnValueHandlers decide how to handle return value. For @ResponseBody/ResponseEntity the HttpMessageConverter serializes to bytes (Jackson for JSON). ResponseBodyAdvice can modify the body before writing.

### 12 Q: Where/when are the HTTP headers and status set?

**Ans:** Controller can set status via ResponseEntity, HttpServletResponse#setStatus, or @ResponseStatus. Filters or interceptors (or ResponseBodyAdvice) can alter headers before final write. Container writes network headers when output stream flushes.

### 13 Q: What is OncePerRequestFilter and how does it affect the filter chain?

**Ans:** A base class that ensures filter logic runs once per request lifecycle. Important when request dispatches happen (FORWARD, ERROR). Filters without such guard might run multiple times during internal dispatch.

### 14 Q: How do exception handling and @ControllerAdvice interact with filters/interceptors?

**Ans:** If an exception occurs inside controller, DispatcherServlet calls HandlerExceptionResolvers. @ControllerAdvice with @ExceptionHandler is part of ExceptionHandlerExceptionResolver. Filters wrapping around the request still execute their finally/cleanup code as the chain unwinds — but afterCompletion in interceptors runs even if exception thrown.

### 15 Q: How does Spring Security integrate into request flow?

**Ans:** Spring Security installs a filter chain (a servlet filter) that authenticates/authorizes before DispatcherServlet. It sets SecurityContext (often in SecurityContextHolder thread-local) and sometimes creates session. For stateless JWT, it validates token and sets authentication.

Gotcha: SecurityContextHolder uses ThreadLocal in servlet model—if you spawn new threads (async), you must propagate SecurityContext or lose authentication context.

### 16 Q: What happens when controller starts an @Async method or returns Callable/DeferredResult?

**Ans:**

@Async runs method on task executor thread pool (default SimpleAsyncTaskExecutor or configured). Caller thread returns; response may be completed asynchronously.

Callable/DeferredResult allow async request processing in MVC: request thread is released; container holds connection open while another thread completes result. Interceptor afterCompletion runs later.

Gotcha: Transactional context does not automatically propagate into async threads unless explicitly passed.

### 17 Q: How are interceptors and AOP different? Which runs first?

**Ans:**

HandlerInterceptor is MVC layer — it applies to controller calls after mapping.

AOP (@Aspect) proxies apply around service or controller bean calls depending on pointcuts and proxies. AOP proxies wrap the bean — so their execution order relative to interceptors depends: interceptors wrap controller invocation inside the DispatcherServlet; AOP proxies are on the bean. In practice: servlet filter → DispatcherServlet → interceptors → AOP proxy around controller/service methods.

### 18 Q: How does the request scope work and when is it created/destroyed?

**Ans:** Request scope (@Scope("request")) is created when the dispatcher receives the request and bound to the current thread for the request lifetime. Destroyed at request completion. In async scenarios, special handling is required to propagate scope into the async thread.

### 19 Q: How are static resources served?

**Ans:** ResourceHttpRequestHandler handles classpath or file resources (from /static, /public). Static resources are resolved before controller mapping by handler mappings. Resource chain supports caching, fingerprinting, gzipped resource resolution. If missing, container may fallback to index.html if SPA configured.

### 20 Q: What causes HTTP 415 Unsupported Media Type?

**Ans:** Client's Content-Type doesn't match any consumes defined on controller, or no HttpMessageConverter can read the body for that Content-Type. Example: sending text/plain to an endpoint expecting JSON.

### 21 Q: Explain HandlerMethod execution when controller method returns ResponseEntity<\StreamingResponseBody>.

**Ans:** Spring invokes the method, gets StreamingResponseBody which is a callback that writes to OutputStream. The container may do streaming chunked transfer; the worker thread remains engaged while streaming (or can be run on an async thread depending on config).

### 22 Q: What happens under the hood for CORS preflight?

**Ans:** Browser sends OPTIONS request with Access-Control-Request-Method header. Spring's CorsFilter or MVC CORS config responds with appropriate Access-Control-Allow-* headers (if configured). If missing → browser blocks request client-side.

### 23 Q: How do @ControllerAdvice and ResponseEntityExceptionHandler affect error responses?

**Ans:** @ControllerAdvice with @ExceptionHandler hooks into ExceptionHandlerExceptionResolver to transform exceptions into ResponseEntity/body. If none matches, default resolvers produce status codes. You can intercept and return custom JSON error payloads.

### 24 Q: What subtlety with proxies and final methods can bite you?

**Ans:** JDK proxies only proxy interfaces — class proxies via CGLIB are used when needed. CGLIB cannot proxy final methods. If your transactional/security annotations are on final methods, proxies won’t intercept them -> annotations ignored.

### 25 Q: How does ContentNegotiation work?

**Ans:** Spring consults request headers (Accept), path extension (deprecated by default), and query parameters (if configured) to choose a media type. It then picks an HttpMessageConverter supporting that media type.

### 26 Q: What about request/response buffering and memory concerns?

**Ans:** Many converters buffer into memory (e.g., MappingJackson2HttpMessageConverter). Large payloads or many concurrent requests can OOM. Use streaming APIs (StreamingResponseBody, Flux), or configure multipart to stream to disk, and set limits (spring.servlet.multipart.max-file-size).

### 27 Q: How are session and cookies created and used?

**Ans:** HttpSession is created lazily when request.getSession(true) is called or when security session needed. JSESSIONID cookie stores session id. Spring Session can externalize session to Redis for clustering. Session replication or use of sticky sessions affects scaling.

### 28 Q: What happens when controller returns void?

**Ans:** Behavior depends: no @ResponseBody: view name inferred from request mapping; with @ResponseBody/@RestController void means no body — status 200 unless modified. HttpServletResponse can be manipulated directly.

### 29 Q: How does @Valid validation interact with argument resolution?

**Ans:** @Valid triggers JSR-303 validation during argument resolution; validation errors populate BindingResult. If BindingResult not present and exception raised, MethodArgumentNotValidException is thrown and can be handled by @ExceptionHandler.

### 30 Q: Where do logs (MDC) and thread locals break in async/reactive flows?

**Ans:** MDC and other ThreadLocals are tied to threads. In async tasks or reactive threads, MDC is not automatically copied. Spring provides Executor wrappers or Reactor context features to propagate context. Not propagating MDC breaks request ID logging across threads.

### 31 Q: What can cause double invocation of filters/servlets for the same request?

**Ans:** Using FORWARD/INCLUDE dispatches, RequestDispatcher.forward or internal error dispatch may cause the same filter mappings to trigger again unless filters guard against multiple invocations (OncePerRequestFilter).

### 32 Q: What’s the lifecycle of a bean with @PostConstruct and BeanPostProcessors during startup?

**Ans:** After dependencies injected, BeanPostProcessor.postProcessBeforeInitialization runs, then @PostConstruct/afterPropertiesSet, then postProcessAfterInitialization. Proxies may be created in post processing.

### 33 Q: How does connection keep-alive and chunked response affect the app under high load?

**Ans:** Keep-alive reduces TCP handshake cost but ties socket resources longer. Chunked streaming means the container keeps the connection open while writing; many long-lived streams can exhaust server threads. Use HTTP2 properly or tune thread pool and timeouts.

### 34 Q: What happens when a filter throws an exception?

**Ans:** The filter chain is broken. The filter or container should translate to an HTTP response; otherwise container default handles error (500). If exception occurs after partial writes, client may see truncated response.

### 35 Q: How does Spring Boot auto-configuration decide to register DispatcherServlet and default error handling?

**Ans:** Auto-configuration classes (e.g., WebMvcAutoConfiguration, DispatcherServletAutoConfiguration) create beans based on classpath and properties. Default error handling uses BasicErrorController which exposes /error and formats errors using ErrorAttributes.