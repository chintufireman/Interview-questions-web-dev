### Hibernate, Spring ORM and JPA realtionship
**A**
1. Relationship Diagram (Hierarchy View)
    ```
    +--------------------------+
    |        Spring ORM        |
    |  (Integration Layer in   |
    |      Spring Framework)   |
    +------------+-------------+
                |
                | uses / manages
                v
    +--------------------------+
    |           JPA            |
    | (Specification / Rules)  |
    +------------+-------------+
                |
                | implemented by
                v
    +--------------------------+
    |        Hibernate         |
    |   (ORM Framework / JPA   |
    |       Implementation)    |
    +--------------------------+
    ```
2. Flow Diagram (How they work together):
    ```
            Your Spring Boot App
                |
                |  @Transactional, EntityManager, Repos
                v
        +------------------+
        |   Spring ORM     |
        |  (manages EM,    |
        |  sessions, txns) |
        +--------+---------+
                |
                |  Uses JPA API
                v
        +------------------+
        |       JPA        |
        | (interfaces,     |
        |  annotations)    |
        +--------+---------+
                |
                |  Implemented by
                v
        +------------------+
        |    Hibernate     |
        | (actual ORM:     |
        |  SQL, caching,   |
        |  dirty check)    |
        +------------------+

    ```

3. Simplest Visual Analogy Diagram:
    ```
    Spring ORM = Car Driver
    JPA        = Steering Rules
    Hibernate  = Engine
    ```
    ```
    Driver (Spring ORM)
        |
        v
    Steering Rules (JPA)
        |
        v
    Engine (Hibernate)
    ```

### Architecture
**A**
1. JPA Architecture (Specification Only): JPA defines interfaces, annotations & behavior, but does not implement ORM
    ```
    +-----------------------------------------------------+
    |                     JPA API                         |
    |-----------------------------------------------------|
    |  EntityManager (interface)                          |
    |  PersistenceContext                                  |
    |  JPQL / Criteria API                                 |
    |  Annotations: @Entity, @Id, @Table, @OneToMany ...   |
    +-----------------------------------------------------+
                        |
                        | implemented by
                        v
    +-----------------------------------------------------+
    |          JPA Provider (Hibernate / EclipseLink)     |
    +-----------------------------------------------------+
                        |
                        v
                Database (SQL)
    ```
    - key point: JPA = only a contract

2. Hibernate Architecture (Actual ORM Engine)
    ```
    Hibernate implements JPA + adds extra features
    +---------------------------------------------------------+
    |                     YOUR ENTITIES                       |
    |            (JPA annotations, Hibernate annotations)     |
    +---------------------------------------------------------+
                        |
                        v
    +---------------------------------------------------------+
    |                   Hibernate Core                        |
    |---------------------------------------------------------|
    |  SessionFactory / Session                               |
    |  HQL / Criteria                                          |
    |  TransactionManager                                      |
    |  Dirty Checking                                          |
    |  1st Level Cache                                         |
    |  2nd Level Cache (EHCache, Infinispan)                   |
    |  Lazy Loading                                            |
    |  JDBC Layer                                              |
    +---------------------------------------------------------+
                        |
                        v
    +---------------------------------------------------------+
    |                   Database (SQL)                        |
    +---------------------------------------------------------+
    ```
    - Hibernate = the real engine that creates SQL, manages sessions, caching, transactions, etc

3. Spring ORM Architecture (Integration Layer)
    ```
    Spring ORM does not perform ORM — it integrates Hibernate/JPA into the Spring ecosystem
    +--------------------------------------------------------------+
    |                        Spring Framework                      |
    |--------------------------------------------------------------|
    |  SPRING ORM MODULE                                           |
    |  - Integrates JPA/Hibernate                                  |
    |  - Creates EntityManagerFactory / SessionFactory             |
    |  - Manages Transactions (via @Transactional)                 |
    |  - Dependency Injection                                       |
    |  - Exception Translation (to DataAccessException)            |
    +--------------------------------------------------------------+
                        |
                        | uses / manages
                        v
    +--------------------------------------------------------------+
    |                            JPA API                           |
    +--------------------------------------------------------------+
                        |
                        | implemented by
                        v
    +--------------------------------------------------------------+
    |                          Hibernate                            |
    +--------------------------------------------------------------+
                        |
                        v
                    Database
    ```
    - Spring ORM = Adapter that lets Spring manage JPA/Hibernate easily

