# Lifecycle Callbacks

Lifecycle Callbacks - это механизм Spring Framework для выполнения кода на определенных этапах жизненного цикла бина. Это позволяет инициализировать ресурсы, освобождать память и выполнять другие операции.

## Содержание

1. [Основы Lifecycle Callbacks](#основы-lifecycle-callbacks)
2. [Initialization Callbacks](#initialization-callbacks)
3. [Destruction Callbacks](#destruction-callbacks)
4. [Порядок выполнения](#порядок-выполнения)
5. [Практические примеры](#практические-примеры)
6. [Лучшие практики](#лучшие-практики)

## Основы Lifecycle Callbacks

Spring предоставляет несколько способов выполнения кода на разных этапах жизненного цикла бина:

### Этапы жизненного цикла

1. **Создание экземпляра** - конструктор
2. **Внедрение зависимостей** - @Autowired, @Value
3. **Initialization Callbacks** - методы инициализации
4. **Использование бина** - обычная работа
5. **Destruction Callbacks** - методы уничтожения

## Initialization Callbacks

### 1. @PostConstruct

Аннотация JSR-250 для методов инициализации:

```java
@Component
public class UserService {
    
    @PostConstruct
    public void init() {
        System.out.println("UserService initialized with @PostConstruct");
        // Инициализация ресурсов
    }
}
```

### 2. InitializingBean.afterPropertiesSet()

Интерфейс Spring для инициализации:

```java
@Component
public class UserService implements InitializingBean {
    
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("UserService initialized with afterPropertiesSet()");
        // Инициализация после установки свойств
    }
}
```

### 3. init-method (XML)

Способ указания метода инициализации в XML:

```xml
<bean id="userService" class="com.example.UserService" init-method="init"/>
```

```java
public class UserService {
    
    public void init() {
        System.out.println("UserService initialized with init-method");
    }
}
```

## Destruction Callbacks

### 1. @PreDestroy

Аннотация JSR-250 для методов уничтожения:

```java
@Component
public class UserService {
    
    @PreDestroy
    public void cleanup() {
        System.out.println("UserService destroyed with @PreDestroy");
        // Освобождение ресурсов
    }
}
```

### 2. DisposableBean.destroy()

Интерфейс Spring для уничтожения:

```java
@Component
public class UserService implements DisposableBean {
    
    @Override
    public void destroy() throws Exception {
        System.out.println("UserService destroyed with destroy()");
        // Освобождение ресурсов
    }
}
```

### 3. destroy-method (XML)

Способ указания метода уничтожения в XML:

```xml
<bean id="userService" class="com.example.UserService" destroy-method="cleanup"/>
```

```java
public class UserService {
    
    public void cleanup() {
        System.out.println("UserService destroyed with destroy-method");
    }
}
```

## Порядок выполнения

### Initialization Callbacks

1. **@PostConstruct** - первый
2. **InitializingBean.afterPropertiesSet()** - второй
3. **init-method** - третий

### Destruction Callbacks

1. **@PreDestroy** - первый
2. **DisposableBean.destroy()** - второй
3. **destroy-method** - третий

### Важные особенности

- **Prototype бины** - destroy callbacks не вызываются
- **Singleton бины** - destroy callbacks вызываются при закрытии контекста
- **Множественные методы** - выполняются в указанном порядке

## Практические примеры

### 1. Инициализация ресурсов

```java
@Component
public class DatabaseConnectionService {
    
    private Connection connection;
    private final String url;
    private final String username;
    private final String password;
    
    public DatabaseConnectionService(
            @Value("${db.url}") String url,
            @Value("${db.username}") String username,
            @Value("${db.password}") String password) {
        this.url = url;
        this.username = username;
        this.password = password;
    }
    
    @PostConstruct
    public void init() throws SQLException {
        System.out.println("Initializing database connection...");
        this.connection = DriverManager.getConnection(url, username, password);
        System.out.println("Database connection established");
    }
    
    @PreDestroy
    public void cleanup() throws SQLException {
        System.out.println("Closing database connection...");
        if (connection != null && !connection.isClosed()) {
            connection.close();
        }
        System.out.println("Database connection closed");
    }
    
    public Connection getConnection() {
        return connection;
    }
}
```

### 2. Кэширование данных

```java
@Component
public class UserCacheService {
    
    private final Map<String, User> cache = new ConcurrentHashMap<>();
    private final UserRepository userRepository;
    
    public UserCacheService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @PostConstruct
    public void init() {
        System.out.println("Loading users into cache...");
        List<User> users = userRepository.findAll();
        users.forEach(user -> cache.put(user.getId(), user));
        System.out.println("Cache initialized with " + cache.size() + " users");
    }
    
    @PreDestroy
    public void cleanup() {
        System.out.println("Clearing cache...");
        cache.clear();
        System.out.println("Cache cleared");
    }
    
    public User getUser(String id) {
        return cache.get(id);
    }
}
```

### 3. Логирование жизненного цикла

```java
@Component
public class LifecycleLoggingService {
    
    private static final Logger logger = LoggerFactory.getLogger(LifecycleLoggingService.class);
    
    @PostConstruct
    public void init() {
        logger.info("LifecycleLoggingService initialized");
    }
    
    @PreDestroy
    public void cleanup() {
        logger.info("LifecycleLoggingService destroyed");
    }
}
```

### 4. Валидация конфигурации

```java
@Component
public class ConfigurationValidationService {
    
    private final Environment environment;
    private final List<String> requiredProperties;
    
    public ConfigurationValidationService(Environment environment) {
        this.environment = environment;
        this.requiredProperties = Arrays.asList("db.url", "db.username", "app.name");
    }
    
    @PostConstruct
    public void validateConfiguration() {
        System.out.println("Validating configuration...");
        
        for (String property : requiredProperties) {
            String value = environment.getProperty(property);
            if (value == null) {
                throw new IllegalStateException("Required property '" + property + "' is not set");
            }
        }
        
        System.out.println("Configuration validation passed");
    }
}
```

### 5. Регистрация в системе мониторинга

```java
@Component
public class MonitoringService {
    
    private final MeterRegistry meterRegistry;
    private final String serviceName;
    
    public MonitoringService(MeterRegistry meterRegistry, @Value("${app.name}") String serviceName) {
        this.meterRegistry = meterRegistry;
        this.serviceName = serviceName;
    }
    
    @PostConstruct
    public void registerMetrics() {
        System.out.println("Registering metrics for service: " + serviceName);
        
        // Регистрация метрик
        meterRegistry.gauge("service.status", 1);
        meterRegistry.counter("service.startups").increment();
    }
    
    @PreDestroy
    public void unregisterMetrics() {
        System.out.println("Unregistering metrics for service: " + serviceName);
        
        // Удаление метрик
        meterRegistry.gauge("service.status", 0);
    }
}
```

## Лучшие практики

### 1. Использование @PostConstruct для инициализации

```java
@Component
public class BestPracticeService {
    
    @PostConstruct
    public void init() {
        // Инициализация ресурсов
        // Валидация конфигурации
        // Регистрация в системах
    }
}
```

### 2. Использование @PreDestroy для очистки

```java
@Component
public class BestPracticeService {
    
    @PreDestroy
    public void cleanup() {
        // Закрытие соединений
        // Освобождение ресурсов
        // Отмена регистрации
    }
}
```

### 3. Обработка исключений

```java
@Component
public class SafeLifecycleService {
    
    @PostConstruct
    public void init() {
        try {
            // Инициализация
        } catch (Exception e) {
            throw new BeanInitializationException("Failed to initialize service", e);
        }
    }
    
    @PreDestroy
    public void cleanup() {
        try {
            // Очистка
        } catch (Exception e) {
            // Логируем ошибку, но не бросаем исключение
            System.err.println("Error during cleanup: " + e.getMessage());
        }
    }
}
```

### 4. Условная инициализация

```java
@Component
@ConditionalOnProperty(name = "app.feature.enabled", havingValue = "true")
public class ConditionalService {
    
    @PostConstruct
    public void init() {
        // Инициализация только при включенном свойстве
    }
}
```

### 5. Асинхронная инициализация

```java
@Component
public class AsyncInitService {
    
    @PostConstruct
    public void init() {
        // Запуск асинхронной инициализации
        CompletableFuture.runAsync(this::asyncInit);
    }
    
    private void asyncInit() {
        // Длительная инициализация
    }
}
```

## Связанные темы

- [BeanPostProcessor](./BeanPostProcessor.md)
- [BeanFactoryPostProcessor](./BeanFactoryPostProcessor.md)
- [IoC и Dependency Injection](../../1. Основы Spring/1.1 IoC и Dependency Injection/README.md)
- [Аннотации Spring](../../1. Основы Spring/1.2 Аннотации Spring/README.md) 