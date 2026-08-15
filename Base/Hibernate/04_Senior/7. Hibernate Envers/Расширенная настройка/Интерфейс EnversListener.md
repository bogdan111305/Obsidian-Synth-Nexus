# Интерфейс EnversListener

Интерфейс `EnversListener` предоставляет возможность кастомизации процесса аудита в Hibernate Envers. Это мощный инструмент для расширения функциональности аудита и интеграции с бизнес-логикой.

## Обзор интерфейса

```java
public interface EnversListener {
    void onEntityAdded(Object entity, Object revision);
    void onEntityModified(Object entity, Object revision);
    void onEntityDeleted(Object entity, Object revision);
    void onCollectionModified(Object entity, Object revision);
}
```

## Основные методы

### onEntityAdded

Вызывается при добавлении новой сущности в аудит.

```java
@Override
public void onEntityAdded(Object entity, Object revision) {
    if (entity instanceof User) {
        User user = (User) entity;
        log.info("Новый пользователь добавлен в аудит: {}", user.getUsername());
        
        // Дополнительная логика
        auditService.notifyUserCreation(user);
    }
}
```

### onEntityModified

Вызывается при изменении существующей сущности.

```java
@Override
public void onEntityModified(Object entity, Object revision) {
    if (entity instanceof Product) {
        Product product = (Product) entity;
        log.info("Продукт изменен: {}", product.getName());
        
        // Проверка критических изменений
        if (product.getPrice().compareTo(product.getPreviousPrice()) > 0) {
            notificationService.notifyPriceIncrease(product);
        }
    }
}
```

### onEntityDeleted

Вызывается при удалении сущности из аудита.

```java
@Override
public void onEntityDeleted(Object entity, Object revision) {
    if (entity instanceof Order) {
        Order order = (Order) entity;
        log.warn("Заказ удален из системы: {}", order.getId());
        
        // Архивирование данных
        archiveService.archiveOrder(order);
    }
}
```

### onCollectionModified

Вызывается при изменении коллекций в сущности.

```java
@Override
public void onCollectionModified(Object entity, Object revision) {
    if (entity instanceof Department) {
        Department dept = (Department) entity;
        log.info("Изменения в отделе: {}", dept.getName());
        
        // Обновление статистики
        statisticsService.updateDepartmentStats(dept);
    }
}
```

## Создание кастомного слушателя

### Базовый пример

```java
@Component
public class CustomEnversListener implements EnversListener {
    
    private static final Logger log = LoggerFactory.getLogger(CustomEnversListener.class);
    
    @Autowired
    private AuditNotificationService notificationService;
    
    @Override
    public void onEntityAdded(Object entity, Object revision) {
        log.info("Сущность добавлена в аудит: {}", entity.getClass().getSimpleName());
        
        // Отправка уведомления
        notificationService.sendAuditNotification("ADD", entity, revision);
    }
    
    @Override
    public void onEntityModified(Object entity, Object revision) {
        log.info("Сущность изменена: {}", entity.getClass().getSimpleName());
        
        // Проверка критических изменений
        checkCriticalChanges(entity);
    }
    
    @Override
    public void onEntityDeleted(Object entity, Object revision) {
        log.warn("Сущность удалена: {}", entity.getClass().getSimpleName());
        
        // Архивирование
        archiveDeletedEntity(entity);
    }
    
    @Override
    public void onCollectionModified(Object entity, Object revision) {
        log.info("Коллекция изменена в: {}", entity.getClass().getSimpleName());
    }
    
    private void checkCriticalChanges(Object entity) {
        // Логика проверки критических изменений
    }
    
    private void archiveDeletedEntity(Object entity) {
        // Логика архивирования
    }
}
```

### Слушатель с бизнес-логикой