4. Combined Architecture (Complete Picture)
    ```
                        SPRING BOOT APPLICATION
                    (Services, Controllers, Repos)
                                |
                                v
    +---------------------------------------------------------+
    |                     Spring ORM                          |
    |       (Transaction Mgmt, Exception Translation, DI)     |
    +-----------------------------+---------------------------+
                                |
                                v
    +---------------------------------------------------------+
    |                          JPA API                        |
    |              (Interfaces + Annotations Only)            |
    +-----------------------------+---------------------------+
                                |
                                v
    +---------------------------------------------------------+
    |                      Hibernate Engine                    |
    |  (SQL Generation, SessionFactory, Caching, Dirty Check) |
    +-----------------------------+---------------------------+
                                |
                                v
    +---------------------------------------------------------+
    |                         Database                         |
    +---------------------------------------------------------+
    ```

### class-level archietecture
**A**
1. JPA Class-Level Architecture: Interfaces only — JPA does NOT implement anything
    ```
    +--------------------------------------------------+
    |                    JPA API                       |
    +--------------------------------------------------+
    | javax.persistence.EntityManagerFactory (IF)      |
    | javax.persistence.EntityManager (IF)             |
    | javax.persistence.Query (IF)                     |
    | javax.persistence.EntityTransaction (IF)         |
    | javax.persistence.PersistenceContext             |
    | javax.persistence.PersistenceUnit                |
    | javax.persistence.* Annotations (@Entity, etc.)  |
    | javax.persistence.spi.PersistenceProvider (IF)   |
    +--------------------------------------------------+
    ```
    - JPA = only interfaces + annotations. No logic.

2. Hibernate Class-Level Architecture
    ```
    Actual implementation of JPA + its own classes
    +------------------------------------------------------+
    |                 org.hibernate Package                |
    +------------------------------------------------------+
    | org.hibernate.SessionFactory (class)                 | -> JPA EMF implementation
    | org.hibernate.Session (class)                        | -> JPA EntityManager implementation
    | org.hibernate.Transaction (class)                    | -> JPA EntityTransaction implementation
    | org.hibernate.cfg.Configuration                      |
    | org.hibernate.service.ServiceRegistry                |
    | org.hibernate.query.QueryImplementor                 |
    | org.hibernate.HibernateException                     |
    +------------------------------------------------------+
                        | implements
                        v
    +------------------------------------------------------+
    |              JPA Provider Interface                  |
    |     javax.persistence.spi.PersistenceProvider        |
    +------------------------------------------------------+
    ```

    Hibernate JPA Implementation Classes (Important)

    ```
    HibernatePersistenceProvider
        implements javax.persistence.spi.PersistenceProvider

    EntityManagerFactoryImpl
        implements javax.persistence.EntityManagerFactory

    SessionImpl
        implements javax.persistence.EntityManager
    ```
    - Hibernate wraps its internal core (Session, Transaction) into JPA interfaces

3. Spring ORM Class-Level Architecture
    ```
    Spring ORM sits ABOVE JPA, not below
    +--------------------------------------------------------------+
    |                   org.springframework.orm                    |
    +--------------------------------------------------------------+
    | LocalContainerEntityManagerFactoryBean                       |
    |    -> Creates EntityManagerFactory                           |
    |    -> Integrates with Hibernate as JPA provider              |
    |--------------------------------------------------------------|
    | JpaVendorAdapter / HibernateJpaVendorAdapter                 |
    |    -> Auto-configures Hibernate properties                   |
    |--------------------------------------------------------------|
    | JpaTemplate (legacy)                                         |
    | HibernateTemplate (legacy)                                   |
    |--------------------------------------------------------------|
    | JpaTransactionManager                                         |
    | HibernateTransactionManager                                   |
    |    -> Manages @Transactional                                 |
    |--------------------------------------------------------------|
    | PersistenceExceptionTranslationPostProcessor (PETPP)          |
    |    -> Converts ORM exceptions to Spring DataAccessException  |
    +--------------------------------------------------------------+
    ```
