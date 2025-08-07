# Аннотация @Transactional и Cglib proxy

## Обзор

CGLIB (Code Generation Library) - это библиотека для создания прокси-объектов во время выполнения. Spring использует CGLIB для создания прокси, когда включена опция `proxyTargetClass = true` или когда класс не реализует интерфейсы.

## Механизм работы CGLIB прокси

### 1. Создание CGLIB прокси

```java
@Service
@Transactional
public class UserService {
    
    public void createUser(User user) {
        // Этот метод будет выполняться через CGLIB прокси
        userRepository.save(user);
    }
}
```

### 2. Структура CGLIB прокси

```java
// CGLIB создает прокси примерно так:
public class UserService$$EnhancerByCGLIB$$12345678 extends UserService {
    
    private UserService target; // Оригинальный объект
    private Callback[] callbacks; // Массив колбэков (интерцепторов)
    
    @Override
    public void createUser(User user) {
        // Получаем TransactionInterceptor из callbacks
        TransactionInterceptor interceptor = (TransactionInterceptor) callbacks[0];
        
        // Выполняем транзакционную логику
        MethodInvocation invocation = new MethodInvocation() {
            public Object proceed() {
                return super.createUser(user); // Вызов родительского метода
            }
        };
        
        return interceptor.invoke(invocation);
    }
}
```

## Настройка CGLIB прокси

### Включение CGLIB

```java
@Configuration
@EnableTransactionManagement(proxyTargetClass = true) // Включаем CGLIB
public class TransactionConfig {
    // Конфигурация
}
```

### Автоматическое включение

```java
@Service
@Transactional
public class UserService {
    // CGLIB будет использован автоматически, так как нет интерфейса
}
```

## Преимущества CGLIB прокси

### 1. Работа с классами без интерфейсов

```java
// ✅ CGLIB работает с любыми классами
@Service
@Transactional
public class UserService {
    // Не нужен интерфейс
}

// ✅ CGLIB работает с final классами (с ограничениями)
@Service
@Transactional
public final class FinalUserService {
    // Ограничения: методы не могут быть final
}
```

### 2. Лучшая производительность выполнения

```java
// CGLIB прокси быстрее выполняются, чем JDK прокси
@EnableTransactionManagement(proxyTargetClass = true)
public class PerformanceConfig {
    // Оптимизированная конфигурация
}
```

### 3. Поддержка protected методов

```java
@Service
@Transactional
public class UserService {
    
    public void publicMethod() {
        protectedMethod(); // Вызов через CGLIB прокси
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    protected void protectedMethod() {
        // CGLIB может перехватывать protected методы
    }
}
```

## Ограничения CGLIB

### 1. Final методы

```java
@Service
@Transactional
public class UserService {
    
    @Transactional
    public final void finalMethod() {
        // ❌ CGLIB НЕ может перехватывать final методы
        // Транзакция НЕ будет работать
    }
}
```

### 2. Private методы

```java
@Service
@Transactional
public class UserService {
    
    public void publicMethod() {
        privateMethod(); // ❌ CGLIB НЕ может перехватывать private методы
    }
    
    @Transactional
    private void privateMethod() {
        // Транзакция НЕ будет работать
    }
}
```

### 3. Конструкторы

```java
@Service
@Transactional
public class UserService {
    
    @Transactional
    public UserService() {
        // ❌ CGLIB НЕ может перехватывать конструкторы
        // Транзакция НЕ будет работать
    }
}
```

## Конфигурация CGLIB

### Настройка через аннотации

```java
@Configuration
@EnableTransactionManagement(
    proxyTargetClass = true,  // Использовать CGLIB
    mode = AdviceMode.PROXY,  // Режим прокси
    order = Ordered.LOWEST_PRECEDENCE  // Приоритет
)
public class CglibTransactionConfig {
    // Конфигурация
}
```

### Программная конфигурация

```java
@Configuration
public class CglibTransactionConfig {
    
    @Bean
    public TransactionInterceptor transactionInterceptor() {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionManager(transactionManager());
        
        // Настройка для CGLIB
        Properties attributes = new Properties();
        attributes.setProperty("*", "PROPAGATION_REQUIRED,ISOLATION_READ_COMMITTED");
        interceptor.setTransactionAttributes(attributes);
        
        return interceptor;
    }
    
    @Bean
    public BeanNameAutoProxyCreator autoProxyCreator() {
        BeanNameAutoProxyCreator creator = new BeanNameAutoProxyCreator();
        creator.setBeanNames("*Service");
        creator.setInterceptorNames("transactionInterceptor");
        creator.setProxyTargetClass(true); // Включаем CGLIB
        return creator;
    }
}
```

## Отладка CGLIB прокси

### Включение логирования

```properties
# application.properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.aop=DEBUG
logging.level.net.sf.cglib=DEBUG
```

### Проверка типа прокси

