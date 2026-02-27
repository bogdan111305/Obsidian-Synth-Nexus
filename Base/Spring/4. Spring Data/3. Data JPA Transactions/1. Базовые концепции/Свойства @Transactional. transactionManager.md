# Свойства @Transactional. transactionManager

## Обзор

Свойство `transactionManager` в аннотации `@Transactional` позволяет указать конкретный менеджер транзакций, который должен использоваться для данного метода или класса. Это особенно полезно в приложениях с множественными источниками данных или различными типами транзакций.

## Базовое использование

### Указание конкретного TransactionManager

```java
@Service
@Transactional(transactionManager = "jpaTransactionManager")
public class UserService {
    
    @Transactional(transactionManager = "jdbcTransactionManager")
    public void createUser(User user) {
        // Использует jdbcTransactionManager
        userRepository.save(user);
    }
    
    @Transactional(transactionManager = "jpaTransactionManager")
    public void updateUser(User user) {
        // Использует jpaTransactionManager
        userRepository.save(user);
    }
}
```

## Конфигурация множественных TransactionManager

### 1. JDBC TransactionManager

```java
@Configuration
public class JdbcTransactionConfig {
    
    @Bean("jdbcTransactionManager")
    public PlatformTransactionManager jdbcTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource);
        return tm;
    }
}
```

### 2. JPA TransactionManager

```java
@Configuration
public class JpaTransactionConfig {
    
    @Bean("jpaTransactionManager")
    public PlatformTransactionManager jpaTransactionManager(EntityManagerFactory emf) {
        JpaTransactionManager tm = new JpaTransactionManager();
        tm.setEntityManagerFactory(emf);
        return tm;
    }
}
```

### 3. Hibernate TransactionManager

```java
@Configuration
public class HibernateTransactionConfig {
    
    @Bean("hibernateTransactionManager")
    public PlatformTransactionManager hibernateTransactionManager(SessionFactory sessionFactory) {
        HibernateTransactionManager tm = new HibernateTransactionManager();
        tm.setSessionFactory(sessionFactory);
        return tm;
    }
}
```

### 4. Atomikos TransactionManager (для распределенных транзакций)

```java
@Configuration
public class AtomikosTransactionConfig {
    
    @Bean("atomikosTransactionManager")
    public PlatformTransactionManager atomikosTransactionManager() {
        UserTransactionManager utm = new UserTransactionManager();
        utm.setForceShutdown(false);
        return new JtaTransactionManager();
    }
}
```

## Сценарии использования

### 1. Разные типы данных

```java
@Service
public class MultiDataSourceService {
    
    @Autowired
    private UserRepository userRepository; // JPA
    
    @Autowired
    private LogRepository logRepository; // JDBC
    
    @Transactional(transactionManager = "jpaTransactionManager")
    public void createUser(User user) {
        // Использует JPA транзакции
        userRepository.save(user);
    }
    
    @Transactional(transactionManager = "jdbcTransactionManager")
    public void logActivity(String activity) {
        // Использует JDBC транзакции
        logRepository.save(new Log(activity));
    }
}
```

### 2. Разные базы данных

```java
@Service
public class MultiDatabaseService {
    
    @Autowired
    private UserRepository userRepository; // Основная БД
    
    @Autowired
    private AnalyticsRepository analyticsRepository; // Аналитическая БД
    
    @Transactional(transactionManager = "primaryTransactionManager")
    public void createUser(User user) {
        // Операции с основной БД
        userRepository.save(user);
    }
    
    @Transactional(transactionManager = "analyticsTransactionManager")
    public void saveAnalytics(Analytics analytics) {
        // Операции с аналитической БД
        analyticsRepository.save(analytics);
    }
}
```

### 3. Разные уровни изоляции

```java
@Service
public class IsolationLevelService {
    
    @Transactional(
        transactionManager = "readCommittedTransactionManager",
        isolation = Isolation.READ_COMMITTED
    )
    public void normalOperation() {
        // Обычные операции с READ_COMMITTED
    }
    
    @Transactional(
        transactionManager = "serializableTransactionManager",
        isolation = Isolation.SERIALIZABLE
    )
    public void criticalOperation() {
        // Критические операции с SERIALIZABLE
    }
}
```