4. Combined Class-Level Architecture (Complete Flow)
    ```
                            +-------------------------------+
                        |     Your Spring Boot App      |
                        +-------------------------------+
                        | @Service / @Repository        |
                        | @Transactional                |
                        +-------------------------------+
                                        |
                                        v
    +-------------------------------------------------------------------+
    |                         Spring ORM Layer                          |
    +-------------------------------------------------------------------+
    | LocalContainerEntityManagerFactoryBean                            |
    |     - Creates EntityManagerFactory                                |
    |     - Injects DataSource                                          |
    |                                                                   |
    | JpaTransactionManager (@Transactional handler)                    |
    |     - Starts/commits/rollbacks JPA transactions                   |
    |                                                                   |
    | PersistenceExceptionTranslationPostProcessor                      |
    |     - Converts HibernateException -> DataAccessException          |
    +-------------------------------------------------------------------+
                                        |
                                        v
    +-------------------------------------------------------------------+
    |                           JPA API Layer                           |
    +-------------------------------------------------------------------+
    | EntityManagerFactory (interface)                                  |
    | EntityManager (interface)                                         |
    | Query, CriteriaBuilder, Metamodel                                 |
    +-------------------------------------------------------------------+
                                        |
                                        | implemented by
                                        v
    +-------------------------------------------------------------------+
    |                     Hibernate Implementation Layer                |
    +-------------------------------------------------------------------+
    | HibernatePersistenceProvider                                      |
    | SessionFactory (EntityManagerFactoryImpl)                         |
    | Session (EntityManagerImpl)                                       |
    | Transaction (EntityTransaction impl)                              |
    | HQL / SQL Generator                                               |
    | Caching (L1, L2), Dirty Checking, Lazy Loading                    |
    +-------------------------------------------------------------------+
                                        |
                                        v
    +-------------------------------------------------------------------+
    |                           JDBC / Database                         |
    +-------------------------------------------------------------------+
    ```

### FULL REAL INTERNAL FLOW
**A**
1. STAGE 1: APPLICATION STARTUP — Spring Boot Auto-Configures JPA
    - Spring Boot scans dependencies
        - spring-boot-starter-data-jpa
        - hibernate-core
        - spring-orm
    - Auto-configuration kicks in (JPAAutoConfiguration)
        - Spring Boot automatically creates:
            ```
            | Component                                | From           |
            | ---------------------------------------- | -------------- |
            | `DataSource`                             | Spring Boot    |
            | `LocalContainerEntityManagerFactoryBean` | **Spring ORM** |
            | `HibernateJpaVendorAdapter`              | Spring ORM     |
            | `JpaTransactionManager`                  | Spring ORM     |
            | `EntityManagerFactory`                   | JPA            |
            | `EntityManager`                          | JPA            |
            | `SessionFactory` (internally)            | Hibernate      |
            ```
        - Hibernate starts
            - Loads entities (@Entity scanning)
            - Creates metadata
            - Validates schema
            - Builds SessionFactory
        
        - End Result of Startup: Your application is fully configured to use
            ```
            Spring Data JPA → Spring ORM → JPA → Hibernate → Database
            ```

2. STAGE 2: USER MAKES A REQUEST
    - Example: `GET /users/5`
    - Request hits Controller: `UserController.getUserById()`
    - Controller calls the Service: `userService.getUserById()`

3. STAGE 3: @Transactional + Spring ORM Activation
    - Service method is annotated with @Transactional
        ```
            @Transactional
            public User getUserById(Long id)
        ```
    - Spring AOP intercepts this method
        - `TransactionInterceptor` is triggered
    - Transaction begins
        - `JpaTransactionManager` is called
        - It opens or joins a transaction
        - Creates or fetches an `EntityManager`
    - EntityManager is bound to the current thread
        - Spring ORM binds it using: `TransactionSynchronizationManager`

4. STAGE 4: REPOSITORY EXECUTION (Main Work Happens Here)
    - Service calls Repository
        ```
        userRepository.findById(id)
        ```
    - Spring Data JPA creates a proxy for repository: 
        - Proxy delegates to: `SimpleJpaRepository`

    - SimpleJpaRepository uses EntityManager
        ```
        entityManager.find(User.class, id)
        ```

    - JPA delegates to Hibernate
        - EntityManager → Hibernate Session:
            ```
            entityManager.getDelegate() --> SessionImpl
            ```
    - Hibernate generates SQL
        - Parse JPQL / method name query
        - Build SQL AST
        - Create prepared statement
    
    - Example SQL: `select * from user where id=?`

    - Hibernate sends SQL through JDBC `PreparedStatement.executeQuery()`

    - Database returns result:
        - ResultSet → Hibernate
        - Hibernate populates your entity

    - First Level Cache: Hibernate stores the entity in Session cache (L1 cache)
    
    - Repository returns entity: User entity → Service method

5. STAGE 5: TRANSACTION COMPLETION
    - Service method ends — AOP returns to TransactionInterceptor
    - Spring decides to commit the transaction: `JpaTransactionManager.commit()`
    - Hibernate flushes changes
        - Dirty checking
        - Generate UPDATE/INSERT/DELETE if needed
        - Execute SQL

    - Hibernate commits the DB transaction: `connection.commit()`
    - EntityManager is closed / released
        - Spring cleans thread-bound EM
    - Control returns to Controller