```java
@Component
public class CglibProxyChecker {
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkCglibProxies(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        UserService userService = context.getBean(UserService.class);
        
        System.out.println("UserService class: " + userService.getClass());
        System.out.println("Is CGLIB proxy: " + AopUtils.isCglibProxy(userService));
        System.out.println("Is JDK proxy: " + AopUtils.isJdkDynamicProxy(userService));
        System.out.println("Is AOP proxy: " + AopUtils.isAopProxy(userService));
        
        // Получаем целевой класс
        Class<?> targetClass = AopUtils.getTargetClass(userService);
        System.out.println("Target class: " + targetClass);
    }
}
```

## Производительность CGLIB

### Сравнение с JDK прокси

```java
// JDK Dynamic Proxy
@EnableTransactionManagement(proxyTargetClass = false)
public class JdkProxyConfig {
    // Быстрее создается, медленнее выполняется
}

// CGLIB Proxy
@EnableTransactionManagement(proxyTargetClass = true)
public class CglibProxyConfig {
    // Медленнее создается, быстрее выполняется
}
```

### Оптимизация CGLIB

```java
@Configuration
@EnableTransactionManagement(proxyTargetClass = true)
public class OptimizedCglibConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager() {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        
        // Оптимизации для CGLIB
        tm.setValidateExistingTransaction(false);
        tm.setTransactionSynchronization(AbstractPlatformTransactionManager.SYNCHRONIZATION_ON_ACTUAL_TRANSACTION);
        
        return tm;
    }
}
```

## Лучшие практики

### 1. Избегайте final методов

```java
// ❌ Плохо
@Service
@Transactional
public class UserService {
    
    @Transactional
    public final void finalMethod() {
        // Транзакция не будет работать
    }
}

// ✅ Хорошо
@Service
@Transactional
public class UserService {
    
    @Transactional
    public void nonFinalMethod() {
        // Транзакция будет работать
    }
}
```

### 2. Используйте public методы

```java
// ❌ Плохо
@Service
@Transactional
public class UserService {
    
    @Transactional
    private void privateMethod() {
        // Транзакция не будет работать
    }
}

// ✅ Хорошо
@Service
@Transactional
public class UserService {
    
    @Transactional
    public void publicMethod() {
        // Транзакция будет работать
    }
}
```

### 3. Правильная настройка

```java
// Для классов без интерфейсов
@EnableTransactionManagement(proxyTargetClass = true)
public class CglibConfig {
    // Используем CGLIB
}

// Для интерфейсов
@EnableTransactionManagement(proxyTargetClass = false)
public class JdkConfig {
    // Используем JDK Proxy
}
```

## Интеграция с другими аспектами

### Spring Security

```java
@Service
@Transactional
@PreAuthorize("hasRole('ADMIN')")
public class SecureUserService {
    
    public void createUser(User user) {
        // CGLIB прокси с транзакциями и безопасностью
    }
}
```

### Spring Cache

```java
@Service
@Transactional
@Cacheable("users")
public class CachedUserService {
    
    public User findUser(Long id) {
        // CGLIB прокси с транзакциями и кэшированием
    }
}
```

### Validation

```java
@Service
@Transactional
@Validated
public class ValidatedUserService {
    
    public void createUser(@Valid User user) {
        // CGLIB прокси с транзакциями и валидацией
    }
}
```

## Решение проблем

### Проблема с final классами

```java
// ❌ Проблема
@Service
@Transactional
public final class FinalUserService {
    // CGLIB не может создать прокси для final классов
}

// ✅ Решение
@Service
@Transactional
public class UserService {
    // Убираем final
}
```

### Проблема с final методами

```java
// ❌ Проблема
@Service
@Transactional
public class UserService {
    
    @Transactional
    public final void finalMethod() {
        // Транзакция не работает
    }
}

// ✅ Решение
@Service
@Transactional
public class UserService {
    
    @Transactional
    public void nonFinalMethod() {
        // Транзакция работает
    }
}
```

### Проблема с private методами

```java
// ❌ Проблема
@Service
@Transactional
public class UserService {
    
    public void publicMethod() {
        privateMethod();
    }
    
    @Transactional
    private void privateMethod() {
        // Транзакция не работает
    }
}

// ✅ Решение
@Service
@Transactional
public class UserService {
    
    public void publicMethod() {
        publicTransactionalMethod();
    }
    
    @Transactional
    public void publicTransactionalMethod() {
        // Транзакция работает
    }
}
```

## Мониторинг и профилирование

### Проверка производительности

```java
@Component
public class CglibPerformanceMonitor {
    
    @EventListener(ApplicationReadyEvent.class)
    public void monitorPerformance(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        // Измеряем время создания прокси
        long startTime = System.currentTimeMillis();
        UserService userService = context.getBean(UserService.class);
        long endTime = System.currentTimeMillis();
        
        System.out.println("CGLIB proxy creation time: " + (endTime - startTime) + "ms");
        
        // Измеряем время выполнения
        startTime = System.currentTimeMillis();
        userService.createUser(new User());
        endTime = System.currentTimeMillis();
        
        System.out.println("CGLIB proxy execution time: " + (endTime - startTime) + "ms");
    }
}
``` 