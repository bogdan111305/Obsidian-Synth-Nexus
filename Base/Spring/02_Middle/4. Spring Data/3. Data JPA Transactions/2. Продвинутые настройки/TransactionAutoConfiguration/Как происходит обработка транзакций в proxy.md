# Как происходит обработка транзакций в proxy

## Обзор

Spring использует механизм прокси (Proxy) для реализации транзакций. Когда вы помечаете метод аннотацией `@Transactional`, Spring создает прокси-объект, который перехватывает вызовы методов и управляет транзакциями.

## Механизм работы прокси

### 1. Создание прокси

```java
@Service
@Transactional
public class UserService {
    
    public void createUser(User user) {
        // Этот метод будет выполняться через прокси
        userRepository.save(user);
    }
}
```

### 2. Структура прокси

```java
// Spring создает прокси примерно так:
public class UserServiceProxy extends UserService {
    
    private UserService target; // Оригинальный объект
    private PlatformTransactionManager transactionManager;
    private TransactionInterceptor transactionInterceptor;
    
    @Override
    public void createUser(User user) {
        // Перехватываем вызов
        Method method = getMethod("createUser");
        
        // Выполняем транзакционную логику
        return transactionInterceptor.invoke(new MethodInvocation() {
            public Object proceed() {
                return target.createUser(user); // Вызов оригинального метода
            }
        });
    }
}
```

## Типы прокси

### 1. JDK Dynamic Proxy (по умолчанию)

```java
@Configuration
@EnableTransactionManagement(proxyTargetClass = false) // JDK Proxy
public class TransactionConfig {
    // Конфигурация
}

@Service
public class UserService implements UserServiceInterface {
    // Интерфейс необходим для JDK Proxy
}
```

### 2. CGLIB Proxy

```java
@Configuration
@EnableTransactionManagement(proxyTargetClass = true) // CGLIB Proxy
public class TransactionConfig {
    // Конфигурация
}

@Service
public class UserService {
    // Интерфейс не обязателен для CGLIB Proxy
}
```

## Жизненный цикл транзакции через прокси

### 1. Начало транзакции

```java
public class TransactionInterceptor {
    
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 1. Получаем настройки транзакции
        TransactionAttribute txAttr = getTransactionAttributeSource()
            .getTransactionAttribute(invocation.getMethod(), invocation.getThis().getClass());
        
        // 2. Получаем TransactionManager
        PlatformTransactionManager tm = determineTransactionManager(txAttr);
        
        // 3. Начинаем транзакцию
        TransactionStatus status = tm.getTransaction(txAttr);
        
        try {
            // 4. Выполняем бизнес-логику
            Object result = invocation.proceed();
            
            // 5. Коммитим транзакцию
            tm.commit(status);
            return result;
            
        } catch (Exception ex) {
            // 6. Откатываем транзакцию при исключении
            completeTransactionAfterThrowing(txAttr, status, ex);
            throw ex;
        }
    }
}
```

### 2. Обработка исключений

```java
protected void completeTransactionAfterThrowing(TransactionAttribute txAttr, 
                                             TransactionStatus status, 
                                             Throwable ex) {
    // Проверяем, нужно ли откатывать транзакцию
    if (txAttr.rollbackOn(ex)) {
        try {
            transactionManager.rollback(status);
        } catch (TransactionSystemException ex2) {
            // Логируем ошибку отката
        }
    } else {
        // Коммитим транзакцию несмотря на исключение
        try {
            transactionManager.commit(status);
        } catch (TransactionSystemException ex2) {
            // Логируем ошибку коммита
        }
    }
}
```

## Внутренние вызовы и прокси

### Проблема внутренних вызовов

```java
@Service
@Transactional
public class UserService {
    
    public void createUser(User user) {
        // Этот метод выполняется через прокси
        userRepository.save(user);
        
        // ❌ Проблема: этот вызов НЕ проходит через прокси
        updateUserStatistics(user);
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateUserStatistics(User user) {
        // Этот метод НЕ будет выполнен в новой транзакции
        // потому что вызов не проходит через прокси
    }
}
```

