### Architecture & Microservices Design

#### Q1. How did you design your microservice boundaries?
**Answer**
1. In our networking project, we followed Domain-Driven Design (DDD) principles to define microservice boundaries.
2. Each microservice was responsible for a specific bounded context within the ISP domain.
For example:
    - The Customer Service handled customer onboarding, profile updates, and status changes.
    - The Subscription Service managed service plans, activations, and renewals.
    - The IP Management Service was responsible for assigning and managing IP addresses.
    - The Provisioning Service dealt with backend activation of routers or edge devices.
3. Each service had its own database, exposed REST APIs, and communicated with others via Kafka (for events) and Feign/WebClient (for synchronous calls).
4. This separation allowed each service to evolve independently, scale as per load, and be maintained by focused teams.
5. For shared logic (like common DTOs or error responses), we used shared libraries, but core business logic remained encapsulated.

6. Bonus (mention if asked about rationale):
    - We grouped functionality based on:
        - Business ownership (what team owns what responsibility)
        - Data lifecycle (who creates/modifies data)
        - Change frequency (services that change often are isolated)
        - Failure boundaries (failures in one service shouldn’t cascade)

#### Q2. How did your services communicate with each other — synchronously or asynchronously? And why?
**Answer:** In our microservices-based networking project, we used a hybrid communication strategy — both synchronous (REST) and asynchronous (Kafka) — based on the use case and dependency.

1. Synchronous Communication (REST via FeignClient / WebClient)
    - We used sync communication when:
        - The caller needed an immediate response or confirmation
        - The services were tightly coupled in a request-response flow
    - Examples:
        - CustomerService calling SubscriptionService to verify plan eligibility.
        - TroubleshootingService calling IPService via WebClient to fetch real-time IP details.
    - This allowed:
        - Quick data retrieval
        - Simpler request/response logic
        - Easier exception handling (timeouts, fallbacks)

2. Asynchronous Communication (via Kafka):
    - We used Kafka when:
        - Services needed to be decoupled
        - Events were domain-driven (e.g., "something happened")
        - We wanted high resilience and loose coupling
    - Examples:
        - CustomerService publishes a CustomerCreated event
        - SubscriptionService listens and provisions a subscription
        - IPService listens to SubscriptionCreated to assign an IP
    - Benefits:
        - Services could fail/retry independently
        - Horizontal scalability via topic partitioning
        - Easier to add new consumers (e.g., AuditService)
3. When & Why Hybrid Strategy Works:
    - Using sync + async allowed us to balance responsiveness and reliability.
    - Critical user-facing workflows used sync calls, while background workflows and data propagation used Kafka events.

#### Q3. What happens if one of your services is down — how does the system behave?\
**Answer:** In our system, if one of the microservices went down, the impact depended on whether it was part of a synchronous call chain or asynchronous (event-driven) flow.

1. If a Sync (REST) Dependency is Down:
    - Example: If TroubleshootingService calls IPService using WebClient and IPService is down.
    - Behavior:
        - WebClient will timeout or return a 5xx error.
        - We handled this using:
            - Retry logic (using RetryTemplate or Spring Retry)
            - Timeout settings (to fail fast)
            - Fallback logic (return a default response or "Service temporarily unavailable")
        - We logged errors and triggered alerts (via Slack/Prometheus/Grafana)
        - This ensured that upstream services didn’t crash even if downstream was unavailable.

2. If an Async (Kafka) Consumer is Down:
    - Example: If IPAssignmentService is down and a SubscriptionCreated event is published.
    - Behavior:
        - Kafka retains the message in the topic (e.g., for 7 days).
        - Once the consumer (IPAssignmentService) is back up, it will resume from the last committed offset.
        - No messages are lost — we only delay processing.
        - This shows decoupling and resilience of event-driven design.

3. How We Detected & Monitored Failures:
    - We monitored:
        - Consumer lag (Kafka)
        - Service health (via /actuator/health)
        - API error rates
    - We configured alerts using Prometheus + Grafana dashboards
    - Logs were centralized using ELK stack

4. Example of a Self-Healing Mechanism: If IP assignment fails 3 times, we publish an IPAssignmentFailed event, and a compensating service (e.g., SubscriptionService) rolls back the subscription — as part of the Saga compensation logic.

#### Q4. How did you maintain data consistency across services?
**Answer:**
Since each of our microservices had its own database, we followed eventual consistency instead of distributed transactions.
We used Kafka and the Outbox Pattern to ensure that state changes were eventually reflected across all services in a reliable, consistent way.

1. Kafka for Event-Driven Updates:
    - When a service updated its local DB (e.g., CustomerService creating a customer), it emitted an event like CustomerCreated via Kafka.
    - Other services like SubscriptionService or IPAssignmentService listened to that event and updated their own data accordingly.
    - This allowed decoupling and asynchronous propagation of changes

2. Outbox Pattern for Reliable Event Publishing: We used the Outbox Pattern to ensure that events were published only if the DB transaction succeeded.
    - How it worked:
        - We wrote the event to an outbox_events table in the same DB transaction as the main business change.
        - A background service or scheduler read from this table and published the event to Kafka.
        - This ensured atomicity between DB change and event publication.

3. Idempotency in Event Handlers:
    - Each service made sure to process each event only once, even if Kafka redelivered it.
    - We used idempotent logic, often checking the event ID or transaction ID before applying any business update.
    - This prevented duplicate state changes in case of retries.

