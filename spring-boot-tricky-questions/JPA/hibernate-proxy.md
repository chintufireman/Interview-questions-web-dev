## Hibernate proxies
**A**
1. What is a Hibernate Proxy? (Simple Answer): A Hibernate Proxy is a fake lightweight object created by Hibernate that
    - Extends your entity class
    - Contains only the ID
    - Loads the real entity data lazily, only when needed
    - It is used ONLY for LAZY loading of entities
    - Hibernate generates a subclass of your entity:
        - Example: `Child$HibernateProxy$v9df3s`

2. Why Hibernate Creates Proxies?
    - To avoid unnecessary SQL queries
    - Example:
        ```
        class Parent {
            @ManyToOne(fetch = FetchType.LAZY)
            private Child child;
        }
        ```
    - When Hibernate loads Parent:
        - It does NOT load Child
        - Instead, it creates a proxy for Child
    - This saves:
        - Time, Memory, SQL calls

3. Diagram: Proxy Creation & Lazy Loading
    ```
    sequenceDiagram
    autonumber
    participant EM as EntityManager
    participant HIB as Hibernate
    participant Parent as Parent Entity
    participant Proxy as Child Proxy
    participant Child as Real Child
    participant DB as Database

    Note over Parent: Parent Loaded
    EM->>HIB: load Parent
    HIB->>Parent: hydrate fields
    HIB->>Proxy: create proxy for Child (id only)

    Note over Proxy: PROXY created<br/>No actual data loaded

    Parent->>Proxy: getChild()

    Proxy->>HIB: intercept method call
    alt First access
        HIB->>DB: SELECT * FROM child WHERE id = ?
        DB->>HIB: return full child row
        HIB->>Child: hydrate real child
        Proxy->>Child: switch target
    end

    Proxy->>Parent: return actual data
    ```
4. What exactly does a Hibernate Proxy contain?
    - A proxy object includes
        - Only the primary key
        - A reference to the Session (to load data later)
        - ✔ A handler (LazyInitializer)
        - ✔ No real entity data
    - Example internal fields (conceptually):
        ```
        id = 10
        session = (Hibernate Session)
        lazyInitializer = (interceptor)
        target = null (until initialized)
        ```

5. When does proxy become the REAL entity?
    - During initialization, when the proxy loads actual data:
        ```
        Proxy -> intercepts call -> loads real entity -> replaces target
        ```
    - After initialization:
        - Proxy still remains proxy, but now contains all data
        - Target points to real object

6. Situations Where Proxies Fail (VERY important in interviews)
    - When @OneToOne uses a foreign key unique constraint
        - Hibernate often loads it EAGERLY → proxy DOESN’T work.
    - When the entity class is final
        - Cannot subclass → No proxy possible
    - When methods are final
        - Proxy cannot override → lazy loading fails
    - When accessing lazy field outside session
        - Causes LazyInitializationException:
            ```
            proxy.getName(); // Session is closed → Exception
            ```
    - When using equals() or hashCode() incorrectly
        - Comparing proxy with real entity can break logic
    - How to Check If an Object Is a Proxy
        - Hibernate API:
            ```
            Hibernate.isInitialized(entity);
            Hibernate.isPropertyInitialized(entity, "child");
            Hibernate.getClass(entity);
            ```
        - If proxy class ≠ actual class → it’s a proxy

    - Example of Proxy Class Name:
        ```
        com.example.Child$HibernateProxy$Vpf5s8
        ```
    - You can see this in logs:
        - System.out.println(child.getClass());

## Interview-Friendly Answer (Short Version)
**A** Hibernate Proxy is a subclass dynamically created by Hibernate to enable lazy loading of entities.
When a lazy association is accessed, the proxy intercepts the call, triggers a SQL query, loads the real entity, and then forwards the method invocation to the real object. Proxies only contain the primary key until they are initialized.
 
They fail when the entity class is final, methods are final, session is closed, or in some one-to-one mappings.