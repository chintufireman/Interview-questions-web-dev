## Hibernate Dirty Checking
**A** Dirty checking = Hibernate’s ability to automatically detect changes in an entity and generate the required SQL (UPDATE) at flush/commit time.

1. What is Dirty Checking?
    - When an entity is:
        - Loaded into the persistence context
        - Tracked inside the first-level cache
        - Modified by you in Java
    - Hibernate compares the snapshot vs current state and executes UPDATE only if something changed

2. How Dirty Checking Works Internally
    - When Hibernate loads an entity:
        ```
        Person{id=1, name="Alex", city="Mumbai"}
        ```
    - Hibernate stores:
        - Original snapshot → immutable copy
        - Managed entity → your live object
        - 📌 Snapshot stored in Persistence Context (1st-level cache)
        - 📌 Your entity stored as a reference

### INTERNAL MECHANISM (INTERVIEW DIAGRAM)
**A**
1. 
```
    ┌─────────────────────────────┐
    │ Persistent Context (1st LC) │
    ├─────────────────────────────┤
    │  Managed Entity Reference   │
    │     Person(name="Alex")     │
    ├─────────────────────────────┤
    │  Snapshot                   │
    │  Person(name="Alex")        │
    └─────────────────────────────┘
```
2. You modify:
    ```
    person.setCity("Pune");
    ```

    On flush():
    Hibernate performs:
    ```
    compare(snapshot, currentState)

    IF changes found → generate UPDATE SQL
    IF no changes → skip SQL
    ```

3. When Dirty Checking Runs?
    - It happens during
    - Flush
    - Before commit
    - When executing a query that requires DB sync
    - Manually calling entityManager.flush()

### Detailed Sequence (Method-Call Flow)
```
@Transaction Method Start
↓
Hibernate loads entity
↓
Snapshot stored
↓
You modify fields
↓
Flush triggered (auto or manual)
↓
HibernateDirtyChecker.compare(snapshot, entity)
↓
If changed → generate UPDATE
↓
Execute SQL → Commit
```

### When Dirty Checking FAILS (Tricky Interview Question)
**A**
1. You modify a detached entity
    - Example:
        ```
        entityManager.clear();
        person.setName("Rahul");  // No effect
        ```
    - Solution: use merge().

2. You change a field not mapped by JPA
    - Example:
        ```
        @Transient private int age;
        ```
    - Transient fields → not tracked

3. Collection changes without proper mapping
    - If missing:
        - mappedBy
        - cascade
        - join table entries
    - Hibernate may fail to detect changes

4. Bytecode Enhancement NOT enabled
    - Certain lazy attributes require bytecode enhancement.

5. You modify the object AFTER flush
    ```
    entityManager.flush();
    person.setCity("Delhi");
    ```
    - Won’t be saved unless flushed again.