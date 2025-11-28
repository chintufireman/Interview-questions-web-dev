### INTERNAL CLASS HIERARCHY (IMPORTANT)
**Ans** Understanding this gives HUGE edge
1. Hierarchy:

    ```
    BeanFactory
    └─ ConfigurableBeanFactory
    └─ ListableBeanFactory
        └─ AutowireCapableBeanFactory
    ApplicationContext
    └─ ConfigurableApplicationContext
    └─ AbstractApplicationContext
        └─ GenericApplicationContext
        └─ AnnotationConfigApplicationContext
    ```

### HOW SPRING CREATES AND WIRES BEANS INTERNALLY
**Ans**
1. Scan Classes
    - Find @Component, @Service, @RestController
    - Stored as BeanDefinition
    - Important: NOTHING is created yet

2. BeanDefinition: A POJO object metadata
    - Bean class
    - Scope
    - Dependency list
    - Factory method
    - Constructor args
    - Init / destroy methods

3. BeanFactory creates beans (lazy)
    - Actual instantiation happens here

4. BeanPostProcessor modifies beans
    - AOP
    - @Autowired
    - @Transactional

### KEY INTERNAL CLASSES THAT CREATE BEANS
**Ans:** These 2 classes matter most
1. DefaultListableBeanFactory
    - Core DI engine
        - stores BeanDefinitions
        - resolves dependencies
        - manages scopes
        - singleton cache

2. AutowireCapableBeanFactory
    - Injects dependencies into target bean using reflection

### How DI Happens (Flow)
**Ans** 
1. Example class 
    ```
        @Service
        public class OrderService {
            @Autowired
            PaymentService payment;
        }
    ```
2. Spring steps
    - Step 1 — Create OrderService instance (constructor)
    - Step 2 — DI phase begins;
        - Class fields scanned
        - @Autowired found on payment
    - Step 3 — AutowireCapableBeanFactory searches dependency
        - Search by Type first (IMPORTANT)
        - PaymentService bean exists → inject
    - Step 4 — BeanPostProcessor
        - JDK proxy created if needed (transaction, cache, async)


### DEPENDENCY INJECTION TRAPS
**Ans**
1. By Type (default) — Dangerous in real projects
    ```
    @Autowired
    private PaymentService payment;
    ```
    Trap: If you add second implementation:
    ```
    @Service
    class UpiPaymentService implements PaymentService {}

    @Service
    class CardPaymentService implements PaymentService {}
    ```
    DI fails: `NoUniqueBeanDefinitionException`

    FIX: @Qualifier
    ```
    @Autowired
    @Qualifier("cardPaymentService")
    PaymentService service;
    ```
    OR on Bean Definition:
    ```
    @Service("card")
    class CardPaymentService implements PaymentService {}
    ```

2. By Name Injection Trap
    - Happens when no @Qualifier but multiple beans
    - AND field name matches bean name
        ```
        @Autowired
        PaymentService cardPaymentService;
        ```
        Spring resolves by name → silently injects
    - Why trap?
        - If someone renames bean
        - You get WRONG implementation
        - No compile-time safety
        - Runtime bug

### FIELD INJECTION = DI TRAP
**Ans** 
1. People love it but it’s dangerous
    ```
    @Autowired
    private Repo repo;
    ```
    - No immutability
    - No testability
    - Reflection based
    - Slow injection

2. Constructor Injection is KING
    ```
    @Service
    public class OrderService {

        private final PaymentService payment;

        public OrderService(PaymentService payment) {
            this.payment = payment;
        }
    }
    ```
    - No reflection
    - Immutable
    - Fail-fast
    - Better in tests
    - Works with final fields
    - Better for circular detection

### HOW DI IS CONTROLLED AT RUNTIME
**Ans**
1. Core class: `AbstractAutowireCapableBeanFactory`
    - This does:
    - create beans
    - populate properties
    - resolve collaborators
    - apply post-processors
    - initialize beans

    Runtime Steps (super important)
    ```
    createBean()
    ↳ createBeanInstance()
    ↳ populateBean()
    ↳ initializeBean()
        ↳ postProcessBeforeInitialization()
        ↳ invokeAwareInterfaces()
        ↳ invokeInitMethods()
        ↳ postProcessAfterInitialization()
    ↳ registerSingleton()
    ```

### BEAN LIFECYCLE (Low to High)
**Ans**
1. Construct object
    - new OrderService() — via reflection

2. Dependency injection
    - @Autowired fields + constructors

3. Bean Post Process Before Init
    - Example: `@Autowired` resolver

4. Init phase
    - @PostConstruct
    - InitializingBean.afterPropertiesSet()
    - custom init-method

5. Post Process After Init
    - Example: AOP Proxy creation

### HUGE TRAP: Proxy created AFTER your @PostConstruct
**Ans**
1. Example:
    ```
        @Service
        public class TxService {

            @Autowired
            Repo repo;

            @PostConstruct
            @Transactional  // ❗ does nothing
            public void init(){
                repo.save(...);
            }
        }
    ```
    WHY?
    Because @Transactional works only after bean is proxied.

    @Init happens before proxy wrapping.

2. Why @Transactional doesn’t work in @PostConstruct?
    - Because proxy not yet applied
    - TransactionInterceptor wraps target after init stage.

### ApplicationContext Refresh Traps
**Ans:** ApplicationContext.refresh() invokes:
1. load bean definitions
2. invoke BeanFactoryPostProcessors
3. instantiate singletons
4. register resolvers
5. create proxies