4. Compensating Actions via Saga Pattern:
    - If a flow failed midway (e.g., IP couldn't be assigned), we emitted a failure event like IPAssignmentFailed.
    - Upstream services like SubscriptionService listened to this and performed a rollback (e.g., canceling the subscription).
    - This kept eventual data correctness intact.

5. Optional Realistic Scenario:
    - If CustomerService created a new user and published CustomerCreated, but SubscriptionService was down — the Kafka event remained in the topic.
    - Once SubscriptionService came back up, it processed the backlog and updated its DB to reflect the correct customer state.

6. Final Summary for Interview:
    - We ensured data consistency using Kafka-based event propagation, the Outbox Pattern for atomic updates + event emission, and idempotent consumers for reliability.
    - For failure handling, we used Saga choreography with compensating events to maintain business correctness across services.

#### Q5. How did you implement the Saga pattern and when was a rollback triggered?
**Answer:** We implemented a choreography-based Saga pattern in our networking project to handle multi-step operations like customer activation and provisioning, which span across multiple microservices (Customer, Subscription, IP, Provisioning).

1. Choreography-Based Saga – Why?
    - We chose choreography because:
        - We didn’t want a central orchestrator service.
        - Each service could independently participate in the saga by listening to events and publishing the next event.
    - This made our architecture more loosely coupled and scalable.

2. How the Saga Flow Worked (Happy Path): 
    - Let’s say a new customer is being onboarded:
        1. CustomerService creates the customer → emits CustomerCreated.
        2. SubscriptionService listens → creates subscription → emits SubscriptionCreated.
        3. IPAssignmentService listens → assigns IP → emits IPAssigned.
        4. ProvisioningService listens → provisions router → emits Provisioned.
    - Each service only performs its local transaction, then publishes the next event.

3. When & How Rollback Happened:
    - If any service in the saga failed, we published a failure event to trigger rollback:
        - Example:
            - If IPAssignmentService fails to assign an IP:
                1. It emits IPAssignmentFailed
                2. SubscriptionService listens and triggers CancelSubscription
                3. CustomerService (optionally) listens to roll back the onboarding state
    - This is called a compensating transaction, handled asynchronously via Kafka

4. Idempotency and Ordering:
    - All compensating actions were idempotent to avoid issues in reprocessing.
    - We used Kafka keys (like customerId) to ensure related events landed in the same partition to preserve order.

5. Final Summary Line:
    - We implemented a choreography-based Saga pattern using Kafka for event flow
    - Each service handled its step and published success or failure events
    - In case of failure, compensating transactions were triggered via failure events, allowing rollback of the previous steps in the flow.

#### Q6. How did you pass authentication tokens when using FeignClient or WebClient?
**Answer:** In our Kafka-based microservice setup, we implemented automatic retry and Dead Letter Topic (DLT) handling using Spring Kafka’s @RetryableTopic annotation. This allowed us to retry transient failures a few times, and then divert the message to a DLT if it still failed.

1. RetryableTopic Setup:
    - We used Spring Boot’s @RetryableTopic like this:
        ```
            @RetryableTopic(
                attempts = 3,
                backoff = @Backoff(delay = 2000),
                dltTopicSuffix = ".DLT"
            )
            @KafkaListener(topics = "ip-events", groupId = "ip-service")
            public void handleIPEvent(String message) {
                // processing logic
            }
        ```
    - This retried the message 3 times, with a 2-second delay between attempts.

2. Why Retry?
    - Most failures were transient (e.g., DB down, timeout, network glitch).
    - Retrying avoided data loss due to short-term issues.

3. Dead Letter Topic (DLT)
    - After max retries, messages were routed to a .DLT topic automatically.
    - We had a separate listener for DLT that:
        - Logged the failure.
        - Persisted the message in DB for manual replay.
        - Triggered alerts (e.g., Slack, email).
            ```
                @KafkaListener(topics = "ip-events.DLT", groupId = "dlt-group")
                public void handleDLT(String message) {
                    // log and alert
                }
            ```
    - This helped us avoid infinite retries and gave us a way to inspect and recover bad messages.

4. Manual Recovery (Optional):
    - For critical messages, we had a tool to manually replay DLT messages after fixing the root issue (e.g., bad data or service outage).

5. Bonus: Circuit Breaker Logic (Optional):
    - If we saw continuous failures (e.g., 10+ DLTs in a row), we used a circuit breaker pattern to:
        - Pause consumers temporarily
        - Auto-recover when the system stabilized

6. Final Summary Line:
    - We used @RetryableTopic in Spring Kafka to retry transient errors 3 times with backoff
    - After failure, messages were routed to Dead Letter Topics (DLTs), where they were logged, inspected, and optionally replayed.
    - This protected the system from infinite retries and allowed reliable error recovery.

### API Integration (FeignClient / WebClient)

#### Q7. How did you pass authentication tokens when using FeignClient or WebClient?
**Answers:**In our project, all internal service-to-service calls were secured using JWT tokens.
When using FeignClient or WebClient, we injected the JWT into the Authorization header as Bearer token.

1. FeignClient – Used RequestInterceptor
    - We created a Spring bean that adds the token to all outgoing Feign requests:
        ```
            @Component
            public class FeignAuthInterceptor implements RequestInterceptor {
                @Override
                public void apply(RequestTemplate template) {
                    String token = fetchInternalJwt();  // can be static, rotated, or passed down from caller
                    template.header("Authorization", "Bearer " + token);
                }
            }
        ```
    - This works globally for all FeignClients.

2. WebClient – Injected Token in Headers
    - When using WebClient:
        ```
            WebClient webClient = WebClient.builder()
                .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + fetchInternalJwt())
                .build();

            webClient.get()
                .uri("http://subscription-service/api/subs")
                .retrieve()
                .bodyToMono(String.class);
        ```
    - We fetched the token before making the request and injected it into the header.

3. How the Token Was Generated / Retrieved:
    - If the original request had a JWT (user request), we propagated the same token.
    - If it was internal logic, we generated a service JWT via an internal auth mechanism.
    - The token included roles or scopes like ROLE_INTERNAL or SUBSCRIPTION_READ.

4. Real-world Scenarios:
    - For user-request chains: Extract token from SecurityContextHolder and forward it
    - For background jobs or async tasks: Use a pre-signed JWT or client credential flow

5. Final Summary (for Interview):
    - For FeignClient, we used a RequestInterceptor to inject a JWT token into the Authorization header.
    - For WebClient, we added the token manually in the header during the request build.
    - The token was either forwarded from the incoming request or generated internally for trusted service calls.

#### Q8. What timeout and retry configurations did you set?
**Answer:** In our microservice project, we configured timeouts and retry policies across both synchronous (REST) and asynchronous (Kafka) communication layers to improve system resilience and prevent cascading failures.

1. FeignClient Timeout + Retry Settings:
    - We configured global Feign timeouts and retry behavior in application.yml:
        
        ```
            feign:
                client:
                    config:
                    default:
                        connectTimeout: 3000
                        readTimeout: 5000
                        loggerLevel: full
                        retryer:
                            period: 2000         # initial retry interval (ms)
                            maxPeriod: 5000      # max interval between retries
                            maxAttempts: 3
        ```
    - This means:
        - Retry 3 times.
        - Wait 2–5 seconds between retries.
        - Timeout if no response in 3–5 seconds.

2. WebClient Timeout + Retry:
    - code:
        ```
            WebClient.builder()
                .baseUrl("http://ip-service")
                .clientConnector(new ReactorClientHttpConnector(
                    HttpClient.create()
                        .responseTimeout(Duration.ofSeconds(5))
                ))
                .build();
        ```
    - We added retries using Spring Retry or a fallback Mono:
        ```
            webClient.get()
            .uri("/api/data")
            .retrieve()
            .bodyToMono(String.class)
            .retry(3)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(ex -> Mono.just("fallback response"));
        ```
    - This allowed us to:
        - Retry temporary errors (timeouts, 5xx).
        - Return a fallback/default value when retries fail.

3. Retry for Kafka Consumers:
    - We used Spring Kafka’s @RetryableTopic:
        ```
            @RetryableTopic(
                attempts = 3,
                backoff = @Backoff(delay = 2000),
                dltTopicSuffix = ".DLT"
            )
            @KafkaListener(topics = "subscription-events")
            public void processEvent(String msg) {
                // business logic
            }
        ```
    - 3 attempts → 2 sec delay → then route to DLT if all fail.

4. RetryTemplate (for custom logic)
    - For DB calls or internal service retries:
        ```
            RetryTemplate retryTemplate = new RetryTemplateBuilder()
                .maxAttempts(3)
                .fixedBackoff(2000)
                .build();

            retryTemplate.execute(context -> {
                // Retryable code here
            });

        ```
    - Used in low-level service methods where needed.

5. Why We Did This:
    - Without timeout and retry limits, a slow or failing service can hold up the thread pool, increase latency, or trigger cascading failures.

6. Final Summary Line for Resume/Interview:
    - We set connection and read timeouts (3–5 sec) for Feign and WebClient, and implemented retry policies using Spring Retry and @RetryableTopic.
    - Failures were routed to DLTs or handled with fallback logic to ensure fault isolation and service resilience.

#### Q9. What was your fallback strategy in case of downstream failure?
**Answer:** In our networking microservice project, we implemented fallback strategies to handle downstream failures — especially for synchronous REST calls between services. Our goal was to make sure that the failure of one service didn't cascade and affect the availability of the entire system.

1. Retries + Fallbacks with FeignClient
    - For Feign, we implemented fallback logic using either:
        - Feign fallback classes, or
        - Manual try-catch + default response
        ```
            @FeignClient(name = "ip-service", fallback = IPServiceFallback.class)
            public interface IPServiceClient {
                @GetMapping("/ip/{id}")
                IPDetails getIPDetails(@PathVariable String id);
            }

            @Component
            public class IPServiceFallback implements IPServiceClient {
                @Override
                public IPDetails getIPDetails(String id) {
                    return new IPDetails("N/A", "unavailable"); // default response
                }
            }

        ```
    - This returned a graceful fallback response without breaking the flow.

2. WebClient – Fallback via onErrorResume
    - code:
        ```
            webClient.get()
            .uri("/ip/{id}", id)
            .retrieve()
            .bodyToMono(IPDetails.class)
            .timeout(Duration.ofSeconds(3))
            .onErrorResume(ex -> Mono.just(new IPDetails("fallback", "error")));
        ```
    - Used onErrorResume to return fallback DTO or redirect to another service

3. Predefined Defaults or Partial Response
    - In critical flows (e.g., customer troubleshooting), if a downstream like IPService was down:
        - We continued the flow with partial data
        - We showed “Unable to fetch IP info, try again later” in the response
        - We didn’t block the rest of the troubleshooting logic
    - This improved user experience and availability

4. Kafka-Based Fallbacks (Async)
    - If an async consumer (like ProvisioningService) failed:
        - We retried via @RetryableTopic
        - After retries, we emitted a failure event
        - A compensating service picked it up and rolled back the previous step
    - This was part of our Saga rollback strategy

5. Monitoring + Alerting (Supportive Strategy)
    - We used:
        - Logging with error tags
        - Prometheus/Grafana alerts for high failure rates
        - DLTs for unprocessed Kafka messages
    - So that the fallback wasn't silent — we knew when something failed

6. Final Summary Line for Interview
    - We implemented fallback strategies using Feign fallback classes, WebClient’s onErrorResume, and default partial responses.
    - For Kafka consumers, we used retries + failure events to trigger compensating actions.
    - This ensured graceful degradation and fault isolation without breaking user flows

#### Q10. How did you handle partial failures across chained services?
**Answers:** In our microservices-based networking project, we handled partial failures differently for synchronous (REST) and asynchronous (Kafka/event-based) service chains.
Our goal was to avoid system-wide failure and ensure data consistency or graceful degradation.

1. In Synchronous (REST) Chains: Retry + Fallback + Partial Success
    - Example:
        - TroubleshootingService → IPService → SubscriptionService
        - If IPService fails:
            - We retried the call up to 3 times using Spring Retry or WebClient .retry()
            - If still failing, we used a fallback response or skipped that step
            - Returned a partial response to the user:
                
                ```
                    {
                        "subscriptionStatus": "Active",
                        "ipAssignment": "Unavailable - try again later"
                    }
                ```
        - This allowed the flow to continue, while logging and alerting the issue

2. In Asynchronous (Kafka) Chains: Compensating Events (Saga Pattern)
    - Example:
        - CustomerCreated → SubscriptionCreated → IPAssigned → Provisioned
            - If IPAssignmentService fails:
                1. It emits a IPAssignmentFailed event
                2. SubscriptionService listens and triggers CancelSubscription
                3. This is part of a choreography-based Saga
            - This undoes previous steps using compensating actions, not rollback

3. Outbox Pattern for Recovery:
    - If the failure occurred during Kafka publishing, we used the Outbox Pattern to:
        - Save the event in a DB table during the same transaction
        - Retry sending later using a scheduled publisher
    - Prevented incomplete flow due to lost events

4. Fallback Queues or Delay Mechanisms:
    - In cases where a service was temporarily down, we used:
        - Delay queues (future retry)
        - Buffering in Redis or temporary tables
    - This allowed eventual retry without disrupting the full chain

5. Alerting + Manual Recovery
    - For non-compensatable failures (e.g., corrupted input), messages were routed to a Dead Letter Topic.
    - Operators were alerted, and they could manually replay the failed step after root cause fix.

6. Final Summary
    - We handled partial failures using a mix of retries, fallback responses, compensating events (Saga), and the Outbox Pattern.
    - In synchronous chains, we degraded gracefully with partial responses.
    - In async chains, we used failure events to roll back prior steps.
    - For unrecoverable failures, we alerted teams and allowed manual intervention through DLTs or recovery tools.

#### Q11 How did you log and trace an entire API call across services?
**Answer:** In our microservices architecture, we implemented distributed tracing using Spring Cloud Sleuth with Zipkin. This allowed us to follow an API call as it traveled across multiple services and Kafka topics, and identify bottlenecks or failures.

1. Correlation ID for Traceability
    - Each incoming API request was assigned a traceId and spanId by Spring Cloud Sleuth
    - These IDs were propagated automatically via:
        - HTTP headers in Feign/WebClient calls
        - Kafka message headers for async events
    - Example headers:
        ```
            X-B3-TraceId: a1b2c3d4
            X-B3-SpanId: 1234abcd
            X-B3-Sampled: 1
        ```
    - This ensured that logs from all services for the same request shared the same traceId.

2. Feign + WebClient Propagation
    - Sleuth integrated directly with Feign and WebClient
    - The traceId was injected into Authorization + tracing headers automatically, so we didn’t have to code it manually.

3. Kafka Message Tracing
    - When producing messages, Sleuth added trace headers to Kafka records.
    - When consuming, these headers were read back so that logs for that message still had the original traceId.
    - Example:
        ```
            headers.add("X-B3-TraceId", traceId);
        ```

4. Centralized Trace Viewing with Zipkin
    - We deployed Zipkin as a tracing server
    - Developers could search by traceId and see the full call chain:
        ```
            TroubleshootingService → CustomerService → SubscriptionService → IPService → ProvisioningService
        ```
    - Each hop showed:
        - Start time
        - Duration
        - Status
        - Any errors

5. Logging Format: Our logs were JSON-formatted with:
    - json:
        ```
            {
                "traceId": "a1b2c3d4",
                "spanId": "1234abcd",
                "level": "INFO",
                "service": "IPService",
                "message": "IP assigned successfully"
            }
        ```
    - This allowed Kibana or Grafana Loki dashboards to search logs by traceId
6. Final Summary Line for Interview:
    - We used Spring Cloud Sleuth to generate and propagate traceIds across REST and Kafka calls, and Zipkin for centralized trace visualization.
    - This allowed us to log and trace an entire API request across all services and quickly pinpoint where failures occurred.

### Spring Boot & Backend

#### Q12 How did you organize your project structure and layers (Controller, Service, Repository)?
**Answer:** In our microservices, we followed a layered architecture within each service for clarity, maintainability, and testability. Each service was organized into Controller, Service, Repository, and Model/DTO packages, along with configuration and utility layers.

Project Structure (Example: SubscriptionService):

    ```
        com.isp.subscriptionservice
        │
        ├── controller
        │   └── SubscriptionController.java   # Handles REST endpoints
        │
        ├── service
        │   ├── SubscriptionService.java      # Interface
        │   └── impl
        │       └── SubscriptionServiceImpl.java # Business logic
        │
        ├── repository
        │   └── SubscriptionRepository.java   # Extends JpaRepository
        │
        ├── model
        │   ├── entity
        │   │   └── Subscription.java         # JPA entity
        │   └── dto
        │       └── SubscriptionDTO.java      # For API communication
        │
        ├── event
        │   └── SubscriptionCreatedEvent.java # Kafka event object
        │
        ├── config
        │   └── KafkaConfig.java               # Producer/Consumer configs
        │
        ├── exception
        │   ├── SubscriptionNotFoundException.java
        │   └── GlobalExceptionHandler.java
        │
        └── util
            └── DateUtils.java

    ```

1. Controller Layer:
    - Accepts HTTP requests
    - Validates inputs
    - Delegates to service layer

2. Service Layer:
    - Contains business logic
    - Calls repositories or external services (via Feign/WebClient)
    - Publishes Kafka events if needed

3. Repository Layer:
    - Direct interaction with DB (MySQL/Postgres)
    - Uses Spring Data JPA for CRUD operations

4. Event Layer:
    - Holds Kafka message payloads and consumers
    - Example: @KafkaListener methods

5. Config Layer
    - Kafka configs, Feign configs, security configs

6. Final Summary Line for Interview:
    - Each microservice followed a layered architecture: Controller for API handling, Service for business logic, Repository for persistence, and dedicated packages for events, configs, and exceptions.
    - This structure improved code maintainability, allowed easier unit testing, and ensured clear separation of concerns.

#### Q13 What kind of exception handling did you implement in your controllers?
**Answer:** In our microservices, we implemented centralized exception handling using Spring Boot’s @ControllerAdvice with @ExceptionHandler.

1. Centralized Global Exception Handler:
    - code:
        ```
            @ControllerAdvice
            public class GlobalExceptionHandler {

                @ExceptionHandler(ResourceNotFoundException.class)
                public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
                    return ResponseEntity.status(HttpStatus.NOT_FOUND)
                            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
                }

                @ExceptionHandler(InvalidRequestException.class)
                public ResponseEntity<ErrorResponse> handleInvalidRequest(InvalidRequestException ex) {
                    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                            .body(new ErrorResponse("BAD_REQUEST", ex.getMessage()));
                }

                @ExceptionHandler(Exception.class)
                public ResponseEntity<ErrorResponse> handleGenericException(Exception ex) {
                    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                            .body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
                }
            }

        ```

2. Standard Error Response Structure:
    - structure:
        ```
            public class ErrorResponse {
                private String errorCode;
                private String message;
                private LocalDateTime timestamp;

                public ErrorResponse(String errorCode, String message) {
                    this.errorCode = errorCode;
                    this.message = message;
                    this.timestamp = LocalDateTime.now();
                }
            }
        ```
3. Validation Error Handling (Bean Validation)
    - Used @Valid in controller methods
    - Handled MethodArgumentNotValidException to return field-level validation errors
    - code:
        ```
            @ExceptionHandler(MethodArgumentNotValidException.class)
            public ResponseEntity<Map<String, String>> handleValidationErrors(MethodArgumentNotValidException ex) {
                Map<String, String> errors = new HashMap<>();
                ex.getBindingResult().getFieldErrors().forEach(error ->
                    errors.put(error.getField(), error.getDefaultMessage())
                );
                return ResponseEntity.badRequest().body(errors);
            }

        ```
4. Final Summary Line for Interview:
    - We used a @ControllerAdvice-based centralized exception handler to catch application-specific, validation, and generic exceptions, returning consistent JSON error responses with proper HTTP status codes across all controllers.

#### Q14. How did you inject config values (like Kafka topic names or Redis TTLs)?
**Answer:** We externalized all configurable values — such as Kafka topic names, Redis TTLs, API URLs, and feature toggles — into application.yml or our centralized config server. Then we injected them using Spring’s @Value or @ConfigurationProperties for type-safe mapping.

1. Using application.yml
    ```
        kafka:
            topics:
                subscriptionCreated: subscription-created-topic
                ipAssigned: ip-assigned-topic

        redis:
            ttl:
                troubleshootingCache: 600 # seconds
    ```
2. Injecting with @Value
    ```
        @Value("${kafka.topics.subscriptionCreated}")
        private String subscriptionCreatedTopic;

        @Value("${redis.ttl.troubleshootingCache}")
        private long troubleshootingCacheTtl;
    ```
3. Type-Safe Config with @ConfigurationProperties
    ```
        @Component
        @ConfigurationProperties(prefix = "kafka.topics")
        public class KafkaTopicsConfig {
            private String subscriptionCreated;
            private String ipAssigned;

            // getters and setters
        }
        Advantage: Cleaner code, grouped configs, easy to maintain
    ```
4. Environment-Specific Overrides
    - Used application-dev.yml, application-prod.yml
    - Switched via --spring.profiles.active=prod
    - example:
        ```
            # application-prod.yml
            kafka:
                topics:
                    subscriptionCreated: subscription-created-prod
        ```
5. Centralized Config (Optional): For shared values (Kafka broker URLs, Redis host), we used Spring Cloud Config Server so that changes didn’t require redeploying services.

6. Final Summary Line for Interview:
    - We kept all Kafka topics, Redis TTLs, and other constants in application.yml or Config Server and injected them using @Value or @ConfigurationProperties.
    - This allowed easy updates, environment-specific configs, and avoided hardcoding.

#### Q15. What validations did you perform in your APIs and where?
**Answer:** We performed multi-layer validation to ensure both data integrity and business rules were enforced. 
    At the Controller Layer — request format and field-level constraints (using @Valid and Bean Validation annotations).
    At the Service Layer — business rule validations (e.g., subscription eligibility, IP availability).
    At the Persistence Layer — database constraints (e.g., unique indexes).

1. Request-Level Validations (Controller Layer)
    - We used Bean Validation (JSR-380) annotations:
        ```
            public class SubscriptionDTO {

                @NotNull(message = "Customer ID is required")
                private Long customerId;

                @NotBlank(message = "Plan name cannot be blank")
                private String planName;

                @Pattern(regexp = "^(ACTIVE|INACTIVE)$", message = "Invalid status")
                private String status;
            }
        ```
        ```
            @PostMapping
            public ResponseEntity<SubscriptionDTO> create(@Valid @RequestBody SubscriptionDTO dto) {
                return ResponseEntity.ok(subscriptionService.createSubscription(dto));
            }
        ```
2. Business Rule Validations (Service Layer)
    - A customer cannot have more than 1 active subscription
    - IP addresses must be from available pool before assignment
    - Subscription plans must be valid and active before provisioning
    - Example:
        ```
            if (subscriptionRepository.existsByCustomerIdAndStatus(customerId, "ACTIVE")) {
                throw new InvalidRequestException("Customer already has an active subscription");
            }
        ```
3. Database-Level Validations:
    - Unique constraints on:
        - ipAddress in IP table
        - subscriptionId in provisioning table
    - Foreign key constraints to maintain referential integrity

4. Cross-Service Validations:
    - For certain checks, we called other microservices via Feign/WebClient:
        - Customer Service → Check if customer exists before creating subscription
        - IP Service → Check IP availability before assignment

5. Error Handling:
    - Invalid requests triggered @ControllerAdvice-handled exceptions, returning structured JSON errors:
        ```
            {
                "errorCode": "BAD_REQUEST",
                "message": "Customer already has an active subscription",
                "timestamp": "2025-08-10T21:20:30"
            }
        ```
6. Final Summary Line for Interview:
    - We validated request formats at the controller level with Bean Validation, enforced business rules in the service layer, and used DB constraints for data integrity.
    - For cross-service validations, we used Feign/WebClient calls before proceeding with business operations.

#### Q16. What testing strategy did you follow — unit, integration, contract?
**Answers:** We followed a multi-level testing strategy to ensure both individual microservices and their interactions worked correctly.

a. Unit Tests — For business logic in isolation.
b. Integration Tests — For DB, Kafka, Redis, and inter-service calls.
c. Contract Tests — For ensuring APIs between services stayed compatible.
d. End-to-End Tests — For simulating a complete business flow.

1. Unit Testing (Service Layer)
    - Framework: JUnit 5 + Mockito
    - Mocked:
        - Repositories
        - Feign/WebClient calls
        - Kafka producers
    - code:
        ```
            @ExtendWith(MockitoExtension.class)
            class SubscriptionServiceTest {
                @Mock private SubscriptionRepository repo;
                @InjectMocks private SubscriptionServiceImpl service;

                @Test
                void shouldCreateSubscriptionSuccessfully() {
                    when(repo.save(any())).thenReturn(new Subscription(1L, "PLAN_A"));
                    SubscriptionDTO dto = service.createSubscription(new SubscriptionDTO(...));
                    assertEquals("PLAN_A", dto.getPlanName());
                }
            }

        ```
2. Integration Testing:
    - Framework: Spring Boot Test
    - Used @SpringBootTest(webEnvironment = RANDOM_PORT) for real HTTP calls
    - Spun up:
        - H2 in-memory DB for JPA
        - Embedded Kafka for event testing
        - Embedded Redis for caching tests
    - code:
        ```
            @SpringBootTest
            class SubscriptionControllerIntegrationTest {
                @Autowired private TestRestTemplate restTemplate;

                @Test
                void testCreateSubscriptionEndpoint() {
                    ResponseEntity<SubscriptionDTO> response = restTemplate.postForEntity(
                        "/subscriptions", new SubscriptionDTO(...), SubscriptionDTO.class
                    );
                    assertEquals(HttpStatus.OK, response.getStatusCode());
                }
            }
        ```

3. Contract Testing (Consumer-Driven Contracts)
    - Tool: Spring Cloud Contract
    - Ensured that if SubscriptionService changed an endpoint, consumers like ProvisioningService wouldn’t break
    - Contracts stored in Git, validated during CI runs

4. End-to-End Testing:
    - Tool: Postman/Newman collection runner
    - Automated flows:
        - Create Customer → Assign Subscription → Assign IP → Provision Device
        - Checked DB + Kafka to ensure the right events were triggered

5. Test Coverage:
    - Maintained 80%+ coverage for core services
    - Unit tests ran on every PR
    - Integration tests ran on nightly builds

6. Final Summary Line for Interview:
    - We used unit tests with Mockito for isolated logic, integration tests with embedded dependencies for real scenarios, contract tests with Spring Cloud Contract to avoid breaking changes, and Postman-based E2E tests to validate complete business flows.

### Redis & Caching

#### Q17. What kind of data did you cache in Redis and why?
**Answer:**
In our microservices, we used Redis for caching frequently accessed, read-heavy data to reduce latency and avoid repeated DB or cross-service calls.

1. Customer Profile Cache
    - Cached after first retrieval from Customer Service
    - Avoided repeated Feign/WebClient calls for the same customer
    - TTL: 24 hours (updated when profile changes)

2. Subscription Plan Cache
    - Subscription plans rarely change
    - Cached in Redis hash format for fast lookup by plan ID
    - TTL: 6 hours (manual invalidation on plan update events)

3. Troubleshooting Results
    - When a customer requested troubleshooting, we often fetched data from multiple systems (e.g., billing, network, provisioning)
    - These results were cached for 10 minutes to speed up subsequent checks during the same support session

4. IP Availability Lookups
    - IP pool availability changes slowly
    - Cached for 5 minutes to reduce DB queries during high provisioning load

5. Why Redis Was Chosen:
    - In-memory speed → Millisecond retrieval times
    - TTL support for automatic expiry
    - Distributed caching → Works across multiple microservice instances
    - Pub/Sub for cache invalidation events

6. Final Summary Line for Interview:
    - We cached read-heavy and rarely changing data like customer profiles, subscription plans, troubleshooting results, and IP availability in Redis to reduce DB load, lower API latency, and improve customer experience, with TTLs tuned to each data type’s volatility.

#### Q18. How did you handle cache invalidation or expiration?
**Answer:**
We used a combination of TTL-based expiry and event-driven invalidation for Redis cache entries.
This ensured cached data was fresh while avoiding unnecessary database or service calls.

1. TTL-Based Expiration:
    - Configured per-cache TTLs in application.yml based on data volatility:
        ```
        spring:
            cache:
                cache-names: customerCache, planCache
                redis:
                    time-to-live:
                        customerCache: 86400000  # 24 hours in ms
                        planCache: 21600000      # 6 hours
        ```
    - Auto-clears stale entries without manual intervention
2. Manual Invalidation on Updates:
    - Used @CacheEvict to remove specific keys when data changes:
        ```
            @CacheEvict(value = "customerCache", key = "#customerId")
            public CustomerDTO updateCustomerProfile(Long customerId, CustomerDTO dto) {
                return feignClient.updateCustomer(customerId, dto);
            }
        ```
    - Ensures updated data is fetched fresh next time.
3. Event-Driven Invalidation (Kafka)
    - When data changed in one service, it published an event
    - Other services subscribed to that Kafka topic and evicted relevant cache entries
    - code:
        ```
            @KafkaListener(topics = "customer-updated")
            public void handleCustomerUpdate(CustomerUpdatedEvent event) {
                redisTemplate.delete("customerCache::" + event.getCustomerId());
            }
        ```
    - Keeps caches in sync across multiple microservice instances

4. Short TTL for Highly Volatile Data
    - For data like IP availability, TTL was just 5 minutes to ensure near-real-time accuracy without constant invalidations.

5. Startup Cache Warm-Up (Optional)
    - For frequently used data (e.g., active plans list), we preloaded the cache on application startup to avoid cold-cache delays.

6. Final Summary Line for Interview:
    - We used TTL-based expiry for regular cleanup, @CacheEvict for direct updates, and Kafka event-driven invalidation for cross-service consistency.
    - This avoided stale data issues while keeping latency low.

#### Q19. What Redis data structure did you use (String, Hash, List)?
**Answer:** We used different Redis data structures depending on the type of data:

1. String — for simple lookups
    - Use case: IP availability cache
    - Key: "ip-availability:poolId"
    - Value: "AVAILABLE" or "UNAVAILABLE"
    - Reason: Fast O(1) GET/SET operations
        ```
            redisTemplate.opsForValue().set("ip-availability:123", "AVAILABLE", Duration.ofMinutes(5));
        ```
2. Hash — for structured objects
    - Use case: Customer profile cache
    - Key: "customer:customerId"
    - Fields: "name", "email", "planId", "status"
    - Reason: Avoided storing the entire object as JSON when only a few fields needed updating
        ```
            redisTemplate.opsForHash().put("customer:101", "name", "John Doe");
            redisTemplate.opsForHash().put("customer:101", "planId", "PLAN_A");
        ```

3. List — for ordered recent troubleshooting events
    - Use case: Store last 10 troubleshooting logs for quick retrieval in UI
    - Reason: Maintains order of events and supports trimming to limit memory usage
        ```
            redisTemplate.opsForList().leftPush("troubleshoot:logs:101", "Checked IP assignment");
            redisTemplate.opsForList().trim("troubleshoot:logs:101", 0, 9);
        ```

4. Why we avoided only using Strings for everything?
    - Storing everything as JSON in Strings would require full deserialization even if only one field was needed.
    - Hashes allowed partial updates without rewriting the entire object

5. Final Summary Line for Interview:
    - We used Strings for simple key-value lookups, Hashes for structured objects like customer profiles, and Lists for ordered troubleshooting logs.
    - This choice reduced memory usage, improved read/write performance, and fit each data’s access pattern.

#### Q20. How did you handle cache misses and fallback to DB?
**Answer**
1. We followed the cache-aside pattern for Redis
    - Check cache first
    - If data is missing (cache miss), fetch from DB or service
    - Store it back in cache with a TTL for future requests
    - This ensured data was always available while minimizing DB hits

2. Implementation Flow
    - Client calls API
    - Controller delegates to Service layer
    - Service tries to fetch from Redis
    - If cache hit → return immediately
    - If cache miss → fetch from DB → store in Redis → return
    - Example Code
        ```
        public CustomerDTO getCustomerProfile(Long customerId) {
            String cacheKey = "customer:" + customerId;

            // 1. Try from cache
            CustomerDTO cached = (CustomerDTO) redisTemplate.opsForValue().get(cacheKey);
            if (cached != null) {
                return cached;
            }

            // 2. Fallback to DB/service
            CustomerDTO fromDb = customerRepository.findById(customerId)
                    .orElseThrow(() -> new ResourceNotFoundException("Customer not found"));

            // 3. Store in cache
            redisTemplate.opsForValue().set(cacheKey, fromDb, Duration.ofHours(24));

            return fromDb;
        }
        ```
3. Avoiding Cache Stampede:
    - Used randomized TTLs for highly requested keys to prevent all keys from expiring at the same time.
    - For certain critical data, pre-warmed the cache at service startup.

4. Handling Cache Failures:
    - If Redis was down, we bypassed the cache and directly used DB results
    - Logged a warning but did not break the user flow

5. Benefits:
    - Faster responses on cache hits
    - Reduced load on DB and external APIs
    - Guaranteed fallback so users didn’t face downtime due to cache issues

6. Final Summary Line for Interview:
    - We implemented the cache-aside pattern — check Redis first, fallback to DB on misses, then repopulate the cache — and used random TTLs plus pre-warming to avoid stampedes.
    - If Redis failed, we bypassed it to keep the system available.

#### Q21. Was Redis used just as cache or for any pub-sub too?
**Answer:** In our project, Redis was primarily used as a cache, but we also leveraged its Pub/Sub feature for certain lightweight, near-real-time updates where full Kafka integration was unnecessary.

1. For example:
    - Cache invalidation across service instances: When one service updated a customer profile, it published a Redis message to a channel like customer-updated. Other instances subscribed to this channel and evicted the stale cache key
    - Lightweight notifications: For quick, internal service-to-service messages that didn’t require persistence or complex consumer groups (where Kafka would be overkill).

2. Why Redis Pub/Sub was used in addition to Kafka:
    - Kafka was used for business events (subscription created, IP assigned, provisioning done)
    - Redis Pub/Sub was used for infrastructure events (cache invalidations, internal quick signals)
    - Pub/Sub messages are fire-and-forget and don’t get stored — so it was perfect for ephemeral updates
    - Example — Cache Invalidation Flow:
        - Publisher:
            ```
            redisTemplate.convertAndSend("customer-updated", customerId);
            ```
        - Subscriber:
            ```
            @EventListener
            public void onRedisMessage(Message message, byte[] pattern) {
                String customerId = new String(message.getBody());
                redisTemplate.delete("customer:" + customerId);
            }

            ```
3. Benefits:
    - Low latency for intra-service notifications
    - Avoided adding extra Kafka topics for internal housekeeping events
    - Simple to set up — no additional brokers needed beyond our cache

4. Final Summary Line for Interview: Redis was mainly used for caching, but we also used its Pub/Sub feature for fast, lightweight notifications — particularly for cache invalidation across microservice instances, while Kafka handled business events.

### Security (JWT & Internal Communication)

#### Q22 How was JWT authentication configured in Spring Security?
**Answer:** We implemented stateless authentication using JWT tokens in Spring Security. Incoming requests were validated by a JWT filter before hitting the controllers, and no session state was stored on the server. Tokens were issued by our Authentication Service and passed between services for authorization.

1. Token Issuance (Authentication Service)
    - User logs in → credentials validated
    - On success, we generated a signed JWT containing:
        - sub (username or service name)
        - roles
        - iat (issued at) and exp (expiry time)
        ```
        String token = Jwts.builder()
            .setSubject(user.getUsername())
            .claim("roles", roles)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 86400000)) // 1 day
            .signWith(SignatureAlgorithm.HS512, secretKey)
            .compact();
        ```
2. Passing JWT in Requests:
    - For external user calls: sent in Authorization: Bearer token header.
    - For internal service calls (Feign/WebClient): added token to headers before making the request.
    - code:
        ```
        RequestInterceptor requestInterceptor = requestTemplate ->
        requestTemplate.header("Authorization", "Bearer " + jwtTokenProvider.getServiceToken());
        ```
3. JWT Filter in Spring Security:
    - A custom OncePerRequestFilter extracted the token from the header
    - Validated signature and expiry using our JwtTokenProvider
    - On success, set an Authentication object in SecurityContext
    - code:
        ```
        public class JwtAuthFilter extends OncePerRequestFilter {
            @Override
            protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
                    throws ServletException, IOException {
                String header = request.getHeader("Authorization");
                if (header != null && header.startsWith("Bearer ")) {
                    String token = header.substring(7);
                    if (jwtTokenProvider.validateToken(token)) {
                        Authentication auth = jwtTokenProvider.getAuthentication(token);
                        SecurityContextHolder.getContext().setAuthentication(auth);
                    }
                }
                chain.doFilter(request, response);
            }
        }
        ```
4. Spring Security Configuration:
    - Disabled session creation (STATELESS)
    - Registered JwtAuthFilter before UsernamePasswordAuthenticationFilter
    - code:
        ```
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
                .authorizeRequests()
                .antMatchers("/auth/**").permitAll()
                .anyRequest().authenticated();
        }

        ```
5. Token Validation Rules:
    - Signature verification using shared secret
    - Expiry check to reject old tokens
    - Optional: checked roles claim to enforce role-based access control (RBAC)

6. Why JWT Was Chosen
    - Stateless → no session storage needed
    - Works well with horizontal scaling
    - Simple to pass between microservices

7. Final Summary Line for Interview:
    - We used JWT for stateless authentication in Spring Security, with a custom filter for token validation, tokens issued by the Auth Service, and propagation via Authorization headers for both user and service-to-service calls.

#### Q23. How did your microservices validate each other in internal calls?
**Answer**  We implemented JWT-based internal authentication between microservices. Each service validated incoming requests from other services by checking a service-issued token in the Authorization header. 

Tokens were issued by our Authentication Service with a dedicated service account and signed with a shared secret key known only to our microservices.

1. Token Generation for Internal Calls:
    - A service account (e.g., subscription-service) had its own credentials
    - Generated a JWT containing:
        - sub = service name
        - roles = allowed actions
        - iat / exp
        - code:
            ```
            String token = jwtTokenProvider.generateServiceToken("subscription-service");
            ```

2. Passing Token in Feign/WebClient
    - For Feign:
        ```
        @Bean
        public RequestInterceptor serviceAuthInterceptor(JwtTokenProvider jwtTokenProvider) {
            return requestTemplate -> requestTemplate.header(
                "Authorization", "Bearer " + jwtTokenProvider.getServiceToken()
            );
        }
        ```
    - For WebClient:
        ```
        webClient.get()
        .uri("/internal/data")
        .header("Authorization", "Bearer " + jwtTokenProvider.getServiceToken())
        .retrieve()
        .bodyToMono(String.class);

        ```

3. Validation in Receiving Service:
    - Incoming requests hit the JWT filter before the controller
    - Filter validated:
        - Token signature using shared secret
        - Token expiry
        - sub claim matches trusted service list
            ```
            if (!trustedServices.contains(jwt.getSubject())) {
                throw new UnauthorizedServiceException("Invalid service caller");
            }
            ```
4. Why We Didn’t Use OAuth2 Between Services
    - Internal calls were inside our private network
    - JWT + shared secret was lightweight and faster
    - OAuth2 was used only for user-facing authentication

5. Benefits
    - No hardcoding of credentials in services — tokens generated dynamically
    - Each service could reject calls from unknown services
    - Maintained traceability by embedding traceId + service name in the JWT claims

6. Final Summary Line for Interview:
    - Internal calls were secured using JWT tokens issued to service accounts. Tokens were passed in Authorization headers, validated by a shared secret, and checked against a trusted service list to ensure only authorized microservices could communicate.


#### Q24. Where did you store shared secrets or keys between services?
**Answer:** we stored them in secure externalized configuration so that they could be rotated without redeploying services. Depending on the environment, we used: Spring Cloud Config Server with encrypted values (AES encryption via spring-cloud-config-server). Kubernetes Secrets in non-fake scenarios (but in our project, Config Server). Local development → .env files loaded via Spring Boot config.

1. Spring Cloud Config Server (Primary)
    - Config Server read from a private Git repo
    - Sensitive values were encrypted using Config Server’s built-in encryption:
        ```
        curl localhost:8888/encrypt -d 'my-secret-key'
        # returns encrypted value
        ```
    - In application.yml:
        ```
        jwt:
            secret: "{cipher}af31d8e3a3f2f4..."
        ```
    - Spring automatically decrypted them at runtime

2. Environment Variables (For Deployment)
    - Overrode secrets via:
        ```
        export JWT_SECRET=my-secret-key
        export KAFKA_PASSWORD=my-password
        ```
    - Spring Boot automatically picked these up from application.yml placeholders
        ```
        jwt:
          secret: ${JWT_SECRET}

        ```
3. Restricted Access:
    - Only the Auth Service and trusted microservices had access to JWT signing keys
    - Secrets were not stored in logs, debug dumps, or exception messages

4. Rotation Policy
    - Keys rotated every 90 days
    - Config updates triggered service refresh via Spring Cloud Bus without full redeployment

5. Final Summary Line for Interview:
    - We stored all shared secrets (JWT signing keys, Kafka credentials, Redis passwords) in an encrypted Config Server repo, loaded them at runtime via environment variables, and rotated them periodically
    - This avoided hardcoding secrets and ensured secure inter-service communication

#### Q25: Did you restrict access to certain endpoints based on roles or scopes?
**Answer:** Yes, we implemented role-based endpoint restrictions using Spring Security.
The roles were embedded inside JWT claims, and Spring Security checked these roles before allowing access to endpoints. For certain APIs, we also used scope-based access control to ensure that even within a role, only specific actions were allowed.

1. Roles in JWT Claims:
    - When issuing tokens in the Auth Service:
        ```
        String token = Jwts.builder()
        .setSubject(username)
        .claim("roles", List.of("ROLE_ADMIN", "ROLE_SUPPORT"))
        .claim("scopes", List.of("SUBSCRIPTION_READ", "SUBSCRIPTION_WRITE"))
        .signWith(SignatureAlgorithm.HS512, secretKey)
        .compact();
        ```
2. Restricting Endpoints in Security Config:
    - code:
        ```
            @Override
            protected void configure(HttpSecurity http) throws Exception {
                http.csrf().disable()
                    .authorizeRequests()
                    .antMatchers("/admin/**").hasRole("ADMIN")
                    .antMatchers(HttpMethod.POST, "/subscriptions/**").hasAuthority("SUBSCRIPTION_WRITE")
                    .antMatchers(HttpMethod.GET, "/subscriptions/**").hasAuthority("SUBSCRIPTION_READ")
                    .anyRequest().authenticated();
            }
        ```
    - Ensured:
        - Only admins could call /admin/**
        - Only users with SUBSCRIPTION_WRITE scope could create/update subscriptions
        - Read-only users could only fetch data

3. Scope Validation for Inter-Service Calls
    - For internal APIs, we checked both:
        - The service name in sub claim
        - The scopes claim to ensure the calling service was allowed to perform the requested action
            ```
            if (!jwtScopes.contains("CUSTOMER_READ")) {
                throw new AccessDeniedException("Service not allowed to read customer data");
            }

            ```

4. Benefits:
    - Granular security → not just role-based, but also permission-based
    - Single JWT token handled both user and service authorization
    - Avoided accidental overexposure of admin APIs

5. Final Summary Line for Interview:
    - Yes — we enforced role-based and scope-based access control using claims in JWT tokens
    - Spring Security configurations ensured only users/services with the right roles or scopes could call sensitive endpoints.

#### Q26: How did you handle expired or tampered JWT tokens?
**Answer:** We handled both expired and tampered JWT tokens at the JWT filter layer in Spring Security, before the request reached any business logic. Tokens were validated for expiry, signature, and claims integrity, and if invalid, we returned a structured 401 Unauthorized response without processing the request further.

1. Expiry Validation:
    - Every JWT had an exp claim set by the Auth Service
    - During validation, we checked if the token’s expiry time was before the current time
        ```
        public boolean isTokenExpired(String token) {
            return extractExpiration(token).before(new Date());
        }
        ```
    - Expired tokens triggered an immediate 401 Unauthorized with an error message:
        ```
        {
            "error": "JWT_EXPIRED",
            "message": "Your session has expired. Please log in again."
        }

        ```
2. Tampering Detection:
    - We verified the token’s HMAC signature using our shared secret key
    - If the signature didn’t match, it meant the payload was altered
        ```
        Jwts.parser()
        .setSigningKey(secretKey)
        .parseClaimsJws(token); // Throws exception if tampered

        ```
    - Tampered tokens triggered:
        ```
        {
            "error": "JWT_INVALID_SIGNATURE",
            "message": "Token signature does not match"
        }

        ```
3. Centralized Handling in JWT Filter:
    - We used a custom OncePerRequestFilter
        ```
        try {
            if (jwtTokenProvider.validateToken(token)) {
                Authentication auth = jwtTokenProvider.getAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        } catch (ExpiredJwtException e) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "JWT expired");
        } catch (JwtException e) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid JWT");
        }

        ```
    - This stopped bad tokens at the gateway/service edge before hitting controllers

4. Refresh Token Mechanism
    - Access tokens were short-lived (15 mins)
    - Users got a refresh token (valid 7 days) to obtain a new access token without re-login
    - Prevented misuse if an access token was leaked

5. Benefits
    - Early rejection of invalid requests
    - Clear error responses for easier debugging
    - Reduced risk from stolen or tampered tokens

6. Final Summary Line for Interview:
    - We validated tokens in a JWT filter before controller access, rejecting expired tokens with a 401 and detecting tampering via signature verification
    - We also used short-lived access tokens with refresh tokens to limit exposure.

### Failure, Recovery, and Observability

#### Q27: How did you detect failures in downstream services or Kafka?
**Answer:** We detected failures using a combination of runtime exception handling, timeouts/retries, and monitoring tools.

For downstream microservices, we implemented Feign/WebClient error handling and health checks.

For Kafka, we used consumer lag monitoring, error topic routing, and alerts in Grafana/Prometheus.

1. Detecting Failures in Downstream Services:
    - Feign/WebClient Exception Handling
        - Configured timeouts so calls didn’t hang indefinitely:
            ```
            feign:
                client:
                    config:
                        default:
                            connectTimeout: 2000
                            readTimeout: 5000
            ```
        - Caught exceptions like:
            - FeignException.ServiceUnavailable
            - WebClientResponseException

        - Logged them with traceId for correlation

    - Circuit Breaker (Resilience4j)
        - If a downstream service kept failing, the circuit opened
        - Allowed us to quickly see in logs + metrics which service was down
    
    - Spring Boot Actuator Health Checks
        - /actuator/health probes for service dependencies
        - Integrated with monitoring tools to raise alerts when unhealthy

2. Detecting Failures in Kafka
    1. Consumer Error Handling
        - If message processing failed, we retried
        - After max retries, message was sent to a Dead Letter Topic (DLT) for investigation
    2. Consumer Lag Monitoring
        - Used Prometheus + Kafka Exporter to track
            - Lag per consumer group
            - Number of unprocessed messages
        - If lag spiked above threshold, alerts triggered
    
    3. Broker/Cluster Health
        - Monitored:
            - Broker availability
            - Partition under-replication
            - Leader election times

3. Alerting
    - Grafana Alerts for:
        - Downstream API failure rate > 5% in 5 mins
        - Kafka consumer lag > threshold
    - Slack/Webhook Notifications sent to support team instantly

4. Benefits
    - Quick isolation of failures
    - Minimal downtime due to fast detection
    - Clear logs + metrics for root cause analysis

5. Final Summary Line for Interview:
    - We detected downstream failures using Feign/WebClient error handling, timeouts, and circuit breakers, and monitored Kafka with consumer lag metrics, DLTs, and broker health checks.
    - Alerts in Grafana/Prometheus notified us instantly when thresholds were breached.

#### Q28: How did you retry operations safely and when did you stop retrying?
**Answer:** We implemented safe retry logic using Resilience4j for HTTP calls and Kafka retry topics for message processing.

Retries were limited, delayed, and idempotent to avoid duplicate side effects.

1. Safe Retries for Downstream Service Calls
    - Tool: Resilience4j Retry + Backoff
    - Configured:
        - Max attempts: 3
        - Wait between retries: exponential backoff (500ms → 1s → 2s)
        - Retry only on transient errors (5xx, timeouts)

        ```
        RetryConfig config = RetryConfig.custom()
        .maxAttempts(3)
        .waitDuration(Duration.ofMillis(500))
        .retryExceptions(TimeoutException.class, IOException.class)
        .build();
        ```
    - Prevented infinite retry loops and avoided retrying on 4xx errors.
2. Safe Retries for Kafka Consumers
    - Used Retry Topics
        - First retry topic → delay of 1 min
        - Second retry topic → delay of 5 min
        - If still failing → moved to Dead Letter Topic (DLT)
        ```
        @RetryableTopic(
            attempts = "3",
            backoff = @Backoff(delay = 1000, multiplier = 2.0),
            dltTopicSuffix = "-dlt"
        )
        public void consume(Message message) {
            // processing logic
        }
        ```
3. Idempotency to Avoid Duplicates
    - Before retrying, we checked if the operation was already completed
    - Example:
        - For "Assign IP" calls → first check if the customer already has an IP before retrying
        - For "Create Subscription" → checked in DB by unique transaction ID
    
4. Stop Retrying When:
    - Permanent errors (4xx, business validation failures)
    - After max retry attempts
    - When downstream service is confirmed unavailable for long (circuit breaker open)

5. Benefits:
    - Reduced load on failing services
    - Prevented accidental double-processing
    - Gave operators visibility into permanent vs transient failures

6. Final Summary Line for Interview:
    - We used Resilience4j for safe HTTP retries and Kafka retry topics for message reprocessing, with exponential backoff, idempotent checks, and a max attempt limit. Retries stopped on permanent errors or after configured attempts, sending failures to a Dead Letter Topic for investigation.

#### Q29: How did you monitor logs and metrics across services?
**Answer:** We used a centralized logging and monitoring stack so we could trace requests, track service health, and set up alerts.

Logging was handled via ELK (Elasticsearch, Logstash, Kibana) with trace IDs, and metrics via Prometheus + Grafana.

This allowed us to view logs and metrics from all microservices in a single place.

1. Centralized Logging (ELK Stack)
    - Logback + JSON Encoder in each service sent logs to Logstash
    - Added traceId and spanId in log patterns for correlation:
        ```
        %X{X-B3-TraceId} %X{X-B3-SpanId} %-5level %logger{36} - %msg%n
        ```
    - Example log:
        ```
        traceId=8a12fe12 spanId=42 level=INFO CustomerService - Retrieved profile in 45ms
        ```
    - Kibana dashboards let us search logs by customer ID or traceId across all services

2. Distributed Tracing:
    - Used Spring Cloud Sleuth + Zipkin
    - Every request carried traceId + spanId via HTTP headers or Kafka message headers
    - Zipkin UI showed end-to-end request flow and where time was spent

3. Metrics Collection (Prometheus + Grafana)
    - Each service exposed /actuator/prometheus endpoint
    - Metrics collected:
        - API request count, latency, error rate
        - Kafka consumer lag
        - Redis hit/miss ratio
        - JVM health (heap, GC pauses)
    - Grafana dashboards for real-time service health
    - Alert rules
        - Error rate > 5% → Slack alert
        - Consumer lag > threshold → Ops ticket

4. Correlation Between Logs & Metrics:  
    - By using traceId, we could:
        - Jump from a Grafana alert to Kibana logs for the exact failing request
        - Identify whether failures were in network calls, Kafka processing, or DB queries

5. Benefits:
    - Single-pane view of all logs and metrics
    - Faster root cause analysis
    - Ability to spot issues before customers reported them

6. Final Summary Line for Interview:
    - We used ELK for centralized logging with trace IDs, Sleuth + Zipkin for distributed tracing, and Prometheus + Grafana for metrics and alerts.
    - This let us monitor all microservices in one place and quickly trace problems across service boundaries.

#### Q30. What alerting system did you have in case a consumer failed or API broke?
**Answer:** We used Prometheus + Grafana for alerting, with integration into Slack and our incident management system. 

Alerts were triggered when key metrics crossed thresholds — for example, Kafka consumer lag spikes or API error rates above normal levels.

1. Alerting for Kafka Consumers:
    - Metric monitored:
        - Consumer lag (kafka_consumergroup_lag)
        - Consumer availability (via /actuator/health)
    - Rules:
        ```
        alert: KafkaConsumerLagHigh
        expr: kafka_consumergroup_lag > 1000
        for: 2m
        labels:
            severity: critical
        annotations:
            description: "Consumer group {{ $labels.group }} lag is high."
        ```
    - If lag > 1000 for more than 2 minutes → alert triggered

2. Alerting for API Failures
    - Metric monitored:
        - API error rate from /actuator/metrics/http.server.requests
        - Rules:
            ```
            alert: HighAPIErrorRate
            expr: sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) 
                / sum(rate(http_server_requests_seconds_count[5m])) > 0.05
            for: 1m
                labels:
                    severity: warning
            annotations:
                description: "API error rate is above 5% for the last minute."

            #If error rate > 5% → alert triggered
            ```
3. Alert Destinations:
    - Slack channel for engineering team → instant visibility
    - PagerDuty / Opsgenie for critical production incidents
    - Automatic ticket creation in Jira for recurring issues

4. Benefits:
    - Proactive detection before customers reported issues
    - Fast response time — engineers got alerts within 1–2 minutes
    - Clear context in alerts (consumer group, topic name, API path) for quicker debugging

5. Final Summary Line for Interview:
    - We used Prometheus + Grafana to trigger alerts for high Kafka consumer lag and elevated API error rates
    - Alerts were sent to Slack and PagerDuty so the team could react within minutes, reducing downtime and customer impact

#### Q31: Did you implement circuit breakers or rate limiting?
**Answer:** Yes, we implemented circuit breakers using Resilience4j to protect against failing downstream services, and rate limiting using Bucket4j for APIs that could be abused or cause backend overload.

1. Circuit Breakers (Resilience4j)
    - Why: Prevent cascading failures if a downstream service is slow or unavailable
    - Config:
        ```
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
        .failureRateThreshold(50) // open if >50% calls fail
        .waitDurationInOpenState(Duration.ofSeconds(30))
        .slidingWindowSize(20)
        .build();
        ```
    - Flow:
        - Circuit opens if failure threshold exceeded
        - While open → fallback method is called immediately (no waiting for downstream)
        - After wait duration → moves to half-open state to test downstream health

2. Fallback Strategies:
    - For internal service calls: Return cached or default data
    - For Kafka: Queue the request for later retry or move to DLT if needed
        ```
        @CircuitBreaker(name = "subscriptionService", fallbackMethod = "fallbackSubscriptions")
        public SubscriptionDTO getSubscriptions(Long customerId) {
            return feignClient.getSubscriptions(customerId);
        }
        ```
3. Rate Limiting (Bucket4j)
    - Why: Prevent abuse and control load during peak times
    - Per-client limit: e.g., 100 requests/min
    - Config in Filter:
        ```
        Bucket bucket = Bucket4j.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build();
        ```
    - If the bucket is empty → return 429 Too Many Requests

4. Benefits
    - Circuit breakers → Reduced cascading failures & improved service stability
    - Rate limiting → Protected APIs from overuse, ensured fair usage among clients

5. Final Summary Line for Interview:
    - Yes — we used Resilience4j circuit breakers to protect against slow/failing services and Bucket4j rate limiting to prevent abuse and control API load during high-traffic periods.

### Database & Data Design

#### Q32: Did each service have its own DB or shared DB? Why?
**Answer:** Each microservice had its own dedicated database schema — no shared DB across services. This followed the Database per Service principle so that each service owned its own data, could evolve independently, and avoided tight coupling.

1. Why Database Per Service?
    - Loose coupling → Services don’t break each other with schema changes
    - Independent scaling → Each DB tuned for that service’s workload
    - Security isolation → A service can’t directly modify another service’s data
    - Easier migrations → Schema changes don’t require coordination with all teams

2. How We Shared Data Between Services?
    - No direct DB calls across services
    - Shared data via APIs or Kafka events
        - Example: Subscription Service publishes an event when a subscription is created → Billing Service consumes it and updates its own DB

3. Example Setup
    - Customer Service DB: MySQL schema customer_db
    - Subscription Service DB: PostgreSQL schema subscription_db
    - Billing Service DB: MySQL schema billing_db
    - Redis was used as a shared cache, but not as a primary DB

4. Handling Cross-Service Queries
    - Used API composition (calling multiple services and aggregating responses)
    - For analytics/reporting, used a read-only reporting DB populated from Kafka events

5. Benefits:
    - Prevented accidental data corruption between services
    - Allowed mixing SQL + NoSQL as per service needs
    - Enabled independent deployment and scaling

6. Final Summary Line for Interview:
    - Each service had its own DB schema to maintain independence, avoid tight coupling, and allow independent scaling.
    - Data sharing was done via APIs and Kafka events, never by direct DB access

#### Q33: How did you manage transactions within a microservice?
**Answer:** Within a microservice, we used Spring’s @Transactional annotation to manage transactions at the service layer.

This ensured that a set of DB operations either all succeeded or all rolled back in case of an exception.

We configured transactions as atomic, consistent, isolated, and durable (ACID) for relational databases.

1. Transaction Management Approach:
    - Declarative transactions via `@Transactional`
    - Placed on service layer methods, not controllers, so business logic was fully covered
        ```
        @Transactional
        public void assignIpToCustomer(Long customerId, String ip) {
            ipRepository.reserveIp(ip);
            customerRepository.updateIp(customerId, ip);
        }
        ```
2. Rollback Rules
    - Default: Rollback on runtime exceptions
    - Custom: Explicit rollback on checked exceptions using:
        ```
        @Transactional(rollbackFor = {CustomBusinessException.class})
        ```

3. Isolation Levels
    - Default: READ_COMMITTED
    - For certain operations, used SERIALIZABLE to prevent dirty reads and phantom reads

4. Connection & Resource Management
    - Relied on connection pooling via HikariCP
    - Transactions were tied to the same DB connection until committed/rolled back

5. Why We Didn’t Use Distributed Transactions:
    - Each microservice had its own DB, so transactions were local only
    - Cross-service consistency handled via Saga pattern with Kafka

6. Benefits:
    - Guaranteed data consistency within the service
    - No partial writes
    - Easier debugging when failures occurred mid-process

7. Final Summary Line for Interview:
    - We used Spring’s declarative `@Transactional` support at the service layer to ensure atomic operations within a single microservice’s DB, with rollback rules and proper isolation levels to maintain ACID properties.

#### Q34: What strategy did you use for pagination and filtering in large APIs?
**Answer:** For large datasets, we implemented server-side pagination using Spring Data JPA’s Pageable interface, combined with filter parameters in the API request.

This allowed us to return only the required subset of data while applying dynamic filters without overloading the DB or the network.

1. Pagination Approach
    - Used Pageable with `PageRequest.of(page, size, sort)`
    - Returned a Page<\T> object containing:
        - Current page data
        - Total elements
        - Total pages
        - Page size
    
        ```
        public Page<Customer> getCustomers(String region, Pageable pageable) {
            return customerRepository.findByRegion(region, pageable);
        }
        ```
2. API Example
    - Request: `GET /customers?region=APAC&page=0&size=20&sort=createdDate,desc`
    - Response: 
        ```
        {
            "content": [ ... 20 customer records ... ],
            "pageable": { "pageNumber": 0, "pageSize": 20 },
            "totalElements": 540,
            "totalPages": 27
        }
        ```

3. Filtering Strategy
    - Allowed filtering on multiple fields via query params:
        ```
        GET /subscriptions?status=ACTIVE&type=PREMIUM&ipRange=192.168.*
        ```
    - For complex filters, used Specification API with JPA Criteria Builder
        ```
        Specification<Subscription> spec = (root, query, cb) ->
         cb.equal(root.get("status"), "ACTIVE");
        ```
4. Performance Optimizations
    - Indexed frequently filtered columns (status, createdDate, region)
    - Limited size parameter to max 100 records per request
    - Used projection DTOs instead of full entities to reduce payload size
    - Cached common filter results in Redis for faster repeated queries

5. Benefits
    - Reduced DB load for large datasets
    - Improved API response time
    - Flexible filtering without client-side heavy lifting

6. Final Summary Line for Interview:
    - We implemented server-side pagination with Spring Data Pageable and dynamic filtering via query parameters, using JPA Specifications and indexes for performance, and capped page size to prevent over-fetching.

#### Q35: How did you map DTOs and entities — did you use ModelMapper or MapStruct?
**Answer:** We used MapStruct to map between DTOs and entities because it provides compile-time type checking, generates fast mapping code, and avoids the runtime reflection overhead of tools like ModelMapper. 

This gave us better performance and early error detection during compilation.

1. Why MapStruct Instead of ModelMapper?
    - MapStruct
        - Compile-time code generation
        - No reflection → better performance
        - Compile-time errors if mapping fields are missing
    - ModelMapper
        - Uses reflection at runtime
        - Slower for large-scale mapping
        - Harder to detect mapping issues until runtime

2. Example Mapping with MapStruct
    - Entity → DTO
        ```
        @Mapper(componentModel = "spring")
        public interface CustomerMapper {
            CustomerDTO toDto(Customer entity);
            Customer toEntity(CustomerDTO dto);
        }
        ```
    - Usage:
        ```
        CustomerDTO dto = customerMapper.toDto(customerEntity);
        ```
3. Handling Field Name Differences
    - code:
        ```
        @Mapping(source = "createdDate", target = "registrationDate")
        CustomerDTO toDto(Customer entity);
        ```
4. Nested Mapping:
    - MapStruct handled nested objects automatically if mappers were defined for them
        ```
        @Mapping(source = "addressEntity", target = "addressDto")
        CustomerDTO toDto(Customer entity);
        ```
5. Benefits:
    - Faster API responses due to zero reflection
    - Type safety at compile time
    - Cleaner service layer — mapping logic kept separate

6. Final Summary Line for Interview:
    - We used MapStruct for DTO ↔ Entity mapping to get compile-time safety, better performance, and maintainable code without runtime reflection overhead.

#### Q36. How did you validate DTOs before mapping them to entities?
**Answer:** We validated DTOs at the controller layer using Jakarta Bean Validation (JSR 380) annotations along with Spring Boot’s built-in `@Valid` support.

This ensured incoming data met business rules before mapping to entities, preventing invalid data from reaching the DB.

1. Validation at Controller Level
    - code:
        ```
        @PostMapping("/customers")
        public ResponseEntity<CustomerDTO> createCustomer(@Valid @RequestBody CustomerDTO customerDto) {
            return ResponseEntity.ok(customerService.createCustomer(customerDto));
        }
        ```
    - `@Valid` triggers validation automatically
    - If validation fails → Spring throws MethodArgumentNotValidException and returns 400 Bad Request

2. Example DTO Validation
    - code:
        ```
        public class CustomerDTO {
            @NotBlank(message = "Name is mandatory")
            private String name;

            @Email(message = "Invalid email format")
            private String email;

            @Pattern(regexp = "^\\d{10}$", message = "Phone must be 10 digits")
            private String phone;

            @PastOrPresent(message = "Date cannot be in the future")
            private LocalDate registrationDate;
        }
        ```
3. Custom Validations
    - For complex rules, created custom annotations:
        ```
        @Target({ ElementType.FIELD })
        @Retention(RetentionPolicy.RUNTIME)
        @Constraint(validatedBy = IpAddressValidator.class)
        public @interface ValidIpAddress {
            String message() default "Invalid IP address";
        }
        ```
4. Why Validate Before Mapping?
    - Stops bad data early — avoids DB errors or partial transactions
    - Keeps service layer clean — only valid objects are processed
    - Avoids unnecessary entity creation — saves CPU/memory
5. Benefits
    - Prevented invalid customer/subscription data from entering system
    - Consistent error messages for clients
    - Reduced debugging effort
6. Final Summary Line for Interview:
    - We validated DTOs using Bean Validation annotations and `@Valid` in controllers, plus custom validators for complex fields, ensuring only valid data was mapped to entities

#### Q37. How did you handle versioning of your APIs when DTOs changed?
**Answer:** We followed a URI-based versioning strategy (`/api/v1/..., /api/v2/...`) for major API changes, and handled minor DTO changes using backward-compatible fields.

This ensured existing clients continued working while new clients could use updated fields or endpoints.

1. URI-Based Versioning for Breaking Changes
    - Example:
        ```
        GET /api/v1/customers/{id}    → returns old DTO
        GET /api/v2/customers/{id}    → returns updated DTO
        ```
    - V2 might:
        - Add new fields
        - Change field names
        - Remove deprecated fields

2. Backward-Compatible Changes
    - If only adding fields:
        - Old clients ignored extra JSON fields
        - Example: Adding `customerType` to response DTO while keeping old fields intact
    - If renaming/removing fields:
        - Kept old field for a deprecation period, marked as `@Deprecated` in DTO

3. Example: Supporting Multiple Versions in Code
    - code: 
        ```
        @RestController
        @RequestMapping("/api/v2/customers")
        public class CustomerControllerV2 {
            @GetMapping("/{id}")
            public CustomerV2DTO getCustomer(@PathVariable Long id) {
                return customerMapperV2.toDto(customerService.getCustomer(id));
            }
        }
        ```
    - Separate DTO and mapper for each version (`CustomerDTOv1`, `CustomerDTOv2`)
4. Versioning in Inter-Service Calls
    - Internal service calls (via Feign/WebClient) also specified API version in URLs
    - This avoided mismatches between microservices during rollout
5. Benefits
    - Zero downtime for existing clients
    - Clear upgrade path to new API versions
    - Easy rollback if a new version introduced issues
6. Final Summary Line for Interview:
    - We used URI-based versioning for major API changes and backward-compatible fields for minor changes, maintaining parallel versions until all clients migrated.

#### Q38: How did you protect your DB from overloaded API calls?
**Answer:** We protected the DB from overload by controlling traffic at the API layer, caching frequent queries, and optimizing DB queries/indexes.

This ensured high performance even during peak loads.

1. Rate Limiting at API Layer:
    - Used Bucket4j to limit requests per client/IP
        ```
        Bucket bucket = Bucket4j.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build();
        ```
    - Prevented abusive or accidental high-volume calls from overwhelming the DB
2. Caching in Redis
    - Cached frequently requested data like:
        - Customer profiles
        - Active subscription details
    - Redis TTL ensured stale data was cleared periodically
    - On cache hit → no DB query executed
3. Pagination + Filtering
    - Forced server-side pagination for large datasets: `GET /customers?page=0&size=20`
    - Stopped clients from fetching entire datasets in one call
4. Optimized Queries and Indexes
    - Added DB indexes on high-traffic filter columns (`status`, `createdDate`)
    - Used projection DTOs to fetch only required fields
        ```
        @Query("SELECT new com.example.dto.CustomerDTO(c.id, c.name) FROM Customer c WHERE c.status = :status")
        List<CustomerDTO> findByStatus(String status);
        ```
5. Async Processing for Heavy Jobs:
    - Moved bulk operations (e.g., export reports) to Kafka-based async processing
    - Prevented long-running queries from blocking API calls
6. Monitoring & Alerts
    - Prometheus alerts if DB CPU usage or query time exceeded thresholds
    - Allowed proactive scaling or traffic throttling
7. Final Summary Line for Interview:
    - We protected the DB by rate limiting API calls, caching frequent queries in Redis, enforcing pagination, indexing key columns, and moving heavy operations to async Kafka processing.

#### Q39. How did you handle bulk data exports without overloading the DB?
**Answer:** We avoided running huge queries directly from the API thread.

Instead, we processed exports asynchronously in batches, cached intermediate results, and used optimized queries with pagination to keep DB load stable.

1. Asynchronous Processing
    - User requests an export → API publishes a Kafka event with export request details
    - A background consumer service processes the request without blocking the API
        ```
        kafkaTemplate.send("export-requests", exportRequest);
        ```
2. Batch Querying
    - Retrieved data in pages instead of one massive query:
        ```
        Page<Customer> page = customerRepository.findAll(PageRequest.of(pageNum, 500));
        ```
    - This kept DB memory and CPU usage stable
3. Streaming Large Data
    - For certain exports, used Spring Data JPA streaming:
        ```
        @QueryHints(value = @QueryHint(name = HINT_FETCH_SIZE, value = "1000"))
        Stream<Customer> streamAll();
        ```
    - Allowed processing rows sequentially without loading all into memory
4. File Storage & Download
    - Once batches were processed → Data written to CSV/Excel file in object storage (GCS/AWS S3)
    - API returned download URL once file was ready
5. DB Protection Measures
    - Applied read replicas for heavy exports
    - Added read-only DB user for export service to prevent accidental writes
6. Benefits
    - No DB spikes during exports
    - Users could request very large reports without timeouts
    - Export jobs were resilient — if a batch failed, only that batch was retried
7. Final Summary Line for Interview:
    - We handled bulk exports asynchronously using Kafka, processed data in batches with pagination, streamed large result sets, and stored final files in object storage for later download, ensuring stable DB performance.

### Deployment and Config (even without DevOps)

#### Q40: How were your microservices deployed — as JARs or containers?
**Answer:** We packaged each microservice as a Spring Boot fat JAR and then built Docker images from those JARs. 

This made deployments consistent across environments and ensured each service had all its dependencies bundled.

1. Build & Packaging
    - Each microservice was built with Maven → produced a runnable JAR: `mvn clean package`
    - Example output: `target/customer-service-1.0.0.jar`
2. Containerization:
    - Created a Dockerfile for each service:
        ```
        FROM openjdk:17-jdk-slim
        COPY target/customer-service-1.0.0.jar app.jar
        ENTRYPOINT ["java", "-jar", "/app.jar"]
        ```
    - Built image:
        ```
        docker build -t customer-service:1.0.0 .
        ```
3. Environment Configuration
    - Used Spring Profiles (`dev`, `qa`, `prod`) to load different configs:
        ```
        java -jar app.jar --spring.profiles.active=prod
        ```
    - Config values (DB URL, Kafka broker list, Redis host) were externalized in config server / env variables
4. Running Services
    - Each service ran in its own container:
        ```
        docker run -d -p 8080:8080 --name customer-service customer-service:1.0.0
        ```
    - Multiple services could run on the same machine without dependency conflicts
5. Benefits:
    - Environment consistency — same image ran in dev, QA, and prod
    - Easy rollback — just deploy previous image version
    - Dependency isolation — no library version clashes
6. Final Summary Line for Interview:
    - We built Spring Boot fat JARs for each microservice, containerized them with Docker, and ran them in isolated containers with externalized configurations per environment.

#### Q41: Where did you store environment-based configs (dev/prod)?
**Answer:** We stored environment-based configurations outside the code to follow the 12-factor app principle.

Configs were managed via Spring Cloud Config Server and environment variables, so the same JAR/container could run in dev, QA, or prod without changes.

1. Spring Cloud Config Server
    - Centralized storage for properties:
        - `application-dev.yml`
        - `application-qa.yml`
        - `application-prod.yml`
    - Config Server fetched configs from a Git repository
    - Each service loaded its config at startup based on `spring.profiles.active`
2. Environment Variables for Sensitive Data
    - Secrets like DB passwords, JWT keys, and Kafka credentials were not in Git
    - Injected as environment variables at runtime:
        ```
        export DB_PASSWORD=prod_secret
        docker run -e DB_PASSWORD=$DB_PASSWORD ...
        ```
    - Accessed in code with:
        ```
        @Value("${DB_PASSWORD}")
        private String dbPassword;
        ```
3. Profile-Specific Properties
    - Application YAML example:
        ```
        spring:
            profiles: dev
            datasource:
                url: jdbc:mysql://dev-db:3306/customerdb
        ---
        spring:
            profiles: prod
            datasource:
                url: jdbc:mysql://prod-db:3306/customerdb
        ```
4. Benefits:
    - Same artifact runs in all environments
    - No hard-coded credentials in code or Docker imag
    - Centralized management → easy to update without redeploying code
5. Final Summary Line for Interview:
    - We used Spring Cloud Config Server with Git-backed YAML files for environment-specific configs, and environment variables for sensitive data, ensuring the same artifact could be promoted across dev, QA, and prod without modification.

#### Q42: How did you manage secrets in properties files securely?
**Answer** We avoided storing secrets in plain text in properties files.

Instead, sensitive configs like DB passwords, JWT signing keys, and Kafka credentials were stored in encrypted form or injected from secure vaults/environment variables at runtime.

1. Spring Cloud Config + Encryption
    - Spring Cloud Config Server supported symmetric or asymmetric encryption
    - code:
        ```
        db:
            password: '{cipher}AQAAABBBBCCCCDDDD'
        ```
    - Config Server decrypted values before sending to the microservice
2. Environment Variables
    - Secrets provided at container startup:
        ```
        docker run -e DB_PASSWORD=prod_secret ...
        ```
    - Accessed in Spring Boot via
        ```
        @Value("${DB_PASSWORD}")
        private String dbPassword;
        ```
3. Secret Management Services
    - For highly sensitive secrets, used HashiCorp Vault (or could be AWS Secrets Manager / GCP Secret Manager)
    - Microservice fetched secrets at startup over HTTPS, authenticated with a Vault token
4. Avoiding Local Exposure
    - `.properties` and `.yml` files in Git contained placeholders, not actual secrets:
        ```
        db:
            password: ${DB_PASSWORD}
        ```
    - This ensured no accidental leak via Git commits
5. Benefits:
    - Prevented exposure of sensitive credentials in source control
    - Easy secret rotation without code changes
    - Centralized and controlled access to sensitive configs
6. Final Summary Line for Interview:
    - We kept secrets out of plain-text configs by using Spring Cloud Config encryption, environment variables, and Vault for runtime retrieval, ensuring no sensitive data was committed to Git

#### Q43: How did you validate that deployment was successful?
**Answer:** We validated deployments by running automated smoke tests, health checks, and log inspections immediately after deployment.

Only after all checks passed was the release marked as successful.

1. Application Health Checks:
    - Each service exposed `/actuator/health` endpoint
        ```
        GET http://service-url/actuator/health
        # Response: {"status": "UP"}
        ```
    - Verified:
        - DB connection
        - Kafka connectivity
        - Redis availability
2. Automated Smoke Tests
    - API smoke tests ran right after deployment
        - Checked key endpoints (`/customers`, `/subscriptions`)
        - Verified authentication flow with valid/invalid JWT
        - Tested core business flows end-to-end
    - Used Postman/Newman scripts for quick validation
3. Log & Metric Monitoring
    - Checked logs for
        - Startup errors
        - Kafka consumer/producer connectivity issues
    - Used Prometheus + Grafana to confirm:
        - Zero error spikes
        - Normal response times
4. Functional Sanity Testing
    - Manually tested one or two major flows in UI:
        - Create customer → Assign IP → Verify in subscription module
    - Ensured both new features and existing features worked

5. Rollback Criteria
    - If smoke tests or health checks failed, deployment was rolled back immediately to the last stable version
6. Final Summary Line for Interview:
    - We validated deployments with actuator health checks, automated smoke tests, log/metric monitoring, and quick functional sanity checks, rolling back if any critical issue was detected.

#### Q44. Did you have any config server or central management?
**Answer:** Yes, we used Spring Cloud Config Server as a central configuration management solution for all microservices.

It stored environment-specific properties (dev, QA, prod) in a Git repository and served them to services at startup.

This allowed us to change configurations without rebuilding or redeploying the services.

1. How It Worked:
    - Config Storage:
        - Stored `application-dev.yml`, `application-prod.yml` etc. in a private Git repo
        - Example entry:
            ```
            spring:
                datasource:
                    url: jdbc:mysql://prod-db:3306/customerdb
            ```
2. Fetching Config
    - Microservices pointed to the config server
        ```
        spring:
            cloud:
                config:
                    uri: http://config-server:8888
                    profile: prod
        ```
3. Runtime Refresh
    - Used Spring Actuator `/refresh` endpoint to update configs without restart
4. Sensitive Data Handling
    - Encrypted secrets in config server or injected via environment variables
5. Final One-Liner for Interview:
    - Yes, we had a Spring Cloud Config Server for central config management, with environment-specific YAMLs in Git and runtime refresh capability for quick, consistent updates across microservices