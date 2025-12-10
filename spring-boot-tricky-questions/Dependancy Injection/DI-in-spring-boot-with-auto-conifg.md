### 🔥 1. How does Spring Boot discover auto-configuration classes?
**Ans**: Expected Answer:
1. @EnableAutoConfiguration (inside @SpringBootApplication)
2. delegates to AutoConfigurationImportSelector
3. which loads class names from
    - (Boot 2.x) META-INF/spring.factories
    - (Boot 3.x) META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
4. These are candidate auto-configurations, not immediately applied
5. Deep Trap
    - Candidates ≠ beans yet.
    - They are metadata, not instantiated.

### 🔥 2. How does Spring decide which auto-configs to apply?
**Ans** Expected Answer:
1. Auto configurations are filtered using Condition Evaluation:
    - @ConditionalOnClass
    - @ConditionalOnBean
    - @ConditionalOnMissingBean
    - @ConditionalOnProperty
    - @ConditionalOnWebApplication
    - @ConditionalOnResource

2. The filtering happens before bean creation, during configuration class processing.

3. Important
    - Evaluation is done by:
        ```
        ConditionEvaluator
        ```
        `ConfigurationClassPostProcessor`

### 🔥 3. Where does auto-config live in the DI lifecycle?
**Ans** Correct Understanding: Auto configurations register @Bean definitions BEFORE your application beans.
1. Order:
    - Auto-config registers bean definitions
    - Component scanning registers bean definitions
    - DI wiring happens
    - BeanFactoryPostProcessor → BeanPostProcessor

2. Auto-config beans are FIRST-CLASS CITIZENS of DI graph.

### 🔥 4. Why does defining a custom bean override auto-config?
**Ans** Because most auto-config beans are wrapped with: `@ConditionalOnMissingBean`
1. example:
    ```
    @Bean
    @ConditionalOnMissingBean
    PlatformTransactionManager txManager(...)
    ```

2. If user declares a bean of the same type → auto-config definition is skipped, not overridden

3. Trap:
    - DI graph changes from:
        - Auto bean → user bean
    - Not from:
        - Auto bean replaced.

### 🔥 5. How does auto-config create environment-aware beans?
**Ans**: 
1. Through:
    ```
    @ConfigurationProperties
    Binder
    Environment
    PropertySource
    ```
2. Process:
    - Properties → resolved from environment
    - Bound to POJO (e.g., DataSourceProperties)
    - Used as constructor args into @Bean factory
    - Example:
        ```
        spring.datasource.*
        spring.jpa.*
        ```
    - Auto-config reads them → DI uses final POJO

### 🔥 6. How does Boot decide which DataSource to inject?
**Ans** Classpath Detection + Properties + Conditions
1. if
    - HikariCP on classpath
    - JPA present
    - no custom datasource bean
    - Boot applies: `DataSourceAutoConfiguration`
    - → creates HikariDataSource

2. If user writes: `@Bean DataSource custom()`
    - Auto-config:
        ```
        @ConditionalOnMissingBean(DataSource.class)
        ```
    - → SKIPS auto-instantiation.

### 🔥 7. Why does @Configuration proxying matter for auto-config DI?
**Ans** Spring CGLIB-proxies @Configuration classes.
1. Meaning: 
    - calling @Bean method returns same singleton instance
    - not a new object
2. If proxying disabled:
    - DI graph breaks (multiple bean instances)
        ```
        @Bean
        public CacheManager cacheManager() { return new CacheManager(); }
        ```
    - Calling internally from same class:
        - proxied → returns existing bean
        - not proxied → new object → circular bugs

### 🔥 8. How does Spring Boot avoid creating unused auto-config beans?
**Ans** It uses lazy bean creation, not eager instantiation.

1. Important:
    - Auto-configurations are registered early,
    - but beans inside them are created on demand.

2. Example:
    - You start a Web app but never use JPA:
        - JPA auto-config will be loaded
        - but EntityManager bean never instantiated

### 🔥 9. Explain circular dependencies through auto-config
**Ans** Auto-config beans can be part of DI graph like user beans.

1. Spring resolves singleton circular deps with
    - 3-level cache:
        - singletonFactories
        - earlySingletonObjects
        - singletonObjects

    - But many auto-configs use constructor injection.

    - Constructor injection circular loops cannot be resolved.

    - Example: 
        - Auto-config creates bean A via constructor →
        - Requires bean B →
        - User config creates bean B →
        - Requires bean A constructor → boom
    -Result:
        - BeanCurrentlyInCreationException

## ⭐ BONUS “S-Class” Answers Engineers Give
**Ans**: Spring Boot auto-config doesn’t create beans first—it creates bean definitions.
Conditions prune configs before the DI graph exists.
Actual objects are created lazily when dependencies demand them