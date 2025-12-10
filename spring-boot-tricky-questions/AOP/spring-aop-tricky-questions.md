### 1. What is the difference between Spring AOP and AspectJ?

**Answer**:

Spring AOP is proxy-based, meaning it creates proxies (usually dynamic proxies) for beans at runtime and applies advice at method level only.

AspectJ is a compile-time or load-time weaving framework that modifies bytecode — it can intercept not just method calls but also constructors, field access, and static blocks.

In short:

Feature|Spring AOP|AspectJ
---|---|---
Weaving|Runtime (proxy-based)|	Compile/load time
Join points|	Methods only|	Methods, fields, constructors
Performance|	Slightly slower|	Faster (compiled)

### ⚙️ 2. Can Spring AOP advise private or static methods?

**Answer**:
No ❌.
Spring AOP works using proxies (either JDK dynamic proxies or CGLIB).

JDK proxies only intercept public methods of interfaces.

CGLIB proxies subclass the target, so they can intercept public and protected methods — but not private, final, or static methods.
Thus, private/static methods cannot be advised.

### 🧠 3. What happens if you call a method annotated with @Transactional or @Around from inside the same class?

**Answer**:
The advice will not be applied 😮
This is because Spring AOP creates a proxy around the bean, and internal method calls (within the same class) don’t go through the proxy — they call the method directly (this.method()), bypassing AOP interception.

Example:
this.someTransactionalMethod() won’t trigger the transaction.
But calling it via the proxy bean (like myService.someTransactionalMethod()) will.

### 🔍 4. What is a Join Point vs a Pointcut in Spring AOP?

**Answer**:

Join Point → a specific execution point in the application (like a method call).

Pointcut → an expression that matches certain join points.

Think of Join Points as all possible “interception points”, and Pointcuts as “filters” selecting which ones to intercept.

### 💡 5. What are the different types of advice in Spring AOP?

**Answer**:

@Before — runs before the method execution.

@AfterReturning — runs after a method returns normally.

@AfterThrowing — runs when a method throws an exception.

@After — runs after method execution (regardless of outcome).

@Around — wraps the method execution; can modify inputs/outputs.

### ⚔️ 6. Why can’t Spring AOP be used for field-level interception?

**Answer**:
Because Spring AOP is proxy-based and proxies work at the method invocation level, not bytecode.
To intercept field access, you need AspectJ, which modifies bytecode during compile/load time.

### 🧩 7. How does Spring decide between JDK Dynamic Proxy and CGLIB Proxy?

**Answer**:

If the target class implements an interface, Spring AOP uses JDK Dynamic Proxy by default.

If the target class does not implement any interface, it falls back to CGLIB proxy.
You can force CGLIB using:

@EnableAspectJAutoProxy(proxyTargetClass = true)

### 🧵 8. What happens if two advices target the same join point?

**Answer**:
Spring applies them in order of precedence:

You can control this using the @Order annotation or Ordered interface.

Lower order value ⇒ higher precedence.
So @Order(1) runs before @Order(2).

### 🧱 9. Can you combine multiple advices on the same method?

**Answer**:
Yes ✅ — a single aspect can have multiple advices targeting the same pointcut (like @Before and @After), or multiple aspects can target the same join point (ordered via @Order).

### 🧨 10. What’s the lifecycle of an aspect bean in Spring?

**Answer**:
Aspect beans are singleton beans by default, managed by Spring’s IoC container.
Their advices are applied to proxied beans at context startup (when proxies are created).

### Why is @Around considered the most powerful advice in Spring AOP?
**Ans**: Because @Around can
    
1. control before and after execution
2. modify method arguments,
3. stop method execution completely
4. return its own custom result
5. handle exceptions
6. measure performance
7. Basically, @Around has full control over the method execution pipeline
    - `Object result = proceedingJoinPoint.proceed();`

### Why does the following NOT trigger AOP?
**Ans:** code
1. example:
    ```
    myService.someMethod();
    myService.someTransactionalMethod(); // both in same class

    ```
2. Internal calls bypass the proxy → AOP is not triggered
3. Proxies only intercept external method calls
    - Fix options:
        - Refactor to separate service class
        - Inject self-proxy (AopContext.currentProxy()) and call via proxy
        - Use AspectJ (not Spring AOP).

### Why does AOP not work on @Configuration classes?
**Ans:** Because @Configuration classes are CGLIB-enhanced already to support @Bean method proxying. Spring avoids double-proxying, so AOP is skipped. 

### What happens if an @Around advice does NOT call proceed()?
**Ans:** The target method will never execute.
1. code:
    ```
        @Around("execution(* com.app.*.*(..))")
        public Object block(ProceedingJoinPoint pjp) {
            return "Method execution prevented!";
        }

    ```
2. This completely swallows the original logic

### Difference between execution() and within() pointcuts?
**Ans:**
1. Matches methods:
    ```
    execution(* com.app.service.*.*(..))
    ```
2. within():
    ```
    within(com.app.service.*)
    ```
3. Tricky difference:
    - execution() matches inherited methods.
    - within() does NOT include inherited methods
    - This often surprises candidates

### How do you write a pointcut that matches methods with a specific annotation?
**Ans:** Example: intercept all methods with @MySecure
```
@Pointcut("@annotation(com.app.annotations.MySecure)")
public void secureMethods() {}

```
Or match classes annotated with:

```
@Pointcut("@within(com.app.annotations.Auditable)")
```

### What happens when multiple advices target the same join point? Who runs first?
**Ans** Order is decided by

1. @Order annotation
2. Ordered interface
3. Default — arbitrary order (not guaranteed)
    - Lower number → higher priority
    - @Order(1) runs before @Order(2).

### Why is an Aspect not triggered when calling a default method from an interface?
**Ans** Spring proxy intercepts interface methods, but default methods run directly, bypassing AOP.