Note: @Scope("prototype") beans do NOT get post-processors on every request.

### Prototype scope injection trap
**Ans**
1. This breaks many devs
    ```
    @Service
    class A {
        @Autowired
        private B b;  // prototype
    }
    ```
2. A is singleton → B injected once → NOT prototype anymore
3. Fix using Provider
    - code:
        ```
        @Autowired
        private Provider<B> provider;
        ```
    - Each call: `provider.get();`
    - new instance


### BeanFactory VS ApplicationContext TRAP
**Ans**
1. When building custom framework or running CLI code:
    ```
    BeanFactory factory = new XmlBeanFactory(...);
    MyService s = factory.getBean(MyService.class);
    ```
2. Everything works EXCEPT:
    - `@Transactional`
    - `@Async`
    - `@Cacheable`
    - `@Validated`

3. Because No PostProcessors enabled
    - Correct:
        ```
        ApplicationContext ctx =
        new ClassPathXmlApplicationContext("app.xml");
        ```

        or

        ```
        new AnnotationConfigApplicationContext(AppConfig.class);
        ```

### EXPERIENCED DEV UNDERSTANDING
**Ans** Bean creation is TWO PHASE:
1. Phase 1:
    - Create raw object instance
    - (no proxy, no injection)
2. Phase 2:
    - Decorate it:
    - Inject dependencies
    - AOP wrap
    - Lifecycle callbacks

### Why does this code break AOP, @Async, caching, etc?
**Ans:**
1. 
    ```
    BeanFactory factory = new DefaultListableBeanFactory();
    factory.getBean(MyService.class);
    ```

2. Because BeanFactory = raw DI engine
3. NO post processors → NO proxies

### Which object creates the proxy?
**Ans**
1. BeanPostProcessor (usually from Spring AOP module)
2. Specifically:
    - `AbstractAutoProxyCreator`
    - `ProxyFactoryBean`
    - `AnnotationAwareAspectJAutoProxyCreator`

### Ambiguity traps: @Primary vs @Qualifier
**Ans**
1. Which wins?
    ```
    @Service
    @Primary
    class UpiPayment implements PaymentService {}

    @Service
    class CardPayment implements PaymentService {}
    ```

    and injection:

    ```
    @Autowired
    PaymentService payment;
    ```
    
    `@Primary wins.`

2. What if @Qualifier is used?
    ```
    @Autowired
    @Qualifier("cardPayment")
    PaymentService p;
    ```
    `@Qualifier` overrides `@Primary`

### Bean lifecycle ordering trap
**Ans**
1. constructor
2. @Autowired injection
3. BeanPostProcessor BEFORE init
4. @PostConstruct
5. BeanPostProcessor AFTER init

### DI by name + by type internal resolution
**Ans**
1. Order of dependency resolution:
    ```
    Type → Qualifier → Bean name fallback →
    Factory/Provider → @Primary
    ```
2. This order explains why injection sometimes “magically works”.

### Refresh + PostProcessor trap
**Ans**
1. When does AOP apply?
    - After:
    - dependencies injected
    - init callbacks completed
    - postProcessAfterInitialization
    - Therefore:
        - Anything inside constructor / @PostConstruct runs WITHOUT AOP.

### Final Boss: Lazy + Proxy + Scope trap
**Ans**
 1. What happens with this?   
    ```
    @Service
    @Lazy
    public class TaskService {
        @Async
        public void run(){}
    }
    ```
    - Async wraps Lazy → Bean never fully initialized → no thread pool → task never runs

2. Correct:
    Split beans:

    ```
    @Service
    public class TaskRunner {
        @Async
        public void asyncRun(){}
    }

    @Service
    @Lazy
    public class TaskService {
        @Autowired TaskRunner runner;

        public void run(){
            runner.asyncRun();
        }
    }
    ```


### HOW SPRING CREATES A SINGLETON BEAN
**Ans**
1. Pseudo lifecycle:
    ```
        create bean
        ↓
        put bean factory into singletonFactories
        ↓
        populate dependencies
        ↓
        move to earlySingletonObjects
        ↓
        apply post processors (proxy, AOP etc)
        ↓
        move to singletonObjects
    ```

    Key: Bean moves across caches

2. EXAMPLE — Circular Dependency
    - Two services:
        ```
        @Service
        class A { @Autowired B b; }

        @Service
        class B { @Autowired A a; }
        ```

        Every beginner thinks this will crash immediately.
        WRONG

    - Spring tries to rescue it using caches

3. Step-by-step (actual internal order):
    - Step 1: Create A
        - instantiation
        - constructor only
        - NO injection yet
    - Spring stores:
        - `singletonFactories["A"] = ObjectFactory(A)`

    - Step 2: Populate dependencies
        - A needs B → not created yet
        - Spring creates B

    - Step 3: Create B
        - same: put B factory into singletonFactories
        - `singletonFactories["B"] = ObjectFactory(B)`

    - Step 4: Populate B dependencies
        - B needs A
        - Spring checks caches in order
        - Caches checked:

        - singletonObjects → (full bean) ❌
        - earlySingletonObjects → (half bean) ❌
        - singletonFactories → (factory!) ✔
        - Get A instance directly from factory
    - This is key: Spring returns early reference of A to B.

    - Step 5: Dependency wiring continues
        - Now B is completed
        - Move B to singletonObjects

    - Step 6: Go back to A
        - Now B exists → inject
        - A finishes
        - Move A to singletonObjects


### Classes involved (real Spring classes)