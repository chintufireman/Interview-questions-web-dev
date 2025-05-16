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

### JWT Token Structure
**Answer**

1. Header: Tells what algorithm is used to sign the token
    ```
    {
    "alg": "HS256",
    "typ": "JWT"
    }

    ```
    Encoded Base64 → 1st part of JWT
2. Payload: Contains the claims (data)
    
    ```
        {
            "sub": "1234567890",
            "name": "John Doe",
            "role": "ADMIN",
            "iat": 1715840000,
            "exp": 1715843600
        }
    ```
    Encoded Base64 → 2nd part of JWT

3. Signature: Ensures the token is not tampered with

### Spring Reactive vs Spring Cloud

**Answer**:
1. table:- 
    |Feature|Spring Reactive|Spring Cloud|
    |---|---|---|
    |Purpose|Build non-blocking, reactive applications|Build and manage distributed microservices|
    |Core Module|Spring WebFluxMultiple: Eureka, Config, Gateway, Sleuth, etc|
    |Programming Style|Reactive, event-driven|Mostly imperative (but supports reactive via WebFlux)|
    |Concurrency Handling|Uses Reactor (Mono, Flux), no thread blocking|Depends on underlying modules (blocking or reactive)|
    |Use Case|High-load APIs, streaming, real-time apps|Service discovery, config management, resiliency, tracing|
    |Key Tools|WebFlux, Reactor|Eureka, Gateway, Config Server, Hystrix, Zipkin|
    |Built On|Project Reactor|Spring Boot + Netflix OSS or newer replacements|
    |Focus Area|App behavior and responsiveness|System-level infrastructure for microservices|

### Spring Cloud: Overview & Architecture

**answer**: Spring Cloud is a set of tools and libraries for building cloud-native microservices.
 
 1. Key Components of Spring Cloud.
    |Component|Description|
    |---|---|
    |Spring Cloud Config|Centralized configuration management (reads from Git, file, etc.)|
    |Eureka|Service Registry for service discovery|
    |Spring Cloud Gateway|API Gateway for routing, rate-limiting, filtering|
    |Spring Cloud LoadBalancer|Client-side load balancing (alternative to Netflix Ribbon)|
    |Spring Cloud OpenFeign|Declarative REST client, integrates with service discovery|
    |Spring Cloud Sleuth|Distributed tracing with unique request IDs|
    |Spring Cloud Zipkin|Works with Sleuth for trace visualization|
    |Spring Cloud Bus|Broadcast configuration updates using messaging (e.g., Kafka, RabbitMQ)|
    |Resilience4j|Circuit breaker and fault tolerance (replaces Hystrix)|

2. Typical Flow
    - Startup: Services register with Eureka Server
    - Discovery: Feign clients or Gateway resolve services via Eureka.
    - Config: Services pull config from Spring Cloud Config Server.
    - Routing: Gateway routes incoming requests to the right service.
    - Tracing: Sleuth generates trace IDs, and Zipkin collects them.
    - Resilience: Resilience4j provides retries, fallback, and circuit breakers.

3. Use Cases Solved by Spring Cloud
    - Decentralized service registration & discovery
    - Centralized configuration
    - Load balancing and routing
    - Distributed tracing
    - Circuit breaking and fallback
    - API gateway management

### NFR – Non-Functional Requirements

**Answer**: Non-Functional Requirements (NFRs) define how a system performs its tasks, rather than what it does (which is handled by Functional Requirements).

1. Common Types of NFRs

|NFR Type|Description|
|---|---|
|Performance|Response time, throughput, latency, resource usage|
|Scalability|Ability to handle growth in load or users|
|Availability|System uptime (e.g., 99.9% SLA)|
|Reliability|System consistency over time; failure recovery|
Security|Authentication, authorization, data encryption, vulnerability protection
Maintainability|Ease of fixing issues, upgrading, or modifying the system
Usability|User interface clarity, user-friendliness
Portability|Ability to run on different platforms/environments
Compliance|Adherence to legal, regulatory, or industry standards
Disaster Recovery|Backup, failover, and recovery plans in case of system failure

2. Example: If you're designing a REST API for a banking app
    - Functional requirement: User can transfer money
    - Non-functional requirement: The API must respond in < 200ms, and must be available 99.99% of the time, with TLS encryption.

3. Non-Functional Requirements (NFR) Template:
    Category|Requirement|Metric / Target
    ---|---|---
    Performance|API should respond within acceptable time|≤ 200ms for 95% of requests
    Scalability|System must handle increased load without performance degradation|Scale to 10x user traffic
    Availability|System should be available round the clock|99.99% uptime (≈ 5 minutes/month downtime)
    Reliability|System must recover from failures automatically|Retry mechanism, auto failover in place
    Security|Data must be secure in transit and at rest|Use HTTPS + AES-256 encryption
    Maintainability|Easy to deploy updates without downtime|Blue-Green Deployment / CI/CD enabled
    Observability|System should be monitorable and traceable|Logs, metrics, traces via Prometheus/Grafana
    Usability|UI should be simple and intuitive|90% of users should complete tasks in < 3 steps
    Portability|System should run on multiple environments|Dockerized, cloud-agnostic design
    Compliance|Meet industry or legal standards|GDPR, HIPAA, PCI-DSS as per domain
    Disaster Recovery|System should recover from disaster quickly|RTO: 30 mins, RPO: 15 mins

4. Definitions:
    - RTO (Recovery Time Objective): Max time to restore system after failure.
    - RPO (Recovery Point Objective): Max acceptable data loss window.

