#### mark and sweep algorithm

**answer**: The Mark and Sweep algorithm is a fundamental garbage collection technique.

1. Overview: The algorithm works in two main phases
    - Mark phase
    - Sweep phase

2. Mark Phase:
    - The garbage collector starts from a set of root objects (like global variables, stack references).
    - It traverses all reachable objects recursively from these roots.
    - Each reachable object is "marked" (e.g., with a boolean flag) to indicate it's still in use.
    - Think of it like this: You’re tracing a family tree from you (the root) and putting a ✔️ on everyone you can reach through relations.

3. Sweep Phase:
    - The algorithm goes through the entire heap
    - If an object is not marked, it means it's unreachable → garbage.
    - These unreachable objects are deallocated, freeing up memory.
    - All marked objects are then unmarked for the next cycle

#### jvm configuration
**Answer**: Sure! JVM (Java Virtual Machine) configuration refers to tuning the behavior of the JVM using options and parameters at runtime.

1. Memory Management Options: Control how much memory is allocated to the JVM

|Option|Description|
|---|---|
|Xms<size>|Initial heap size (e.g., -Xms512m)|
|Xmx<size>|Maximum heap size (e.g., -Xmx2g)|
|Xmn<size>|Size of the Young Generation|
|XX:MaxPermSize|(Java 7 and earlier) Max size for PermGen|
|XX:MetaspaceSize|(Java 8+) Initial size for Metaspace|

2. Garbage Collection Options

|Option|Description|
|---|---|
|XX:+UseG1GC|Garbage-First GC (default in Java 9+)|
|XX:+UseZGC / -XX:+UseShenandoahGC|Low-pause-time collectors (Java 11+)|
|XX:+UseConcMarkSweepGC|Low-latency, concurrent collector (deprecated)|

3. Thread and Stack Settings

|Option|Description|
|---|---|
|Xss<size>|Thread stack size (e.g., -Xss512k)|
|XX:ParallelGCThreads=<n>|Number of threads for GC|

4. Performance and Debugging

|Option|Description|
|---|---|
|XX:+PrintGC|Print GC events|

5. Class Loading and JIT

|Option|Description|
|---|---|
|XX:+TieredCompilation|Use both client and server JIT compilers|

#### Garbage collector types
**Ans**:

|GC Type|Focus|Pause Time|Parallelism|Notes|
|---|---|---|---|---|
|Serial|Simplicity|Long|Single thread|for small ops|
|Parallel|Throughput|Medium-Long|Multi-thread|Good for batch jobs|
|G1|Balanced|Short|yes|Default since Java 9|
|ZGC|Ultra-low pause|Very Short (<10ms)|yes|Large heap, new-gen GC|

#### Fail-Fast vs Fail-Safe Iterators in Java
**Ans**:

1. Fail-Fast Iterator
    - Definition: Throws a ConcurrentModificationException if the collection is structurally modified after the iterator is created (except through the iterator’s own methods).
    - Collections: Found in most Java Collection Framework classes like ArrayList, HashSet, HashMap, etc.
    - Detection Mechanism: Uses a modification count (modCount) internally. If there's a mismatch during iteration, it fails fast.
    - not thread safe

2. Fail-Safe Iterator
    - Definition: Does not throw an exception if the collection is modified during iteration
    - Collections: Found in concurrent collections like CopyOnWriteArrayList, ConcurrentHashMap
    - Detection Mechanism: Iterates over a clone or snapshot of the collection, so original changes don’t affect it.

#### ThreadLocal in Java
**ans**: ThreadLocal provides thread-local variables. Each thread accessing it gets its own, independent copy of the variable.

1. key points:
    - Useful for storing data per thread, like user sessions, DB connections, etc.
    - Not shared between threads
    - Stored in a Thread’s own memory (Thread.threadLocals).

2. Common Methods
```
ThreadLocal<String> tl = new ThreadLocal<>();

tl.set("A");       // Sets value for current thread
String val = tl.get(); // Gets value for current thread
tl.remove();       // Removes value for current thread
```

3. use case

```
public class Example {
    private static ThreadLocal<Integer> userId = new ThreadLocal<>();

    public static void main(String[] args) {
        Runnable task = () -> {
            userId.set((int)(Math.random() * 100));
            System.out.println(Thread.currentThread().getName() + " => " + userId.get());
        };

        new Thread(task).start();
        new Thread(task).start();
    }
}

```
4. When to Use
    - Avoid sharing state between threads
    - Store per-thread context like request ID, locale, security credentials

### API gateway architecture
**Ans**: A single entry point that sits between clients and backend services in a microservices architecture

1. Architecture Components
    - Client: Sends API requests (e.g., web/mobile app).
    - API Gateway: Receives all client requests
    - Microservices: Perform actual business logic

