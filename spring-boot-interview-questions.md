### Q1 bean life cycle in detail?
**Answer**
1. Instantiation (Object Creation)
    - The Spring container creates an instance of the bean using the constructor (default or parameterized) or a factory method.
2. Populating Properties (Dependency Injection)

    - Spring injects dependencies into the bean (via constructor, setter injection, or field injection).

    - This is done based on the bean configuration in XML, Java Config, or Annotations (@Autowired, @Value, etc.).

3. Bean Post-Processing (Before Initialization)

    - The bean passes through BeanPostProcessor methods (postProcessBeforeInitialization()).

    - This allows for custom modifications before initialization.

4. Initialization (Custom Initialization Logic)

    - If the bean implements the InitializingBean interface, its afterPropertiesSet() method is called.
    
    - If the bean has an init-method defined in XML or is annotated with @PostConstruct, it is executed.

    - ```
        @PostConstruct
            public void init() {
            System.out.println("Custom Init Method Executed");
        }
        ```
5. Bean Post-Processing (After Initialization)

    - The bean passes through postProcessAfterInitialization() of the BeanPostProcessor.

    - This is useful for modifying the bean further after initialization.

6. Ready to Use

7. Destruction (Cleanup Before Removal)

    - When the Spring container shuts down, beans go through the destruction phase.
    - If the bean implements DisposableBean, the destroy() method is called.

    - If a destroy-method is configured or @PreDestroy annotation is used, it executes cleanup logic.

    - ```
        @PreDestroy
            public void cleanup() {
                System.out.println("PreDestroy Cleanup Executed");
            }
8. Full Bean Lifecycle Summary in Order

    - Bean Instantiation (Constructor or Factory Method)
    - Dependency Injection (Setter, Constructor, Field Injection)
    - postProcessBeforeInitialization() (BeanPostProcessor)
    - @PostConstruct or afterPropertiesSet() (Custom Initialization)
    - postProcessAfterInitialization() (BeanPostProcessor)
    - Bean Ready for Use
    - @PreDestroy or destroy() (Cleanup Before Shutdown)
9. java init configuration

    ```
            @Bean(initMethod = "init", destroyMethod = "cleanup")
            public MyBean myBean() {
                return new MyBean();
            }
    ```


