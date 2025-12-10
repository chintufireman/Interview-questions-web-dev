
### 1. BIT HARDEST: WHEN DOES TRANSACTION START?
**Ans:** 
1. Proxy intercepts method call
2. Proxy checks @Transactional metadata
3. Proxy begins transaction (using PlatformTransactionManager)
4. Proxy calls your actual method

### 2. What happens when one transactional method calls another with different propagation?
**Ans:**
1. example
    - code:
        ```
        @Transactional
        public void m1() {
            m2();
        }
        @Transactional(propagation = REQUIRES_NEW)
        public void m2() { }

        ```
2. You think:
    - m1: transaction 1
    - m2: REQUIRES_NEW → transaction 2 (suspended + new)
    - Correct only if:
        - m2() is invoked through proxy
    - But here it's internal call, so:
        - m2() is NOT proxied
        - REQUIRES_NEW is IGNORED
        - It runs inside m1()’s transaction
    - This is tricky as hell, so here’s the dry run
        - code:
            ```
            controller → proxy → m1()
            m1() → internal call → real m2()
            m2() bypasses proxy → REQUIRES_NEW ignored
            ```

### Does @Transactional work if placed on a controller method?
**ANs** Yes, it works — but NOT recommended because controllers shouldn't carry business logic

### How does @Transactional behave with Spring Security method-level interceptors?
**A** Which one runs first?
1. Security interceptor runs before transaction interceptor by default

### Why does @Transactional NOT work on private methods?
**Ans:**
1. Because proxies can only intercept public method calls coming from outside the bean.
    - Private methods = never proxied
    - Internal calls = never proxied

### What happens when a checked exception is thrown in a transactional method?
**Ans** 
1. By default → Spring DOES NOT rollback on checked exceptions
    - Must configure:
        ```
        @Transactional(rollbackFor = Exception.class)
        ```

### What if you catch the exception inside the method? Does rollback happen?
**Ans** NO rollback → because the exception never reaches the proxy

### Can you force rollback manually even if there's no exception?
**Ans** Yes: TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();

### What happens if a @Transactional method is marked readOnly = true but performs an insert/update?
**Ans** 
1. Spring will not prevent the write
2. Hibernate skip dirty-checking
3. DB still allows writes
4. Sometimes leads to weird behavior like update silently ignored

### What is the difference between REQUIRED and NESTED propagation?
**Ans** 
1. REQUIRED → uses outer transaction
2. NESTED → creates a savepoint, only works when DB supports savepoints (JDBC, not JPA)

### Explain the internal flow of TransactionInterceptor
**Ans** 
    ```
    Proxy → TransactionInterceptor → TransactionManager → begin → method → commit/rollback
    ```

### Does @Transactional work for asynchronous methods? (like @Async)
**Ans** NO, because @Async runs in a different thread → proxy-bound TX not propagated

### Why does @Transactional not work in constructors?
**Ans** Proxy not created until object instantiation finishes. Constructor calls are not proxied

### What happens if a transaction times out?
**ANs** TransactionTimedOutException

### Difference between Spring and JTA transactions
**Ans**
1. Spring TX works for single DB
2. JTA (XA) is distributed for multiple DBs / JMS / Microservices

### Can two separate @Transactional methods run in parallel in same thread?
**Ans** No.
Thread-bound transaction prevents nested parallel execution.

### What happens if you annotate an interface with @Transactional instead of class or method?
**A**
1. Works with JDK proxies
2. Fails with CGLIB unless configured
3. Tricky and many fail this question

### Does @Transactional open DB connection before method call?
**A** internally
1. Connection acquired lazily
2. At first DB operation via ORM. Not immediately at begin()

### Can you use multiple transaction managers in the same app?
**A** yes using:
```
@Qualifier("tm1")
@Transactional(transactionManager = "tm2")
```

### What happens if two transaction managers are used in same app but no qualifier is applied?
**A** Bean resolution conflict → app fails to start

### If an outer transaction uses REQUIRED and inner uses REQUIRES_NEW, and inner fails, does outer rollback?
**A**
1. Inner rollback does NOT rollback outer
2. Outer continues normally
3. Because REQUIRES_NEW is independent

