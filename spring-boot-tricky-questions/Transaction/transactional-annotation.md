### Explain internal working of @Transactional 
**Ans:**
1. flow:
    ```
    When Spring sees @Transactional, it wraps the bean in a proxy. 
    All method calls go through the proxy which delegates to 
    TransactionInterceptor. 
    This interceptor reads the transactional metadata and uses 
    PlatformTransactionManager to start, suspend, join, commit, or 
    rollback transactions. 
    Spring binds the DB connection to the thread, executes the method, 
    and then commits or rolls back based on exceptions. 
    If a method is called internally, proxy is bypassed and 
    transaction does not apply.
    ```