### 4. Разные таймауты

```java
@Service
public class TimeoutService {
    
    @Transactional(
        transactionManager = "fastTransactionManager",
        timeout = 10
    )
    public void fastOperation() {
        // Быстрые операции с коротким таймаутом
    }
    
    @Transactional(
        transactionManager = "slowTransactionManager",
        timeout = 300
    )
    public void slowOperation() {
        // Медленные операции с длинным таймаутом
    }
}
```

## Конфигурация в Spring Boot

### Автоматическая конфигурация

Spring Boot автоматически создает `TransactionManager`:

```java
@Configuration
@EnableTransactionManagement
public class TransactionConfig {
    
    // Spring Boot автоматически создает DataSourceTransactionManager
    // если есть DataSource и нет других TransactionManager
}
```

### Ручная конфигурация

```java
@Configuration
@EnableTransactionManagement
public class ManualTransactionConfig {
    
    @Primary
    @Bean("primaryTransactionManager")
    public PlatformTransactionManager primaryTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource);
        return tm;
    }
    
    @Bean("secondaryTransactionManager")
    public PlatformTransactionManager secondaryTransactionManager(DataSource secondaryDataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(secondaryDataSource);
        return tm;
    }
}
```

### Конфигурация с профилями

```java
@Configuration
@Profile("development")
public class DevTransactionConfig {
    
    @Bean("devTransactionManager")
    public PlatformTransactionManager devTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource);
        return tm;
    }
}

@Configuration
@Profile("production")
public class ProdTransactionConfig {
    
    @Bean("prodTransactionManager")
    public PlatformTransactionManager prodTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource);
        tm.setValidateExistingTransaction(true);
        return tm;
    }
}
```

## Практические примеры

### 1. Микросервисная архитектура

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private NotificationService notificationService;
    
    @Transactional(transactionManager = "orderTransactionManager")
    public void createOrder(Order order) {
        // Основная логика заказа
        orderRepository.save(order);
        
        // Асинхронное уведомление (без транзакции)
        notificationService.sendNotification(order);
    }
}
```

### 2. Аудит и логирование

```java
@Service
public class AuditService {
    
    @Autowired
    private AuditRepository auditRepository;
    
    @Transactional(transactionManager = "auditTransactionManager")
    public void auditOperation(String operation, String userId) {
        // Аудит в отдельной БД
        Audit audit = new Audit(operation, userId, LocalDateTime.now());
        auditRepository.save(audit);
    }
}
```

### 3. Кэширование

```java
@Service
public class CacheService {
    
    @Autowired
    private CacheRepository cacheRepository;
    
    @Transactional(transactionManager = "cacheTransactionManager")
    public void updateCache(String key, String value) {
        // Операции с кэш-БД
        cacheRepository.save(new CacheEntry(key, value));
    }
}
```

### 4. Аналитика

```java
@Service
public class AnalyticsService {
    
    @Autowired
    private AnalyticsRepository analyticsRepository;
    
    @Transactional(transactionManager = "analyticsTransactionManager")
    public void trackEvent(String event, String userId) {
        // Аналитические данные в отдельной БД
        Analytics analytics = new Analytics(event, userId, LocalDateTime.now());
        analyticsRepository.save(analytics);
    }
}
```

## Обработка ошибок

### 1. Проверка доступности TransactionManager

```java
@Component
public class TransactionManagerChecker {
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkTransactionManagers(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        String[] transactionManagerNames = {
            "jpaTransactionManager",
            "jdbcTransactionManager",
            "hibernateTransactionManager"
        };
        
        for (String name : transactionManagerNames) {
            if (context.containsBean(name)) {
                log.info("TransactionManager '{}' is available", name);
            } else {
                log.warn("TransactionManager '{}' is not available", name);
            }
        }
    }
}
```

### 2. Fallback механизм

```java
@Service
public class FallbackService {
    
    @Transactional(transactionManager = "primaryTransactionManager")
    public void primaryOperation() {
        try {
            // Основная операция
        } catch (Exception e) {
            // Fallback к вторичному TransactionManager
            fallbackOperation();
        }
    }
    
