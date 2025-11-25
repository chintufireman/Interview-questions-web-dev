### 1. HTTP at real production scale
1. Not “GET vs POST”. You must know:
    - HTTP Keep Alive vs Close
    - HTTP pipelining
    - TCP slow start
    - Head-of-line blocking
    - HTTP/2 multiplexing
    - HTTP/3 QUIC (UDP-based)
    - Note: Why does HTTP/2 reduce latency but can still suffer Head-of-line blocking?



### 2. Request Lifecycle Internals (Important)
1. You should know how your request travels:

    - TCP Handshake

    - TLS Handshake
    - Reverse proxies (NGINX / Envoy)
    - Load balancer strategies (Least conn, Round-robin)
    - Application server → servlet container
    - Thread-per-request model
    - Question: What happens from browser request → until controller returns response?
    - If your answer doesn’t involve: TLS → LB → Reverse Proxy → Servlet Container → Worker pool → GC → DB → Response serialization You lose.






### 3. Servlet Container Internals (Nobody explains)

1. Especially for 5 YOE:

    - Tomcat thread pools

    - Max threads
    - Connection timeout
    - Request queue
    - Blocking IO vs Non-blocking IO
    - Undertow vs Netty
    - Interviewer will ask: Why your Spring Boot API times out when requests surge?

### 4. Thread starvation in Spring MVC

1. @Async default executor

2. Servlet container thread pool exhaustion

3. Deadlocks due to blocking DB calls

4. Why async + DB = disaster

### 5. Reactive Programming Internals (NO BS)
Not “Flux vs Mono”
1. Real depth:

    - Reactor Scheduler types

    - Backpressure strategies

    - Cold vs Hot streams

    - NIO event loop

    - Zero-copy networking

    - Netty pipeline

    - Why WebFlux avoids servlet stack

    - Real question: Why WebFlux is NOT always faster?



### 6. Serialization Performance

1. Every large backend deals with this.
Know:
    - Jackson vs Gson vs Kryo
    - Streaming parser vs object mapping
    - Avoiding reflection cost
    - Custom serializers
    - How to handle: Large JSON arrays (100k+)
    - Streaming APIs with Flux<\ServerSentEvent>

## 7. IO Blocking Hell

1. Deep concepts rarely anyone knows:

    - OS kernel buffers

    - Disk IO vs Network IO

    - Page cache

    - Zero-copy IO

    - MappedByteBuffer

    - Why Netflix moved to Netty?

### 8. API Latency Budgeting

1. Very important for 5 YOE.

    - P99 latency

    - Cold + warm requests

    - Queue wait time vs business logic

    - DB round trips count

    - Real question: How do you reduce response time from 500ms to 100ms?

### 9. HTTP Caching beyond basics

1. NOT browser cache. Know:

    - Surrogate keys

    - Cache invalidation strategies

    - ETag vs Last-Modified

    - Immutable caching

    - Cache guarantees

    - CDN caching (CloudFront, Akamai)

    - Edge caching

    - Interview question: How do you invalidate cache when only partial data changes?

### 10. Content Delivery Internals
1. Learn:

    - Prefetch

    - Brotli vs GZIP

    - Chunked encoding

    - Transfer-Encoding

    - Keep-alive timeout

    - Range requests

    - Cache-Control headers

### 11. JSON pitfalls that kill APIs

1. BigDecimal precision loss

2. Jackson polymorphic type handling

3. Float rounding errors

4. LocalDateTime timezone nightmares

5. ISO 8601 formats

6. Why timestamps break across microservices?

### 12. Authentication Internals (Industry-grade)

1. Not just JWT.

    - HMAC SHA256 signing

    - Key rotation (KMS)

    - Public/private key auth (RSA)

    - JTI tokens

    - Token replay attacks

    - OAuth2 grant weaknesses

    - PKCE flow

    - Practical question:Why refresh tokens are stored in DB but access tokens aren’t?



### 13. Distributed Transactions (This kills everyone)

1. 2PC vs 3PC

2. XA transactions

3. Outbox pattern

4. Saga choreography vs orchestration

5. Compensation logic

6. Idempotent consumers

7. Real question: User paid successfully but order service crashed — now what?

### 14. Message Ordering Hell

1. Kafka questions they’ll ask:

    - Partition key selection

    - Sticky partitioners

    - Why duplicates happen

    - Idempotent producer

    - Consumer offset commit strategies

### 15. DB Query Optimization at scale

1. NOT indexes only.

2. Covering index

3. Index-only scans

4. Bitmap index

5. Hash join vs nested loop

6. Query plans

7. VACUUM (Postgres)

8. Dead tuples

9. Real scenario: Your report API takes 8 minutes – debug step by step.

🔥 16. Distributed Cache Problems (Very deep)

Cache poisoning

Cache stampede

Negative caching

Write-through vs write-behind

Cache warming

Consistency vs availability

👉 Good interview question:

Why Redis causes stale reads in microservices?

🔥 17. API Gateway internals

Envoy filters

Rate limiting buckets

JWT offloading

Circuit breaker per route

Request batching

WebSockets tunneling

Shadow traffic

🔥 18. Real Production Failures

You absolutely must know:

cascading failure

thundering herd

memory leak due to heap pressure

slow GC

thread pool starvation

retry storms

N+1 network calls

🔥 19. Consistency Models

Interviewers LOVE this:

Strong

Eventual

Causal

Read-your-writes

Sessions consistency

👉 Real question:

Which consistency model does DynamoDB support?

🔥 20. Data Partitioning

Real backend topic:

Vertical vs Horizontal partitioning

Range partition

Hash partition

Consistent hashing

Hot partition problems

🔥 21. API Observability – 5 YOE Mandatory

Not just logs.

Request trace IDs

Span IDs

Baggage propagation

Red flags: log loops, sensitive PII

OpenTelemetry

Jaeger / Zipkin internals

🔥 22. Load Patterns

System design level:

Burst traffic

Seasonality

Black Friday load

Queue-based buffering

Elastic scaling

🔥 23. Deployment & Traffic shifting

Not only Docker.

Know:

Blue/Green

Canary

Shadow deploy

A/B test

Rolling upgrades

🔥 24. Failover Patterns

Multi region

Read replicas

Leader election

Split brain problem

CAP theorem real world

🔥 25. Security at infra level

Most backend devs don’t know:

mTLS between microservices

Vault secret rotation

CSRF tokens in SPAs

SSRF attacks on cloud metadata

Header security

CORS preflight