|Responsibility|Description|
|---|---|
|Routing|Routes requests to the correct microservice|
|Authentication|Validates tokens (e.g., JWT)|
|Rate Limiting|Prevents abuse by limiting requests|
|Caching|Stores frequent responses to reduce load|
|Load Balancing|Distributes traffic across service instances|
|Request Transformation|Modifies headers, paths, payloads|
|Aggregation|Combines responses from multiple services|
|Monitoring/Logging|Logs API usage, errors, etc|    

### service discovery vs eureca server
**ans**:
1. Service Discovery (Concept)
    - Definition: Mechanism by which microservices automatically find each other without hardcoding IPs/ports.
    - Types:
        - Client-side discovery (e.g., Netflix Eureka)
        - Server-side discovery (e.g., AWS ELB)
2. Eureka Server (Tool)
    - Netflix OSS tool for client-side service discovery.
    - Acts as a registry where services register themselves.
    - Clients query Eureka to find service locations.

    - How Eureka Works
        - Microservice registers with Eureka.
        - Eureka maintains a registry of all services.
        - Clients query Eureka to get the instance of a target service.
        - Load balancing is handled on the client side.

### method level security vs class level
**answer**:

1. Class-Level Security
    - Applied on the entire class
    - Affects all methods in the class
    - Declared using annotations like @PreAuthorize, @RolesAllowed, etc

2. Method-Level Security
    - Applied on individual methods
    - Allows fine-grained control per method
        ```
            @PreAuthorize("hasRole('ADMIN')")
            public String getData() { ... }
        ```
### @apioperation for description in api
**Answer**:

1. @ApiOperation is used to describe an API endpoint in Swagger UI.
2. Comes from io.swagger.annotations (used in SpringFox for Swagger 2)

```
    @ApiOperation(value = "Create User", notes = "Creates a new user and returns it", response = User.class)
    @PostMapping("/user")
    public User createUser(@RequestBody User user) {
        ...
    }
```

### rest api  what need to take care
**Answer**

1. Use Proper HTTP Methods:- GET, POST, PUT, PATCH, DELETE
2. Use Meaningful Resource URIs
3. Status code:

    |Code|Meaning|
    |---|---|
    |200|OK|
    |201|Created|
    |204|No content|
    |400|Bad request|
    |401|Unauthorized|
    |404|Not found|
    |500|Internal server error|

4. Input Validation
    - Validate request body, params, headers
    - Return helpful error messages

5. Security
    - Use HTTPS
    - Apply authentication (e.g., JWT, OAuth)
    - Authorize sensitive endpoints
6. Versioning
    - Use URI or header versioning:
        - /api/v1/users
        - Header: Accept: application/vnd.company.v1+json

7. Consistent Response Format

```
{
  "status": "success",
  "data": { ... },
  "message": "User created successfully"
}
```
8. Error Handling

9. Pagination & Filtering
    - Support query params: ?page=1&size=10&sort=name

10. Logging and Monitoring
    - Log requests and responses
    - Monitor API usage and failures

### how u do monitoring using promethus and grafana

**answer**
1. Add Dependencies

    ```
    <!-- Micrometer + Prometheus -->
    <dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Spring Boot Actuator -->
    <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    ```
2. Configure application.properties
    ```
    management.endpoints.web.exposure.include=health,info,prometheus
    management.endpoint.prometheus.enabled=true
    ```
3. Run Spring Boot App: Prometheus metrics available at
    - `http://localhost:8080/actuator/prometheus`

4. Setup Prometheus
    - Install Prometheus and update prometheus.yml:
    ```
        scrape_configs:
        - job_name: 'spring-boot-app'
        metrics_path: '/actuator/prometheus'
        static_configs:
        - targets: ['localhost:8080']
     ```
5. Setup Grafana:
    - Install Grafana
    - Add Prometheus as a data source

### @Rollback Annotation

**Answer**:
1. Used in JUnit tests to automatically rollback database changes after a test method runs. Ensures test data does not persist in the DB.
2. How it works:
    - @Transactional starts a transaction for the test.
    - @Rollback(true) rolls it back after the test finishes.

3. @Rollback is mainly for test cleanup, ensuring tests remain isolated and repeatable

### @TestSuite Annotation
**Answer**:
1. In JUnit 5, you use the @Suite annotation from org.junit.platform.suite.api.
2. To group multiple test classes and run them together as a test suite.
3. JUnit 4 had a @RunWith(Suite.class) approach.

    ```
    <dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-suite</artifactId>
    <version>1.9.3</version> <!-- or latest -->
    </dependency>

    ```
    ```
    import org.junit.platform.suite.api.SelectClasses;
    import org.junit.platform.suite.api.Suite;

    @Suite
    @SelectClasses({
        UserServiceTest.class,
        OrderServiceTest.class
    })
    public class ApplicationTestSuite {
        // No code needed. This runs all specified tests.
    }

    ```
