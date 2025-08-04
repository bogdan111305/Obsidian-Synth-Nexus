# BeanPostProcessor

BeanPostProcessor - это интерфейс Spring Framework, который позволяет вмешиваться в процесс создания и инициализации бинов. Это мощный механизм для кастомизации жизненного цикла бинов.

## Содержание

1. [Основы BeanPostProcessor](#основы-beanpostprocessor)
2. [Методы интерфейса](#методы-интерфейса)
3. [Порядок выполнения](#порядок-выполнения)
4. [Aware интерфейсы](#aware-интерфейсы)
5. [Практические примеры](#практические-примеры)
6. [Лучшие практики](#лучшие-практики)

## Основы BeanPostProcessor

BeanPostProcessor позволяет модифицировать бины на двух этапах их жизненного цикла:
- **До инициализации** - после создания экземпляра, но до вызова методов инициализации
- **После инициализации** - после всех методов инициализации

### Интерфейс BeanPostProcessor

```java
public interface BeanPostProcessor {
    
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
    
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

## Методы интерфейса

### postProcessBeforeInitialization

Вызывается **до** методов инициализации бина:
- `@PostConstruct`
- `InitializingBean.afterPropertiesSet()`
- `init-method` (XML)

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("Before initialization: " + beanName);
        
        // Здесь можно модифицировать бин
        if (bean instanceof UserService) {
            // Логика модификации
        }
        
        return bean;
    }
}
```

### postProcessAfterInitialization

Вызывается **после** всех методов инициализации:

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("After initialization: " + beanName);
        
        // Бин может быть уже проксирован
        if (bean instanceof UserService) {
            // Логика модификации
        }
        
        return bean;
    }
}
```

### Важные особенности

1. **Имя бина не изменяется** - всегда остается исходным
2. **В postProcessAfterInitialization** бин может быть уже проксирован
3. **Возвращаемое значение** - может быть модифицированный или новый объект

## Порядок выполнения

BeanPostProcessor выполняются в определенном порядке:

### 1. BeanPostProcessor с приоритетом

```java
@Component
public class HighPriorityBeanPostProcessor implements BeanPostProcessor, Ordered {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("High priority BPP - Before: " + beanName);
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("High priority BPP - After: " + beanName);
        return bean;
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

### 2. BeanPostProcessor с PriorityOrdered

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor, PriorityOrdered {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("Custom BPP - Before: " + beanName);
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("Custom BPP - After: " + beanName);
        return bean;
    }
    
    @Override
    public int getOrder() {
        return 100;
    }
}
```

### Приоритеты выполнения

1. **PriorityOrdered** - наивысший приоритет
2. **Ordered** - обычный приоритет
3. **Без приоритета** - выполняется в порядке регистрации

## Aware интерфейсы

Aware интерфейсы позволяют бинам получать доступ к специфичным объектам Spring.

### Основные Aware интерфейсы

```java
public interface ApplicationContextAware {
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}

public interface BeanFactoryAware {
    void setBeanFactory(BeanFactory beanFactory) throws BeansException;
}

public interface EnvironmentAware {
    void setEnvironment(Environment environment);
}

public interface ResourceLoaderAware {
    void setResourceLoader(ResourceLoader resourceLoader);
}
```

### ApplicationContextAwareProcessor

Spring использует специальный BeanPostProcessor для обработки Aware интерфейсов:

```java
@Override
@Nullable
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    if (!(bean instanceof EnvironmentAware ||
          bean instanceof EmbeddedValueResolverAware ||
          bean instanceof ResourceLoaderAware ||
          bean instanceof ApplicationEventPublisherAware ||
          bean instanceof MessageSourceAware ||
          bean instanceof ApplicationContextAware ||
          bean instanceof ApplicationStartupAware)) {
        return bean;
    }
    
    // Внедрение соответствующих зависимостей
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
    
    return bean;
}
```

### Пример использования Aware интерфейсов

```java
@Component
public class UserService implements ApplicationContextAware, EnvironmentAware {
    
    private ApplicationContext applicationContext;
    private Environment environment;
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
    
    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
    
    public void someMethod() {
        // Использование ApplicationContext
        UserRepository userRepository = applicationContext.getBean(UserRepository.class);
        
        // Использование Environment
        String profile = environment.getActiveProfiles()[0];
    }
}
```

## Практические примеры

### 1. Логирование создания бинов

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingBeanPostProcessor.class);
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        logger.info("Initializing bean: {}", beanName);
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        logger.info("Bean initialized: {}", beanName);
        return bean;
    }
}
```

### 2. Валидация бинов

```java
@Component
public class ValidationBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof Validatable) {
            Validatable validatable = (Validatable) bean;
            if (!validatable.isValid()) {
                throw new BeanInitializationException("Bean " + beanName + " is not valid");
            }
        }
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

### 3. Кастомизация прокси

```java
@Component
public class CustomProxyBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof UserService) {
            // Создание кастомного прокси
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                new CustomInvocationHandler(bean)
            );
        }
        return bean;
    }
    
    private static class CustomInvocationHandler implements InvocationHandler {
        private final Object target;
        
        public CustomInvocationHandler(Object target) {
            this.target = target;
        }
        
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("Before method: " + method.getName());
            Object result = method.invoke(target, args);
            System.out.println("After method: " + method.getName());
            return result;
        }
    }
}
```

### 4. Метрики и мониторинг

```java
@Component
public class MetricsBeanPostProcessor implements BeanPostProcessor {
    
    private final MeterRegistry meterRegistry;
    
    public MetricsBeanPostProcessor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof UserService) {
            return new MetricsUserService((UserService) bean, meterRegistry);
        }
        return bean;
    }
}
```

## Лучшие практики

### 1. Производительность

```java
@Component
public class EfficientBeanPostProcessor implements BeanPostProcessor {
    
    private final Set<String> processedBeans = new HashSet<>();
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // Обрабатываем только один раз
        if (processedBeans.add(beanName)) {
            // Логика обработки
        }
        return bean;
    }
}
```

### 2. Безопасность

```java
@Component
public class SafeBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        try {
            // Логика обработки
            return bean;
        } catch (Exception e) {
            throw new BeanInitializationException("Error processing bean: " + beanName, e);
        }
    }
}
```

### 3. Условная обработка

```java
@Component
@ConditionalOnProperty(name = "app.bean.postprocessor.enabled", havingValue = "true")
public class ConditionalBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // Обработка только при включенном свойстве
        return bean;
    }
}
```

## Связанные темы

- [BeanFactoryPostProcessor](./BeanFactoryPostProcessor.md)
- [Lifecycle Callbacks](./Lifecycle Callbacks.md)
- [Event Listeners](../2.2 Event Listeners/README.md)
- [IoC и Dependency Injection](../../1. Основы Spring/1.1 IoC и Dependency Injection/README.md) 