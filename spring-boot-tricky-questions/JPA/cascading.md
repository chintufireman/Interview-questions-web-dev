## Lifecycle callback order with cascading
**A**
Lifecycle Callback Order (with Cascading)
1. PERSIST Flow (EntityManager.persist / save)
When you call persist(parent) and cascading is enabled (e.g., cascade = CascadeType.PERSIST):
    - Order:
        - @PrePersist (parent)
        - @PrePersist (child) – only if PERSIST is cascaded
        - Insert statements for parent
        - Insert statements for child
        - @PostPersist (parent)
        - @PostPersist (child)
    - Key Notes:
        - Callbacks always trigger before and after the SQL, not during flush.
        - Order: parent first, then children for both pre and post events

2. LOAD Flow (Entity retrieved from DB)
    - Triggered when:
        - find()
        - getReference() (when initialized)
        - Lazy initialization
    - Order:
        - SQL SELECT
        - Entity hydration (fields populated)
        - @PostLoad (parent)
        - @PostLoad (child) – if child loaded due to fetch or lazy initialization

3. UPDATE / DIRTY CHECK Flow (Entity modified while managed)
    - When Hibernate flushes the Persistence Context:
        - Order
            - Dirty checking — Hibernate detects changes
            - @PreUpdate (parent)
            - SQL UPDATE (parent)
            - @PostUpdate (parent)
        - For cascades:
            - Only entities that changed trigger callbacks
            - When child is dirty: same sequence for child
        - Order with cascade
            - Parent changed → Child unchanged:
                - Only parent triggers @PreUpdate/@PostUpdate
            - Child changed → Parent unchanged:
                - Only child triggers callbacks

4. REMOVE Flow (EntityManager.remove / delete)
    - Order:
        - @PreRemove (parent)
        - @PreRemove (child) – only if REMOVE cascaded
        - SQL DELETE child
        - SQL DELETE parent
        - @PostRemove (parent)
        - @PostRemove (child)

    - Notes:
        - Delete order is child first, then parent
        - Callback order is still parent first, then child

5. DETACH Flow:
    - Triggered by:
        - `entityManager.detach(obj)`
        - `clear()`
        - entity goes out of persistence context due to transaction end (if not flushed)
    - Order:
        - @PreDetach
        - Entity removed from Persistence Context
        - @PostDetach
    - Cascade:
        - DETACH cascades similarly like persist/remove

### Sequence Diagram with Cascading (Simple)
```
persist(parent)
   |
   |-- @PrePersist(parent)
   |
   |-- cascade persist -> child
   |        |
   |        |-- @PrePersist(child)
   |
   |-- INSERT parent
   |-- INSERT child
   |
   |-- @PostPersist(parent)
   |-- @PostPersist(child)

```

### Important Real Interview Points
**A**
1. Cascading controls which entities get lifecycle events.
    - If you don’t enable cascade = PERSIST, child’s callbacks won’t run.
2. Order of callbacks is always parent → children
3. Order of SQL is always children → parent for delete
4. Updates only trigger events if Hibernate detects changes.
5. @PostLoad happens every time the entity is hydrated (even during lazy loading).