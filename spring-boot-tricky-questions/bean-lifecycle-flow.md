### Bean Lifecycle Order (Actual Sequence)
**Ans** 
1. If you can recite this, you are mid–senior instantly:
    ```
    1. Scan → BeanDefinition
    2. BeanFactoryPostProcessor
    3. Instantiate
    4. Dependency Injection
    5. Aware callbacks
    6. @PostConstruct / init()
    7. BeanPostProcessor (AOP here)
    8. Proxy created
    9. Put in Singleton Cache
    ```
2. Just memorize the separation:
    - Before Init = Real Bean
    - After Init = Proxy Bean

3. This one detail prevents 95% of your bugs