### Решение через самовызов

```java
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserService self; // Инжектируем прокси
    
    public void createUser(User user) {
        userRepository.save(user);
        
        // ✅ Решение: используем прокси для вызова
        self.updateUserStatistics(user);
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateUserStatistics(User user) {
        // Теперь этот метод выполнится в новой транзакции
    }
}
```

## Настройка прокси

### Конфигурация через аннотации

```java
@Configuration
@EnableTransactionManagement(
    proxyTargetClass = true,  // Использовать CGLIB
    mode = AdviceMode.PROXY,  // Режим прокси
    order = Ordered.LOWEST_PRECEDENCE  // Приоритет
)
public class TransactionConfig {
    // Конфигурация
}
```

### Программная конфигурация

```java
@Configuration
public class TransactionConfig {
    
    @Bean
    public TransactionInterceptor transactionInterceptor() {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionManager(transactionManager());
        
        // Настройка правил транзакций
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
        return creator;
    }
}
```

## Отладка прокси

### Включение логирования

```properties
# application.properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.aop=DEBUG
logging.level.org.springframework.orm.jpa=DEBUG
```

### Проверка типа прокси

```java
@Component
public class ProxyChecker {
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkProxies(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        UserService userService = context.getBean(UserService.class);
        
        System.out.println("UserService class: " + userService.getClass());
        System.out.println("Is proxy: " + AopUtils.isAopProxy(userService));
        System.out.println("Is CGLIB proxy: " + AopUtils.isCglibProxy(userService));
        System.out.println("Is JDK proxy: " + AopUtils.isJdkDynamicProxy(userService));
    }
}
```

## Производительность прокси

### Сравнение типов прокси

```java
// JDK Dynamic Proxy - быстрее создается, медленнее выполняется
@EnableTransactionManagement(proxyTargetClass = false)
public class JdkProxyConfig {
    // Подходит для интерфейсов
}

// CGLIB Proxy - медленнее создается, быстрее выполняется
@EnableTransactionManagement(proxyTargetClass = true)
public class CglibProxyConfig {
    // Подходит для классов без интерфейсов
}
```

### Оптимизация

```java
@Configuration
@EnableTransactionManagement(
    proxyTargetClass = true,  // Используем CGLIB для лучшей производительности
    mode = AdviceMode.PROXY
)
public class OptimizedTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager() {
        // Оптимизированный TransactionManager
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setValidateExistingTransaction(false); // Отключаем валидацию для производительности
        return tm;
    }
}
```

## Лучшие практики

### 1. Избегайте внутренних вызовов

```java
// ❌ Плохо
@Service
@Transactional
public class UserService {
    public void method1() {
        method2(); // Внутренний вызов
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void method2() {
        // Не будет выполнено в новой транзакции
    }
}

// ✅ Хорошо
@Service
@Transactional
public class UserService {
    @Autowired
    private UserService self;
    
    public void method1() {
        self.method2(); // Вызов через прокси
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void method2() {
        // Будет выполнено в новой транзакции
    }
}
```

### 2. Используйте правильный тип прокси

```java
// Для интерфейсов
@EnableTransactionManagement(proxyTargetClass = false)
public class InterfaceConfig {
    // JDK Proxy
}

// Для классов без интерфейсов
@EnableTransactionManagement(proxyTargetClass = true)
public class ClassConfig {
    // CGLIB Proxy
}
```

### 3. Минимизируйте количество прокси

```java
// ❌ Плохо - много прокси
@Service
@Transactional
@Cacheable
@Validated
public class UserService {
    // Множественные прокси могут снизить производительность
}

// ✅ Хорошо - объединяем функциональность
@Service
@Transactional
public class UserService {
    // Минимальное количество прокси
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
        // Метод выполняется через прокси с транзакциями и безопасностью
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
        // Метод выполняется через прокси с транзакциями и кэшированием
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
        // Метод выполняется через прокси с транзакциями и валидацией
    }
}
``` 