```java
@Component
public class BusinessEnversListener implements EnversListener {
    
    @Autowired
    private EmailService emailService;
    
    @Autowired
    private AuditRepository auditRepository;
    
    @Override
    public void onEntityAdded(Object entity, Object revision) {
        if (entity instanceof Employee) {
            Employee employee = (Employee) entity;
            
            // Уведомление HR отдела
            emailService.sendNotification(
                "hr@company.com",
                "Новый сотрудник добавлен",
                "Сотрудник " + employee.getName() + " добавлен в систему"
            );
            
            // Сохранение дополнительной информации
            auditRepository.saveAuditInfo(employee, "EMPLOYEE_CREATED", revision);
        }
    }
    
    @Override
    public void onEntityModified(Object entity, Object revision) {
        if (entity instanceof Employee) {
            Employee employee = (Employee) entity;
            
            // Проверка изменения зарплаты
            if (employee.getSalaryChanged()) {
                emailService.sendNotification(
                    "finance@company.com",
                    "Изменение зарплаты",
                    "Зарплата сотрудника " + employee.getName() + " изменена"
                );
            }
        }
    }
    
    @Override
    public void onEntityDeleted(Object entity, Object revision) {
        if (entity instanceof Employee) {
            Employee employee = (Employee) entity;
            
            // Уведомление о увольнении
            emailService.sendNotification(
                "hr@company.com",
                "Сотрудник удален",
                "Сотрудник " + employee.getName() + " удален из системы"
            );
        }
    }
    
    @Override
    public void onCollectionModified(Object entity, Object revision) {
        // Обработка изменений коллекций
    }
}
```

## Регистрация слушателя

### Через конфигурацию Hibernate

```java
@Configuration
public class EnversConfig {
    
    @Bean
    public EnversListener customEnversListener() {
        return new CustomEnversListener();
    }
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        
        Properties properties = new Properties();
        properties.setProperty("hibernate.envers.audit_listener", 
                             "com.example.CustomEnversListener");
        
        em.setJpaProperties(properties);
        return em;
    }
}
```

### Через application.properties

```properties
# Настройка кастомного слушателя
hibernate.envers.audit_listener=com.example.CustomEnversListener
```

## Практические сценарии

### 1. Аудит безопасности

```java
@Component
public class SecurityEnversListener implements EnversListener {
    
    @Override
    public void onEntityModified(Object entity, Object revision) {
        if (entity instanceof User) {
            User user = (User) entity;
            
            // Проверка изменения роли
            if (user.getRoleChanged()) {
                securityService.logRoleChange(user);
                securityService.notifySecurityTeam(user);
            }
            
            // Проверка изменения пароля
            if (user.getPasswordChanged()) {
                securityService.logPasswordChange(user);
            }
        }
    }
}
```

### 2. Аудит финансовых операций

```java
@Component
public class FinancialEnversListener implements EnversListener {
    
    @Override
    public void onEntityModified(Object entity, Object revision) {
        if (entity instanceof Transaction) {
            Transaction transaction = (Transaction) entity;
            
            // Проверка крупных транзакций
            if (transaction.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0) {
                complianceService.reportLargeTransaction(transaction);
            }
            
            // Проверка подозрительной активности
            if (fraudDetectionService.isSuspicious(transaction)) {
                fraudDetectionService.flagTransaction(transaction);
            }
        }
    }
}
```

### 3. Аудит производительности

```java
@Component
public class PerformanceEnversListener implements EnversListener {
    
    private final Map<String, Long> operationCounts = new ConcurrentHashMap<>();
    
    @Override
    public void onEntityAdded(Object entity, Object revision) {
        String entityType = entity.getClass().getSimpleName();
        operationCounts.merge(entityType, 1L, Long::sum);
        
        // Логирование статистики
        if (operationCounts.get(entityType) % 100 == 0) {
            log.info("Статистика аудита для {}: {} операций", 
                    entityType, operationCounts.get(entityType));
        }
    }
}
```

## Интеграция с Spring

### Использование Spring Events

