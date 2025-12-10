## 🔥 Sequence Diagram – Full Lifecycle (Spring + JPA + Hibernate)
**A**
    ```
    User → Controller → Service → @Transactional → EntityManager → Hibernate → DB
    ```

    ```
    participant User
    participant Controller
    participant Service
    participant TxInterceptor
    participant EntityManager
    participant Hibernate
    participant DB

    User -> Controller: HTTP Request
    Controller -> Service: call service()

    Service -> TxInterceptor: @Transactional detected

    TxInterceptor -> EntityManager: createProxy()
    EntityManager -> Hibernate: openSession()
    Hibernate -> DB: getConnection()

    Service -> EntityManager: persist/find/merge
    EntityManager -> Hibernate: queue operations
    Hibernate -> DB: (delayed SQL)

    TxInterceptor -> EntityManager: commit()
    EntityManager -> Hibernate: flush()
    Hibernate -> DB: execute SQL
    DB --> Hibernate: results
    Hibernate -> EntityManager: clear PC

    TxInterceptor -> EntityManager: close()
    EntityManager -> Hibernate: release resources

    Controller -> User: Response
    ```

### Lifecycle of SessionFactory / EntityManagerFactory
**A** Hibernate’s SessionFactory = JPA’s EntityManagerFactory.

Both represent the core heavyweight, thread-safe factory object used to create per-request sessions/entity managers
1. High-Level Lifecycle Diagram
    ```
    ┌───────────────────┐
    │   Application     │
    │   Startup         │
    └─────────┬─────────┘
            │
            ▼
    ┌─────────────────────────┐
    │ Build SessionFactory /  │
    │ EntityManagerFactory    │
    └─────────┬───────────────┘
            │
            ▼
    ┌─────────────────────────┐
    │  Create Session /       │
    │  EntityManager (per req)│
    └─────────┬───────────────┘
            │
            ▼
    ┌─────────────────────────┐
    │  Transaction Boundary   │
    │(Begin → Commit/Rollback)│
    └─────────┬───────────────┘
            │
            ▼
    ┌─────────────────────────┐
    │    Close Session /      │
    │    EntityManager        │
    └─────────┬───────────────┘
            │
            ▼
    ┌─────────────────────────┐
    │ Application Shutdown    │
    │ Close Factory           │
    └─────────────────────────┘
    ```

2. Detailed Lifecycle Steps
    - A Application Startup → Factory Creation (Heavyweight)
    - Hibernate:
        - Configuration loads hibernate.cfg.xml / properties
        - Metadata & mappings parsed
        - Database dialect chosen
        - Connection pool initialized
        - Second-level cache initialized
        - HQL parser prepared
        - SessionFactory built
     - JPA
        - `Persistence.createEntityManagerFactory()`
        - Reads `persistence.xml`
        - Loads entities via scanning
        - Creates provider-specific factory
        - Same things as Hibernate internally
    - Spring Boot
        - Auto-configures:
            - `LocalContainerEntityManagerFactoryBean`
            - `HibernateJpaVendorAdapter`
        - Creates EntityManagerFactory ONCE at app start
        - Factory objects are very heavy → only one per DB typically

    - B Request Comes → Create Session / EntityManager
    - Session (Hibernate):
        - Lightweight
        - Short-lived
        - Represents a unit-of-work
     - EntityManager (JPA):
        - Also short-lived
        - Per request / per transaction
    
    - Spring creates it automatically through @Transactional:
        ```
        Open EntityManager → Bind to thread → Start Transaction
        ```

    - C Transaction Boundary
        - When @Transactional starts:
            - Spring opens EM/Session
            - Begins transaction (Connection.setAutoCommit(false))
            - Enlists JDBC connection in ACID transaction
            - Persistence context created

        - During the transaction:
            - Dirty checking watches changes
            - Writes accumulate in first-level cache
            - SQL executed only:
                - on flush
                - on commit
                - or manually

    - D. Commit / Rollback
        - Commit:
            - Flush persistence context
            - Execute queued INSERT/UPDATE/DELETE
            - Commit JDBC transaction
            - Clear persistence context
        - Rollback:
            - Do not flush
            - Discard persistence context changes
            - Transaction rolled back on DB

    - E. Close Session / EntityManager
        - Spring does it automatically after method completes.
        - After this, lazy proxies can NO LONGER fetch data →
        - this is why you get LazyInitializationException.

    - F. Application Shutdown
        - Framework shuts down SessionFactory / EntityManagerFactory.
        - During shutdown:
            - All caches cleared
            - Connection pool closed
            - Factory destroyed
            - Services stopped