    @Transactional(transactionManager = "secondaryTransactionManager")
    public void fallbackOperation() {
        // Вторичная операция
    }
}
```

## Лучшие практики

### 1. Именование TransactionManager

```java
// ✅ Хорошо - описательные имена
@Transactional(transactionManager = "userTransactionManager")
@Transactional(transactionManager = "auditTransactionManager")
@Transactional(transactionManager = "analyticsTransactionManager")

// ❌ Плохо - неясные имена
@Transactional(transactionManager = "tm1")
@Transactional(transactionManager = "tm2")
```

### 2. Группировка операций

```java
@Service
@Transactional(transactionManager = "userTransactionManager")
public class UserService {
    
    // Все методы используют один TransactionManager
    public void createUser(User user) {}
    public void updateUser(User user) {}
    public void deleteUser(Long id) {}
}

@Service
@Transactional(transactionManager = "auditTransactionManager")
public class AuditService {
    
    // Все методы используют один TransactionManager
    public void auditOperation(String operation) {}
    public void logActivity(String activity) {}
}
```

### 3. Документирование

```java
/**
 * Создает пользователя в основной БД.
 * Использует userTransactionManager для операций с пользователями.
 */
@Transactional(transactionManager = "userTransactionManager")
public User createUser(User user) {
    return userRepository.save(user);
}

/**
 * Логирует активность в аудит БД.
 * Использует auditTransactionManager для независимого аудита.
 */
@Transactional(transactionManager = "auditTransactionManager")
public void logActivity(String activity) {
    auditRepository.save(new Audit(activity));
}
```

### 4. Тестирование

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = {TestTransactionConfig.class})
class MultiTransactionManagerTest {
    
    @Test
    @Transactional(transactionManager = "testTransactionManager")
    void testWithSpecificTransactionManager() {
        // Тест с конкретным TransactionManager
    }
}

@Configuration
class TestTransactionConfig {
    
    @Bean("testTransactionManager")
    public PlatformTransactionManager testTransactionManager() {
        return new DataSourceTransactionManager(testDataSource());
    }
}
```

## Отладка и мониторинг

### 1. Логирование транзакций

```properties
# application.properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.transaction.interceptor=TRACE
```

### 2. Мониторинг TransactionManager

```java
@Component
public class TransactionManagerMonitor {
    
    @EventListener(ApplicationReadyEvent.class)
    public void monitorTransactionManagers(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        Map<String, PlatformTransactionManager> transactionManagers = 
            context.getBeansOfType(PlatformTransactionManager.class);
        
        log.info("Found {} TransactionManagers:", transactionManagers.size());
        transactionManagers.forEach((name, tm) -> {
            log.info("  - {}: {}", name, tm.getClass().getSimpleName());
        });
    }
}
```

### 3. Проверка настроек

```java
@Component
public class TransactionSettingsChecker {
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkTransactionSettings(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        // Проверяем настройки всех TransactionManager
        context.getBeansOfType(PlatformTransactionManager.class)
            .forEach((name, tm) -> {
                log.info("TransactionManager '{}' settings:", name);
                // Анализируем настройки
            });
    }
}
```

## Производительность

### 1. Выбор оптимального TransactionManager

```java
// Для простых операций
@Transactional(transactionManager = "simpleTransactionManager")
public void simpleOperation() {
    // Быстрые операции
}

// Для сложных операций
@Transactional(transactionManager = "complexTransactionManager")
public void complexOperation() {
    // Сложные операции с дополнительными настройками
}
```

### 2. Кэширование TransactionManager

```java
@Configuration
public class CachedTransactionConfig {
    
    @Bean("cachedTransactionManager")
    public PlatformTransactionManager cachedTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource);
        tm.setValidateExistingTransaction(false); // Улучшает производительность
        return tm;
    }
}
```

### 3. Мониторинг производительности

```java
@Component
public class TransactionPerformanceMonitor {
    
    @EventListener(ApplicationReadyEvent.class)
    public void setupPerformanceMonitoring(ApplicationReadyEvent event) {
        // Настройка мониторинга производительности транзакций
        // Метрики, алерты, логирование
    }
} 