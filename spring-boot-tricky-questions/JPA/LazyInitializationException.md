## LazyInitializationException
**A**
1. Lazy Loading Works (Session OPEN)
    - No exception. Everything loads normally.
    ```
    sequenceDiagram
    autonumber
    participant Client as Client Code
    participant TX as @Transactional
    participant EM as EntityManager
    participant PC as Persistence Context
    participant Proxy as Child Proxy
    participant Child as Real Child
    participant DB as Database

    Client->>TX: call serviceMethod()
    TX->>EM: open transaction + create PersistenceContext

    Client->>EM: parent = em.find(Parent,100)
    EM->>DB: SELECT * FROM parent WHERE id=100
    DB->>EM: parent row
    EM->>Proxy: create proxy for child_id = 20

    Client->>parent: parent.getChild()
    parent->>Proxy: return proxy

    Client->>Proxy: proxy.getName()

    Proxy->>PC: is session open?
    PC->>Proxy: YES

    Proxy->>DB: SELECT * FROM child WHERE id=20
    DB->>Proxy: child row
    Proxy->>Child: hydrate real child

    Proxy->>Client: return child.getName()
    ```
2. LazyInitializationException (Session CLOSED)
    - This is the classic case interviewers expect you to explain
    - Reason:
        - Proxy needs the EntityManager/Session to load data
        - But the session is closed (outside @Transactional)
        - So proxy cannot fetch data → exception
    ```
    sequenceDiagram
    autonumber
    participant Client as Controller Layer
    participant Service as Service Method
    participant TX as @Transactional
    participant EM as EntityManager
    participant Proxy as Child Proxy

    Client->>Service: getParent()
    Service->>TX: enter @Transactional
    TX->>EM: open session

    Service->>EM: parent = find(Parent,100)
    EM->>Proxy: create lazy child proxy
    TX->>EM: commit & close session
    Service->>Client: return parent (session is now CLOSED)

    Client->>parent: parent.getChild().getName()
    parent->>Proxy: return proxy
    Client->>Proxy: proxy.getName()

    Proxy->>EM: try to fetch data (needs session)
    EM-->>Proxy: Session is closed

    Proxy-->>Client: LazyInitializationException
    ```

## Where exactly exception is thrown?
**A**
1. Accessing a lazy field outside @Transactional
    - Classic service → controller mistake.

2. JSON serialization (Jackson) tries to access lazy fields
    - @RestController without DTOs = boom 💥

3. Returning entities directly to the UI
    - Because controller runs after session is closed.
4. Using detached entities
    - Any proxy inside a detached entity will fail.

5. Lazy OneToOne sometimes fails even inside TX
    - Because Hibernate sometimes uses proxies incorrectly.

### ✔️ How to fix it? (Expected in interview)
**A**
1. Option 1 — Keep session open:
    - Add `@Transactional` at service layer

2. Option 2 — Use JOIN FETCH
    ```
    @Query("select p from Parent p join fetch p.child where p.id = :id")
    ```

3. Option 3 — Use DTO Projection
    - Best practice for REST APIs

4. Option 4 — Enable Open Session in View (Not recommended)