### webclient vs resttemplate
**Answer**
1. RestTemplate Example (Blocking)

    ```
    RestTemplate restTemplate = new RestTemplate();
    String response = restTemplate.getForObject("http://example.com/api", String.class);
    System.out.println(response);
    ```
2. WebClient Example (Non-blocking)

    ```
        WebClient webClient = WebClient.create();
        webClient.get()
        .uri("http://example.com/api")
        .retrieve()
        .bodyToMono(String.class)
        .subscribe(System.out::println);
    ```

### how do we test our microservices
**Answer**
1. Unit Testing:
    - Test individual components (services, controllers, repositories)
    - Use frameworks like JUnit, Mockito for mocking dependencies
    - Fast and isolated tests

2. Integration Testing:
    - Test interaction between components within a microservice
    - Test DB access, API endpoints, message brokers, etc
    - Use Spring Boot Test with in-memory databases (H2), or embedded Kafka/RabbitMQ.
    - Use TestRestTemplate or WebTestClient to test REST endpoints.

3. Contract Testing
    - Ensure service interfaces between microservices match expectations.
    - Tools: Pact, Spring Cloud Contract.
    - Useful for consumer-driven contracts to avoid breaking changes.

4. End-to-End (E2E) Testing
    - Test full workflows across multiple microservices
    - Usually done in a staging environment
    - Tools: Cypress, Selenium, Postman/Newman

5. Performance Testing
    - Test service behavior under load
    - Tools: JMeter, Gatling, Locust

6. Chaos Testing
    - Introduce failures to test resilience
    - Tools: Chaos Monkey or custom fault injection

7. API Testing
    - Test REST APIs for correctness and error handling
    - Tools: Postman, Swagger UI, RestAssured.

### Quality Gates – in CI/CD & Code Quality

**Answer**: Quality Gates are a set of rules or conditions that code must meet before it can be merged, deployed, or released. They ensure your codebase maintains high quality and reliability.

1. Common Quality Gate Conditions:

Check|Purpose
---|---
No Critical or Blocker Bugs|Avoid severe defects in production
Coverage Threshold (e.g., 80%)|Ensure enough unit test coverage
No Vulnerabilities|Avoid known security issues (e.g., via SonarQube)
No Code Smells|Improve maintainability and readability
Low Technical Debt|Keep refactoring cost under control
All Tests Pass|Ensure code correctness
Static Code Analysis|Enforce coding standards, e.g., naming
No TODOs or commented code|Clean production-ready code

2. Tools That Support Quality Gates
    - SonarQube: Most popular for quality gates + reports
    - GitHub Actions + linters: Code style, lint, test checks
    - Jenkins + SonarQube: CI pipeline integration for quality gates
    - Azure DevOps / GitLab CI: Built-in support for gates

3. Example: Fail if:
    - Bugs > 0
    - Coverage < 80%
    - Duplicated Lines > 3%
    - Vulnerabilities > 0

### integration testing how u do

**Answer**: Integration testing checks whether different parts of a service (controllers, services, DB, external APIs, message brokers) work together as expected.

1. Framework:
    - @SpringBootTest – loads full application context.
    - @AutoConfigureMockMvc or @WebTestClient for HTTP calls.

2. Test Tools:
    - In-memory DB: H2 or TestContainers for PostgreSQL/MySQL.
    - Mock external APIs: WireMock.
    - Embedded Kafka/RabbitMQ (if needed)
3. Sample Test (REST + DB):
    ```
        @SpringBootTest
        @AutoConfigureMockMvc
        class UserControllerIntegrationTest {

            @Autowired
            private MockMvc mockMvc;

            @Autowired
            private UserRepository userRepository;

            @BeforeEach
            void setUp() {
                userRepository.save(new User("John", "john@mail.com"));
            }

            @Test
            void testGetUser() throws Exception {
                mockMvc.perform(get("/users/1"))
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.name").value("John"));
            }
        }

    ```

4. Sample Test (WebClient-based service):
    ```
        @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
        class WebClientIntegrationTest {

            @Autowired
            private WebTestClient webTestClient;

            @Test
            void testGetEndpoint() {
                webTestClient.get().uri("/api/data")
                    .exchange()
                    .expectStatus().isOk()
                    .expectBody()
                    .jsonPath("$.id").isEqualTo(1);
            }
        }
    ```
5. What I Validate
    - REST endpoint + DB flow
    - Response structure
    - HTTP status codes
    - Error handling (404, 500).
    - Interaction with Kafka, Redis, external services (mocked)

### how u resolve merge conflict in bitbuket
**Answer**: It happens when two branches have changes to the same part of the same file, and Git doesn’t know which one to keep.

1. Fetch and Checkout the Branch: Start by fetching the latest code and checking out the branch you want to merge into (usually main or develop):
    ```
        git fetch origin
        git checkout main
        git pull origin main
    ```
2. Merge the Feature Branch: Try merging the feature branch (e.g., feature-branch) into main.
    ```
        Try merging the feature branch (e.g., feature-branch) into main
    ```
    - If there's a conflict, Git will alert you:
        ```
            Auto-merging filename
            CONFLICT (content): Merge conflict in filename
            Automatic merge failed; fix conflicts and then commit the result.
        ```
3. Resolve the Conflict: Open the conflicted file. You'll see conflict markers like
    ```
        <<<<<<< HEAD
        Current branch changes
        =======
        Incoming changes from feature-branch
        >>>>>>> feature-branch
    ```
4. Mark File as Resolved: `git add filename`

5. Commit the Merge: git commit.

6. Push the Changes to Bitbucket: git push origin main