### How does Spring detect transaction metadata?
**AN**  `TransactionAttributeSource`

### What is transaction synchronization?
**A** callbacks like:
1. before commit
2. after commit
3. after completion

### What if method is annotated with both @Transactional and @Async?
**A**
1. Transaction behavior is LOST because async changes thread
2. Unless using distributed transaction library

### Can a transaction span multiple threads?
**A** No — Spring TX is thread-bound

### Does rollback happen if a RuntimeException is thrown inside a CompletableFuture in a @Transactional method?
**A** No. Because that exception is in a different thread

### What is dirty checking and how does it relate to transactions?
**A**
1. Dirty checking decides what to update BEFORE commit
2. Tricky because many think commit writes everything — but Hibernate sends SQL before commit.

### How do transactions behave in WebFlux / Reactive Spring?
**A** Classic `@Transactional` does not apply. Reactive uses `TransactionalOperator`

### What happens if propagation = MANDATORY but no transaction exists?
**A** Spring throws exception immediately, before method body runs

### What happens if a transaction is opened, but your method never touches a database?
**A** Does Spring still commit / rollback anything?
1. Yes, TX begins and ends, even if no SQL executed — overhead

### Can the outer transaction see uncommitted data from the inner transaction?
**A**
1. (e.g., outer with REQUIRED, inner with REQUIRES_NEW)
2. (Trick: No. REQUIRES_NEW commits independently. Outer TX isolation applies.)

### How does flush() inside a @Transactional method affect rollback?
**A** (Trick: flush() sends SQL to DB but does NOT commit → can still rollback.)

### What happens if flush() fails inside a transaction?
**A** (Trick: Hibernate throws exception → Spring rolls back TX.)

### If a transaction has already started, can you change its isolation level?
**A** NO. Isolation level is decided at begin() time only

### Can @Transactional handle distributed transactions (2-phase commit)?
**A** Only if using JtaTransactionManager + XA drivers

### What happens if you annotate both CLASS and METHOD with @Transactional
**A** Method-level annotation overrides class-level

### What if a @Transactional method returns a Stream (JPA)
**A** Lazy stream uses open session → transaction closes early → LazyInitializationException

### Why do transactions often not work in tests annotated with @MockBean?
**A** Mock beans bypass proxies → TX annotations lost

### How does nested transaction behave if OUTER commits but INNER rolled back?
**A** Nested uses savepoints → inner rollback doesn't rollback outer

### What happens if an exception occurs in @PostConstruct inside a transactional bean?
**A** Proxy not created yet → @Transactional ignored inside @PostConstruct

### What happens if a @Transactional method throws an exception, but a finally block catches and swallows it?
**A** Spring sees no exception → commit happens

### Can you start multiple transactions inside a single @Transactional method
**A** No — transaction manager allows only 1 TX per thread

### What if you call a @Transactional(propagation = NEVER) method from inside a REQUIRED transaction?
**A** Immediate exception

### How does Spring decide if DATA needs to be written to DB on commit?
**A** Hibernate dirty checking + persistence context changes

### What if a bean has two methods
**A** 
1. methodA(): @Transactional(REQUIRED)
2. methodB(): @Transactional(NOT_SUPPORTED)
3. and methodA calls methodB internally
4. No effect — proxy is bypassed → NOT_SUPPORTED is ignored

### What happens if two @Transactional methods call each other recursively?
**A** Only the FIRST call enters proxy → rest are non-transactional

### What if a @Transactional method calls a REST API that internally touches DB in another microservice?
**A** Is this a distributed transaction?
1. No — Spring transactions do NOT cross services

### How does Spring manage rollback for WebFlux Mono/Flux pipelines if using @Transactional on reactive code?
**A** Classic @Transactional does not work. Need Reactor-aware TX operator

### If you open a JPA Query with flushMode = COMMIT, how does it affect @Transactional?
**A** Hibernate defers flush until commit → fewer DB roundtrips

### Does Spring commit the transaction before or after calling @TransactionalEventListener?
**A** Depending on phase = BEFORE_COMMIT, AFTER_COMMIT, AFTER_ROLLBACK