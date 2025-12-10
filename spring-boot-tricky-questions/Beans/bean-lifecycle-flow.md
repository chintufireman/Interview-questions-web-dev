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

### DI Flow with All Important Classes (FULL CHAIN)
```
ApplicationContext.refresh()
      │
      ▼
BeanDefinitionReader → BeanDefinition
      │
      ▼
DefaultListableBeanFactory
      │
      ▼
getMergedLocalBeanDefinition()
      │
      ▼
RootBeanDefinition
      │
      ▼
SmartInstantiationAwareBeanPostProcessor
  ├ predictBeanType()
  ├ determineCandidateConstructors()
  └ getEarlyBeanReference()
      │
      ▼
createBeanInstance()
  └ ConstructorResolver
      │
      ▼
populateBean()
  ├ inject fields/setters
  └ resolveDependency()
       ├ AutowireCandidateResolver
       ├ qualifiers
       ├ generics
       └ scope/proxies
      │
      ▼
initializeBean()
  ├ BeanPostProcessor.beforeInit
  ├ init/afterPropertiesSet
  └ BeanPostProcessor.afterInit (AOP)
      │
      ▼
Singleton Cache
   ├ singletonObjects
   ├ earlySingletonObjects
   └ singletonFactories

```

### 🔥 Final complete ordered class list (memorize this)
```
ApplicationContext
↓
BeanDefinitionReader
↓
BeanDefinition
↓
DefaultListableBeanFactory
↓
MergedBeanDefinition (RootBeanDefinition)
↓
SmartInstantiationAwareBeanPostProcessor
↓
ConstructorResolver
↓
DefaultListableBeanFactory.resolveDependency()
↓
AutowireCandidateResolver
↓
BeanPostProcessor
↓
Proxying layer (AOP / Caching / Async)
↓
Singleton caches

```

## interview sentence for bean lifecycle
**Ans**: Spring loads bean definitions, applies BeanFactoryPostProcessors, registers BeanPostProcessors, instantiates beans, injects dependencies, runs initialization callbacks, and then post-processes (AOP). FactoryBean runs before DI to produce the actual bean object.