
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
### Why does @Transactional NOT work on private methods?
**Ans:**
1. Because proxies can only intercept public method calls coming from outside the bean.
    - Private methods = never proxied
    - Internal calls = never proxied