6. COMPLETE FLOW DIAGRAM SUMMARY
    ```
    User Request
        ↓
    Controller
        ↓
    Service (@Transactional) ← Spring AOP starts Tx
        ↓
    JpaTransactionManager ← Spring ORM
        ↓
    EntityManager ← JPA
        ↓
    Session ← Hibernate
        ↓
    SQL → JDBC → Database
        ↓
    Results → Hibernate → EntityManager → Repository
        ↓
    Service (Business Logic)
        ↓
    Commit Transaction (flush, commit) ← Spring ORM
        ↓
    Controller Response
    ```
### JPA / Hibernate Entity Lifecycle States
**A** There are 4 entity lifecycle states
1. TRANSIENT (New Object)
    - Object is created in Java but Hibernate does not know about it
        ```
        User user = new User();  // TRANSIENT
        user.setName("Rahul");
        ```
    - Characteristics
        - No row in DB
        - No identifier (ID) or ID is null
        - Not associated with Session/EntityManager
        - Will NOT be saved automatically
        - Transition → Persistent
            - via: persist(), save(), merge(), flushing a cascade operation

2. PERSISTENT / MANAGED (Inside Persistence Context)
    - Object is now managed by Hibernate’s Persistence Context (L1 Cache).
        ```
        entityManager.persist(user);
        ```
    - Characteristics:
        - Has an ID
        - Any change you make is automatically tracked (dirty checking)
        - Will be saved in DB on flush or commit
    - Example: `user.setName("Updated Name");`
    - Hibernate will generate: UPDATE user SET name='Updated Name' during flush
    - How an entity becomes persistent?
        - persist(entity)
        - save(entity)
        - find()
        - getReference()
        - JPQL/HQL result
        - Automatically through cascading

3. DETACHED (Previously persistent, now unmanaged)
    - Entity was persistent earlier but is no longer associated with Persistence Context
    - Causes:
        - entityManager.clear()
        - entityManager.close()
        - Transaction ends (if not joined)
        - Manual detach
        - Serialization / sending across layers
            ```
            User user = entityManager.find(User.class, 1);
            entityManager.close(); // Now user is DETACHED
            ```
    - Characteristics:
        - Still has a database identity
        - Changes are NOT tracked
        - Updating fields does NOT trigger any SQL
        - To make it persistent again
            - merge(entity): (Merge copies state into a new managed instance)

4. REMOVED (Scheduled for deletion)
    - Entity is marked for deletion but not yet deleted until flush
        ```
        entityManager.remove(user);
        ```
    - Characteristics:
        - Still managed
        - Will be deleted from DB on flush or commit
        - After flush → transitions to DETACHED

### FULL ENTITY LIFECYCLE DIAGRAM (Text Version)
**A**
```
        new User()  
           |
           | persist(), save(), merge(), cascade
           v
  +-------------------+
  |   TRANSIENT       |
  +-------------------+
           |
           | persist/save
           v
  +-------------------+
  |   PERSISTENT      |
  +-------------------+
  |   (Managed)       |
  +-------------------+
     |     |      |
     |     |      |
     |     | remove() → REMOVED
     |     |
     | detach(), clear(), close()
     v
  +-------------------+
  |    DETACHED       |
  +-------------------+
           |
           | merge()
           v
         PERSISTENT

```
### Methods and Their Effects on Entity Lifecycle
**A**
Method|From State|To State|Meaning
---|---|---|---
`new`|-|Transient|Just a Java object
`persist()`|Transient|Persistent|Add object to persistence context
`save()` (Hibernate)|Transient|Persistent|Same as persist with immediate ID generation
`find()` / `get()`|-|Persistent|Loads from DB + managed
`merge()`|Detached|Persistent|Copies state into managed entity
`remove()`|Persistent|Removed|Marks for delete
`clear()`|Persistent|Detached|Detach all
`close()`|Persistent|Detached|Close session
`flush()`|Persistent|Persistent|Writes to DB but does NOT change state
Transaction commit|Persistent|Persistent|Flush + commit

JPA entities live in four states: transient (new object), persistent (managed by EntityManager), detached (no longer managed), and removed (marked for deletion). Hibernate moves entities between these states using operations like persist, merge, remove, detach, flush, and commit. Changes to persistent entities are tracked automatically and synchronized with the database during flush/commit

### JPA Entity Lifecycle Events (Callbacks)
**A**
1. These are special annotations that let you hook into the entity lifecycle and execute logic when an entity reaches specific states

