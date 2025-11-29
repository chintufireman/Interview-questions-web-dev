## 🌑 SPRING DI: INTERNAL FLOW (From Application Start → Bean Fully Injected)
**Ans:**
1. Phase 1 — Spring Boot Starts
    ```
    User runs: SpringApplication.run(App.class)
          ↓
    SpringApplication creates ApplicationContext (Default: AnnotationConfigApplicationContext)
    ```
    - Key:
        - Boot picks ApplicationContext, not BeanFactory directly
        - This context wraps a DefaultListableBeanFactory (real DI engine)
    
2. Phase 2 — Scan + BeanDefinition Phase (NO OBJECTS YET)
    ```
    ApplicationContext
        ↓
    ComponentScan (from @SpringBootApplication)
        ↓
    Classpath scanning
        ↓
    BeanDefinition objects created (metadata only)
    ```
    - BeanDefinition = metadata
        - bean name
        - scope
        - class type
        - dependencies
        - autowire mode
        - lifecycle callbacks

3. Phase 3 — BeanFactoryPostProcessor kicks in
    ```
    BeanFactory
        ↓
    Load BeanDefinitions
        ↓
    Invoke BeanFactoryPostProcessor
    ```
    - Example:
        - ConfigurationClassPostProcessor
        - PropertySourcesPlaceholderConfigurer

    - They modify BeanDefinitions BEFORE bean creation:
        - Resolve @Configuration
        - Detect @Bean methods
        - Replace ${env} placeholders
        - Build proxy metadata

4. Phase 4 — Bean Creation Order
    ```
    Request Bean A
    ↓
    Check cache (singleton registry)
    ↓
    If not exist → create
    ```
    Creation steps:
    - 4.1 Instantiate: Constructor injection
    - Full recursive dependency graph resolution
        ```
        Constructor → create dependencies first
        ```
    - 4.2 Populate properties
        ```
        @Autowire field
        @Inject
        @Qualifier
        ```
    - 4.3 Apply awareness callbacks
        ```
        BeanNameAware
        BeanFactoryAware
        ApplicationContextAware
        ```
    - 4.4 Post-process BEFORE init: `BeanPostProcessor#postProcessBeforeInitialization`
        - Examples:
            - AOP proxy
            - Async proxy
            - Validation proxy
    - 4.5 Call Initialization
        - InitializingBean#afterPropertiesSet
        - @InitMethod
    
    - 4.6 Post-process AFTER init
        ```
        BeanPostProcessor#postProcessAfterInitialization
        ```
        - This is the final proxy creation moment
        - AOP proxies exist here. Never before

5. Phase 5 — Register in Cache
    - DefaultListableBeanFactory singletonObjects.put(beanName, beanInstance)

6. Phase 6 — Inject That Bean Into Others
    - Now any other bean that depends on it receives:
        - the actual instance
        - OR a proxy instance

7. FINAL LIFECYCLE MAP
    ```
    BeanDefinition → BeanFactoryPostProcessor → Instantiate → Populate →
    Aware → PreInitProcess → Init → PostInitProcess → Singleton Cache → READY
    ```