4. Run all related tests together
5. Useful for grouping unit tests, integration tests, etc

### Security Policies in API Gateway
**Answer**:
1. Authentication:
    - Verify identity of the client
    - Supports:
        - OAuth2 / JWT
        - API Keys
        - Basic Auth
    - Example: Validate JWT token in request header
2. Authorization:
    - Check if the authenticated user has permission to access a resource
    - Role-based or scope-based
    - Example: Allow only users with ADMIN role to access /admin/**
3. Rate Limiting & Throttling:
    - Restrict the number of requests per user/IP/time
    - Protects from DDoS attacks or abuse
    - Example: Max 100 requests per minute per IP.
4. IP Whitelisting / Blacklisting
    - Allow or block requests from specific IP addresses
5. Request Validation
    - Validate headers, parameters, body formats before routing
6. HTTPS Enforcement
    - Redirect HTTP to HTTPS
    - Ensures secure communication with SSL/TLS
7. CORS Policy
    - Define which origins are allowed to access APIs
    - Prevents Cross-Origin Request Forgery (CSRF) risks
8. Logging & Auditing
    - Log every incoming request with metadata
9. Encryption
    - Encrypt sensitive headers or payloads at the gateway before forwarding
10. Custom Policy Plugins
    - Some gateways (like Kong, Apigee, AWS API Gateway) allow you to write custom plugins/policies for advanced security.


### Security Policies in AWS API Gateway
**Answer**

1. Authentication & Authorization

|Mechanism|Description|
|---|---|
|IAM-based|Use AWS IAM roles/policies to control who can call the API|
|Lambda Authorizers|Custom auth logic using Lambda (e.g., verify custom tokens, headers).|
|API Keys|Simple method to identify callers (used with usage plans).|

2. Throttling & Rate Limiting
    - Usage Plans: Define how many requests per second/month are allowed
    - Protects backend from overloading or abuse

3. Resource Policies
    - Define who (which AWS principals or IPs) can access an API.
    - Example: Restrict API to specific IP ranges or VPCs.

4. CORS (Cross-Origin Resource Sharing)
    - Controls access from different domains
    - Must be explicitly enabled for frontends running on separate origins

5. Logging & Monitoring
    - Integrated with CloudWatch:
        - Request logs
        - Error logs
        - Metrics like latency, 4xx/5xx rates
6. Encryption
    - Enforce HTTPS-only access
    - Backend integration with VPC Link for private API access

7. WAF (Web Application Firewall)
    - Attach AWS WAF to API Gateway to block:
        - SQL Injection
        - XSS
        - IP blocks/rate limits
        - Bad bots
8. conclusion: AWS API Gateway offers a layered security model using IAM, Cognito, Lambda authorizers, API keys, WAF, and resource policies to secure APIs at every level.

### How to handle exception in kafka

**Answer:**

1. Kafka Listener Exception Handling:
    - When using @KafkaListener, handle exceptions using
    - ErrorHandler (single record):
        ```
            @Bean
            public ErrorHandler errorHandler() {
                return (thrownException, data) -> {
                    // log or take action
                    System.out.println("Exception: " + thrownException.getMessage());
                    System.out.println("Failed record: " + data);
                };
            }
        ```
        Attach to container:
        
            ```
                @Bean
                    public ConcurrentKafkaListenerContainerFactory<?, ?> kafkaListenerContainerFactory() {
                    ConcurrentKafkaListenerContainerFactory<?, ?> factory = new ConcurrentKafkaListenerContainerFactory<>();
                    factory.setConsumerFactory(consumerFactory());
                    factory.setErrorHandler(errorHandler());
                    return factory;
                }
            ```
        - SeekToCurrentErrorHandler (default)
            - Retries the message a few times.
            - After that, skips or routes to Dead Letter Topic (DLT).
                
                ```
                    factory.setErrorHandler(new SeekToCurrentErrorHandler());
                ```
        - Dead Letter Publishing: Send failed messages to a Dead Letter Topic
            ```
                @Bean
                public DeadLetterPublishingRecoverer recoverer(KafkaTemplate<?, ?> template) {
                    return new DeadLetterPublishingRecoverer(template);
                }
            ```
            Attach it:
                ```
                factory.setErrorHandler(new SeekToCurrentErrorHandler(recoverer, new FixedBackOff(1000L, 3)));
                ```
    - Logging & Monitoring
        - Always log exceptions with message key and offset
        - Use tools like ELK, Prometheus + Grafana to track failures