2. Here are all official JPA events:
    ```
    | Event            | Trigger Time                    | Typical Use                              |
    | ---------------- | ------------------------------- | ---------------------------------------- |
    | **@PrePersist**  | Before INSERT                   | Set defaults, timestamps, audit fields   |
    | **@PostPersist** | After INSERT                    | Logging, sending notifications           |
    | **@PreUpdate**   | Before UPDATE                   | Update `updatedAt`, validation           |
    | **@PostUpdate**  | After UPDATE                    | Auditing, logging                        |
    | **@PreRemove**   | Before DELETE                   | Cleanup child records, validations       |
    | **@PostRemove**  | After DELETE                    | Logging                                  |
    | **@PostLoad**    | After entity is loaded (SELECT) | Initialize transient fields, format data |
    ```

3. Where to place callback methods?
    - Define inside entity class
        ```
        @Entity
        public class User {

            @PrePersist
            void beforeInsert() {
                System.out.println("PrePersist");
            }
        }
        ```
    - Use a separate Listener class
        ```
        @EntityListeners(UserEntityListener.class)
        @Entity
        public class User { ... }
        ```
        - Listener class:
            ```
            public class UserEntityListener {

                @PreUpdate
                public void onUpdate(User user) {
                    // logic
                }
            }
            ```
4. Real-world use cases
    - Audit fields
        ```
        @PrePersist
        public void prePersist() {
            createdAt = Instant.now();
        }

        @PreUpdate
        public void preUpdate() {
            updatedAt = Instant.now();
        }
        ```
    - Computed fields
        ```
        @PostLoad
            public void postLoad() {
                fullName = firstName + " " + lastName;
            }
        ```
### What flush does internally?
**A**
1. Performs dirty checking
    - Compares entity values with their snapshot

2. Generates SQL for: INSERT UPDATE DELETE

3. Executes these SQL statements using JDBC
    - BUT does NOT commit the transaction yet

    ```
    @Transactional
    public void updateUser() {
        User user = userRepository.findById(1L).get();
        user.setName("Rahul");   // dirty change

        // No SQL yet

        List<Order> orders = orderRepository.findAll();  
        // Flush happens here BEFORE SELECT!
    }
    ```
    - Why is flush triggered before the SELECT?
        - Because Hibernate must keep DB and persistence context consistent
        - It cannot run a SELECT while there are pending changes related to the data model

### When does Hibernate flush automatically?
**A**
1. At transaction commit
    ```
    @Transactional
    public void saveOrder() {
        orderRepository.save(order);  // no SQL
    } // flush → commit → SQL executes
    ```

2. Before executing JPQL/HQL queries
    - To ensure the query sees latest changes
        ```
        user.setName("Amit");
        entityManager.createQuery("from User").getResultList();
        // flush happens here before SELECT
        ```

3. When manually calling flush()
    - `entityManager.flush()`;

4. If flush mode is AUTO (default) and needed to maintain consistency
    - FlushModeType.AUTO → automatic flush based on situations

5. Flushing is Hibernate’s mechanism to synchronize in-memory entity changes with the database by executing SQL, without committing the transaction. It happens at commit time, before JPQL queries, or when explicitly triggered


### How flush works with batch inserts
**A**
1. How Hibernate normally inserts (without batching)
    ```
    save() → Hibernate builds INSERT SQL → flush → send SQL → DB  
    save() → flush → send SQL → DB  
    save() → flush → send SQL → DB  
    Each insert = one SQL → slow.
    ```
    - With batching ON → Hibernate delays SQL & flushes in chunks
    ```
    hibernate.jdbc.batch_size = 20
    ```
    - Hibernate will:
        - Store 20 insert operations in memory (Persistence Context / ActionQueue)
        - NOT send SQL immediately
        - When batch limit hits → flushes 20 INSERTS in one batch
        - Faster, Fewer DB round-trips, Massive performance boost

2. Persistence Context still holds the entities
    - Flush does NOT clear them
    - To avoid memory blow-up, you must call
        ```
        session.flush();
        session.clear();
        ```
    - Or using JPA:
        ```
        entityManager.flush();
        entityManager.clear();
        ```

3. Full Timeline of Batch Insert Flow
    ```
    save(entity1) → queued
    save(entity2) → queued
    ...
    save(entity20) → queue count = batch_size → flush triggered
        ↓
        Bind 20 rows into PreparedStatement
        ↓
        JDBC executeBatch()
        ↓
        DB receives all 20 INSERTs in one network call
        ↓
    Queue cleared
    ...
    Next 20 queued → flushed
    ...
    Transaction ends → final flush → commit → done
    ```