```java
@Component
public class SpringEnversListener implements EnversListener {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Override
    public void onEntityAdded(Object entity, Object revision) {
        eventPublisher.publishEvent(new EntityAuditedEvent(entity, "ADD", revision));
    }
    
    @Override
    public void onEntityModified(Object entity, Object revision) {
        eventPublisher.publishEvent(new EntityAuditedEvent(entity, "MODIFY", revision));
    }
    
    @Override
    public void onEntityDeleted(Object entity, Object revision) {
        eventPublisher.publishEvent(new EntityAuditedEvent(entity, "DELETE", revision));
    }
    
    @Override
    public void onCollectionModified(Object entity, Object revision) {
        eventPublisher.publishEvent(new EntityAuditedEvent(entity, "COLLECTION_MODIFY", revision));
    }
}

// Событие Spring
public class EntityAuditedEvent {
    private final Object entity;
    private final String operation;
    private final Object revision;
    
    // Конструктор, геттеры...
}
```

## Лучшие практики

### 1. Производительность

```java
@Component
public class OptimizedEnversListener implements EnversListener {
    
    private final ExecutorService executor = Executors.newFixedThreadPool(5);
    
    @Override
    public void onEntityAdded(Object entity, Object revision) {
        // Асинхронная обработка для тяжелых операций
        executor.submit(() -> processEntityAsync(entity, revision));
    }
    
    private void processEntityAsync(Object entity, Object revision) {
        // Тяжелая обработка
    }
}
```

### 2. Обработка ошибок

```java
@Component
public class SafeEnversListener implements EnversListener {
    
    @Override
    public void onEntityModified(Object entity, Object revision) {
        try {
            processEntityModification(entity, revision);
        } catch (Exception e) {
            log.error("Ошибка при обработке изменения сущности", e);
            // Не прерываем процесс аудита
        }
    }
    
    private void processEntityModification(Object entity, Object revision) {
        // Логика обработки
    }
}
```

### 3. Фильтрация сущностей

```java
@Component
public class FilteredEnversListener implements EnversListener {
    
    private final Set<Class<?>> auditedClasses = Set.of(
        User.class, Product.class, Order.class
    );
    
    @Override
    public void onEntityAdded(Object entity, Object revision) {
        if (shouldAudit(entity)) {
            processAudit(entity, revision);
        }
    }
    
    private boolean shouldAudit(Object entity) {
        return auditedClasses.contains(entity.getClass());
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class CustomEnversListenerTest {
    
    @Mock
    private AuditNotificationService notificationService;
    
    @InjectMocks
    private CustomEnversListener listener;
    
    @Test
    void onEntityAdded_ShouldSendNotification() {
        // Given
        User user = new User("test", "test@example.com");
        Object revision = new Object();
        
        // When
        listener.onEntityAdded(user, revision);
        
        // Then
        verify(notificationService).sendAuditNotification("ADD", user, revision);
    }
}
```

### Integration тесты

```java
@SpringBootTest
@TestPropertySource(properties = {
    "hibernate.envers.audit_listener=com.example.CustomEnversListener"
})
class EnversListenerIntegrationTest {
    
    @Autowired
    private EntityManager entityManager;
    
    @Test
    void shouldTriggerListenerOnEntitySave() {
        // Given
        User user = new User("test", "test@example.com");
        
        // When
        entityManager.persist(user);
        entityManager.flush();
        
        // Then - проверяем, что слушатель сработал
        // (через логи или моки)
    }
}
```

## Заключение

Интерфейс `EnversListener` предоставляет мощные возможности для кастомизации процесса аудита в Hibernate Envers. Правильное использование этого интерфейса позволяет:

- Интегрировать аудит с бизнес-логикой
- Отправлять уведомления о важных изменениях
- Собирать статистику и метрики
- Обеспечивать безопасность и соответствие требованиям
- Оптимизировать производительность

При создании кастомных слушателей важно помнить о производительности, обработке ошибок и правильной интеграции с существующей архитектурой приложения. 