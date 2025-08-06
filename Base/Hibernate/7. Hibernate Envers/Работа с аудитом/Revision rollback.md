# Revision rollback

Revision rollback (откат ревизий) - это механизм восстановления данных к предыдущему состоянию в Hibernate Envers. Этот процесс позволяет вернуть сущности к определенной версии из истории изменений.

## Обзор концепции

Откат ревизий включает в себя:
- Восстановление данных из аудита
- Создание новой ревизии для отката
- Сохранение информации об откате
- Обработка связанных сущностей

## Базовый механизм отката

### Простой откат сущности

```java
@Service
@Transactional
public class BasicRollbackService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    private AuditReader getAuditReader() {
        return auditReaderFactory.get(entityManager);
    }
    
    public <T> T rollbackToRevision(Class<T> entityClass, Serializable id, Number targetRevision) {
        AuditReader auditReader = getAuditReader();
        
        // Получение сущности из целевой ревизии
        T entityAtRevision = auditReader.find(entityClass, id, targetRevision);
        
        if (entityAtRevision == null) {
            throw new EntityNotFoundException("Сущность не найдена в ревизии " + targetRevision);
        }
        
        // Получение текущей сущности
        T currentEntity = entityManager.find(entityClass, id);
        
        if (currentEntity == null) {
            throw new EntityNotFoundException("Текущая сущность не найдена");
        }
        
        // Копирование данных из целевой ревизии
        copyEntityData(entityAtRevision, currentEntity);
        
        // Сохранение изменений
        entityManager.merge(currentEntity);
        
        return currentEntity;
    }
    
    private <T> void copyEntityData(T source, T target) {
        BeanUtils.copyProperties(source, target, "id", "version");
    }
    
    public User rollbackUser(Long userId, Number targetRevision) {
        return rollbackToRevision(User.class, userId, targetRevision);
    }
    
    public Product rollbackProduct(Long productId, Number targetRevision) {
        return rollbackToRevision(Product.class, productId, targetRevision);
    }
}
```

### Откат с созданием новой ревизии

```java
@Service
@Transactional
public class RollbackWithRevisionService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    private AuditReader getAuditReader() {
        return auditReaderFactory.get(entityManager);
    }
    
    public <T> T rollbackWithNewRevision(Class<T> entityClass, Serializable id, 
                                        Number targetRevision, String rollbackReason) {
        AuditReader auditReader = getAuditReader();
        
        // Получение сущности из целевой ревизии
        T entityAtRevision = auditReader.find(entityClass, id, targetRevision);
        
        if (entityAtRevision == null) {
            throw new EntityNotFoundException("Сущность не найдена в ревизии " + targetRevision);
        }
        
        // Создание новой сущности с данными из целевой ревизии
        T newEntity = createEntityFromRevision(entityAtRevision);
        
        // Установка причины отката
        setRollbackReason(newEntity, rollbackReason);
        
        // Сохранение новой сущности
        entityManager.persist(newEntity);
        
        return newEntity;
    }
    
    private <T> T createEntityFromRevision(T entityAtRevision) {
        try {
            @SuppressWarnings("unchecked")
            Class<T> entityClass = (Class<T>) entityAtRevision.getClass();
            T newEntity = entityClass.getDeclaredConstructor().newInstance();
            BeanUtils.copyProperties(entityAtRevision, newEntity, "id", "version");
            return newEntity;
        } catch (Exception e) {
            throw new RuntimeException("Ошибка при создании сущности из ревизии", e);
        }
    }
    
    private <T> void setRollbackReason(T entity, String reason) {
        // Установка причины отката (если сущность поддерживает это)
        if (entity instanceof RollbackableEntity) {
            ((RollbackableEntity) entity).setRollbackReason(reason);
        }
    }
}
```

## Расширенный откат

### Откат с кастомной ревизией

```java
@Service
@Transactional
public class CustomRollbackService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    private AuditReader getAuditReader() {
        return auditReaderFactory.get(entityManager, CustomRevisionEntity.class);
    }
    
    public <T> T rollbackWithCustomRevision(Class<T> entityClass, Serializable id, 
                                           Number targetRevision, String rollbackReason) {
        AuditReader auditReader = getAuditReader();
        
        // Получение сущности из целевой ревизии
        T entityAtRevision = auditReader.find(entityClass, id, targetRevision);
        
        if (entityAtRevision == null) {
            throw new EntityNotFoundException("Сущность не найдена в ревизии " + targetRevision);
        }
        
        // Получение информации о целевой ревизии
        CustomRevisionEntity targetRevisionInfo = auditReader.findRevision(CustomRevisionEntity.class, targetRevision);
        
        // Создание новой сущности
        T newEntity = createEntityFromRevision(entityAtRevision);
        
        // Установка метаданных отката
        setRollbackMetadata(newEntity, targetRevisionInfo, rollbackReason);
        
        // Сохранение
        entityManager.persist(newEntity);
        
        return newEntity;
    }
    
    private <T> void setRollbackMetadata(T entity, CustomRevisionEntity targetRevision, String reason) {
        if (entity instanceof RollbackableEntity) {
            RollbackableEntity rollbackableEntity = (RollbackableEntity) entity;
            rollbackableEntity.setRollbackReason(reason);
            rollbackableEntity.setRollbackFromRevision(targetRevision.getRev());
            rollbackableEntity.setRollbackFromUser(targetRevision.getUsername());
            rollbackableEntity.setRollbackTimestamp(System.currentTimeMillis());
        }
    }
    
    public User rollbackUserWithMetadata(Long userId, Number targetRevision, String reason) {
        return rollbackWithCustomRevision(User.class, userId, targetRevision, reason);
    }
}
```

### Откат связанных сущностей

```java
@Service
@Transactional
public class RelatedEntityRollbackService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    private AuditReader getAuditReader() {
        return auditReaderFactory.get(entityManager);
    }
    
    public Order rollbackOrderWithItems(Long orderId, Number targetRevision) {
        AuditReader auditReader = getAuditReader();
        
        // Получение заказа из целевой ревизии
        Order orderAtRevision = auditReader.find(Order.class, orderId, targetRevision);
        
        if (orderAtRevision == null) {
            throw new EntityNotFoundException("Заказ не найден в ревизии " + targetRevision);
        }
        
        // Создание нового заказа
        Order newOrder = createOrderFromRevision(orderAtRevision);
        
        // Откат связанных элементов заказа
        rollbackOrderItems(newOrder, orderAtRevision, targetRevision);
        
        // Сохранение
        entityManager.persist(newOrder);
        
        return newOrder;
    }
    
    private Order createOrderFromRevision(Order orderAtRevision) {
        Order newOrder = new Order();
        BeanUtils.copyProperties(orderAtRevision, newOrder, "id", "version", "items");
        return newOrder;
    }
    
    private void rollbackOrderItems(Order newOrder, Order orderAtRevision, Number targetRevision) {
        AuditReader auditReader = getAuditReader();
        
        // Получение элементов заказа из целевой ревизии
        List<OrderItem> itemsAtRevision = orderAtRevision.getItems();
        
        for (OrderItem itemAtRevision : itemsAtRevision) {
            OrderItem newItem = new OrderItem();
            BeanUtils.copyProperties(itemAtRevision, newItem, "id", "version", "order");
            newItem.setOrder(newOrder);
            newOrder.getItems().add(newItem);
        }
    }
}
```

## Валидация отката

### Проверка возможности отката

```java
@Service
public class RollbackValidationService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    private AuditReader getAuditReader() {
        return auditReaderFactory.get(entityManager);
    }
    
    public RollbackValidationResult validateRollback(Class<?> entityClass, Serializable id, 
                                                   Number targetRevision) {
        AuditReader auditReader = getAuditReader();
        
        RollbackValidationResult result = new RollbackValidationResult();
        
        // Проверка существования сущности в целевой ревизии
        Object entityAtRevision = auditReader.find(entityClass, id, targetRevision);
        if (entityAtRevision == null) {
            result.addError("Сущность не найдена в ревизии " + targetRevision);
            return result;
        }
        
        // Проверка существования текущей сущности
        Object currentEntity = entityManager.find(entityClass, id);
        if (currentEntity == null) {
            result.addError("Текущая сущность не найдена");
            return result;
        }
        
        // Проверка бизнес-правил
        validateBusinessRules(entityAtRevision, currentEntity, result);
        
        // Проверка зависимостей
        validateDependencies(entityClass, id, targetRevision, result);
        
        return result;
    }
    
    private void validateBusinessRules(Object entityAtRevision, Object currentEntity, 
                                    RollbackValidationResult result) {
        // Проверка бизнес-правил
        if (entityAtRevision instanceof User) {
            User userAtRevision = (User) entityAtRevision;
            User currentUser = (User) currentEntity;
            
            // Проверка, не нарушает ли откат бизнес-правила
            if (userAtRevision.getStatus() == UserStatus.DELETED && 
                currentUser.getStatus() == UserStatus.ACTIVE) {
                result.addWarning("Откат может восстановить удаленного пользователя");
            }
        }
    }
    
    private void validateDependencies(Class<?> entityClass, Serializable id, 
                                   Number targetRevision, RollbackValidationResult result) {
        // Проверка зависимостей
        if (entityClass == Order.class) {
            validateOrderDependencies((Long) id, targetRevision, result);
        }
    }
    
    private void validateOrderDependencies(Long orderId, Number targetRevision, 
                                        RollbackValidationResult result) {
        // Проверка зависимостей заказа
        AuditReader auditReader = getAuditReader();
        
        // Проверка, не связан ли заказ с активными транзакциями
        List<Object> relatedTransactions = auditReader.createQuery()
            .forEntitiesAtRevision(Transaction.class, targetRevision)
            .add(AuditEntity.property("order.id").eq(orderId))
            .getResultList();
        
        if (!relatedTransactions.isEmpty()) {
            result.addWarning("Заказ связан с транзакциями, откат может повлиять на них");
        }
    }
}
```

### Класс результата валидации

```java
public class RollbackValidationResult {
    
    private final List<String> errors = new ArrayList<>();
    private final List<String> warnings = new ArrayList<>();
    
    public void addError(String error) {
        errors.add(error);
    }
    
    public void addWarning(String warning) {
        warnings.add(warning);
    }
    
    public boolean isValid() {
        return errors.isEmpty();
    }
    
    public boolean hasWarnings() {
        return !warnings.isEmpty();
    }
    
    public List<String> getErrors() {
        return Collections.unmodifiableList(errors);
    }
    
    public List<String> getWarnings() {
        return Collections.unmodifiableList(warnings);
    }
    
    public String getSummary() {
        StringBuilder summary = new StringBuilder();
        
        if (!errors.isEmpty()) {
            summary.append("Ошибки: ").append(String.join(", ", errors));
        }
        
        if (!warnings.isEmpty()) {
            if (summary.length() > 0) {
                summary.append("; ");
            }
            summary.append("Предупреждения: ").append(String.join(", ", warnings));
        }
        
        return summary.toString();
    }
}
```

## Практические сценарии

### Откат пользователя

```java
@Service
@Transactional
public class UserRollbackService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    private AuditReader getAuditReader() {
        return auditReaderFactory.get(entityManager, CustomRevisionEntity.class);
    }
    
    public User rollbackUser(Long userId, Number targetRevision, String reason) {
        // Валидация отката
        RollbackValidationService validationService = new RollbackValidationService();
        RollbackValidationResult validation = validationService.validateRollback(User.class, userId, targetRevision);
        
        if (!validation.isValid()) {
            throw new RollbackValidationException("Ошибка валидации отката: " + validation.getSummary());
        }
        
        if (validation.hasWarnings()) {
            log.warn("Предупреждения при откате пользователя: {}", validation.getSummary());
        }
        
        // Выполнение отката
        return rollbackUserWithMetadata(userId, targetRevision, reason);
    }
    
    private User rollbackUserWithMetadata(Long userId, Number targetRevision, String reason) {
        AuditReader auditReader = getAuditReader();
        
        User userAtRevision = auditReader.find(User.class, userId, targetRevision);
        User newUser = new User();
        
        // Копирование данных
        BeanUtils.copyProperties(userAtRevision, newUser, "id", "version");
        
        // Установка метаданных отката
        newUser.setRollbackReason(reason);
        newUser.setRollbackFromRevision(targetRevision);
        newUser.setRollbackTimestamp(System.currentTimeMillis());
        
        // Сохранение
        entityManager.persist(newUser);
        
        return newUser;
    }
    
    public List<User> rollbackMultipleUsers(List<Long> userIds, Number targetRevision, String reason) {
        List<User> rolledBackUsers = new ArrayList<>();
        
        for (Long userId : userIds) {
            try {
                User rolledBackUser = rollbackUser(userId, targetRevision, reason);
                rolledBackUsers.add(rolledBackUser);
            } catch (Exception e) {
                log.error("Ошибка при откате пользователя {}: {}", userId, e.getMessage());
                // Продолжаем с другими пользователями
            }
        }
        
        return rolledBackUsers;
    }
}
```

### Откат заказа с элементами

```java
@Service
@Transactional
public class OrderRollbackService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    private AuditReader getAuditReader() {
        return auditReaderFactory.get(entityManager);
    }
    
    public Order rollbackOrder(Long orderId, Number targetRevision, String reason) {
        // Валидация
        RollbackValidationService validationService = new RollbackValidationService();
        RollbackValidationResult validation = validationService.validateRollback(Order.class, orderId, targetRevision);
        
        if (!validation.isValid()) {
            throw new RollbackValidationException("Ошибка валидации отката заказа: " + validation.getSummary());
        }
        
        // Выполнение отката
        return rollbackOrderWithItems(orderId, targetRevision, reason);
    }
    
    private Order rollbackOrderWithItems(Long orderId, Number targetRevision, String reason) {
        AuditReader auditReader = getAuditReader();
        
        Order orderAtRevision = auditReader.find(Order.class, orderId, targetRevision);
        Order newOrder = new Order();
        
        // Копирование основных данных заказа
        BeanUtils.copyProperties(orderAtRevision, newOrder, "id", "version", "items");
        
        // Установка метаданных отката
        newOrder.setRollbackReason(reason);
        newOrder.setRollbackFromRevision(targetRevision);
        newOrder.setRollbackTimestamp(System.currentTimeMillis());
        
        // Откат элементов заказа
        rollbackOrderItems(newOrder, orderAtRevision, targetRevision);
        
        // Сохранение
        entityManager.persist(newOrder);
        
        return newOrder;
    }
    
    private void rollbackOrderItems(Order newOrder, Order orderAtRevision, Number targetRevision) {
        AuditReader auditReader = getAuditReader();
        
        for (OrderItem itemAtRevision : orderAtRevision.getItems()) {
            OrderItem newItem = new OrderItem();
            BeanUtils.copyProperties(itemAtRevision, newItem, "id", "version", "order");
            newItem.setOrder(newOrder);
            newOrder.getItems().add(newItem);
        }
    }
}
```

## Обработка ошибок

### Безопасный откат

```java
@Service
@Transactional
public class SafeRollbackService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    private AuditReader getAuditReader() {
        return auditReaderFactory.get(entityManager);
    }
    
    public Optional<Object> safeRollback(Class<?> entityClass, Serializable id, 
                                       Number targetRevision, String reason) {
        try {
            // Валидация
            RollbackValidationService validationService = new RollbackValidationService();
            RollbackValidationResult validation = validationService.validateRollback(entityClass, id, targetRevision);
            
            if (!validation.isValid()) {
                log.error("Ошибка валидации отката: {}", validation.getSummary());
                return Optional.empty();
            }
            
            // Выполнение отката
            Object rolledBackEntity = performRollback(entityClass, id, targetRevision, reason);
            
            return Optional.of(rolledBackEntity);
            
        } catch (Exception e) {
            log.error("Ошибка при выполнении отката", e);
            return Optional.empty();
        }
    }
    
    private Object performRollback(Class<?> entityClass, Serializable id, 
                                 Number targetRevision, String reason) {
        AuditReader auditReader = getAuditReader();
        
        Object entityAtRevision = auditReader.find(entityClass, id, targetRevision);
        Object newEntity = createEntityFromRevision(entityAtRevision);
        
        // Установка метаданных отката
        setRollbackMetadata(newEntity, reason, targetRevision);
        
        // Сохранение
        entityManager.persist(newEntity);
        
        return newEntity;
    }
    
    private Object createEntityFromRevision(Object entityAtRevision) {
        try {
            Class<?> entityClass = entityAtRevision.getClass();
            Object newEntity = entityClass.getDeclaredConstructor().newInstance();
            BeanUtils.copyProperties(entityAtRevision, newEntity, "id", "version");
            return newEntity;
        } catch (Exception e) {
            throw new RuntimeException("Ошибка при создании сущности из ревизии", e);
        }
    }
    
    private void setRollbackMetadata(Object entity, String reason, Number targetRevision) {
        if (entity instanceof RollbackableEntity) {
            RollbackableEntity rollbackableEntity = (RollbackableEntity) entity;
            rollbackableEntity.setRollbackReason(reason);
            rollbackableEntity.setRollbackFromRevision(targetRevision);
            rollbackableEntity.setRollbackTimestamp(System.currentTimeMillis());
        }
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class RollbackServiceTest {
    
    @Mock
    private EntityManager entityManager;
    
    @Mock
    private AuditReader auditReader;
    
    @InjectMocks
    private BasicRollbackService service;
    
    @Test
    void rollbackToRevision_ShouldRestoreEntity() {
        // Given
        User userAtRevision = new User("old", "old@example.com");
        User currentUser = new User("current", "current@example.com");
        
        when(auditReader.find(User.class, 1L, 5)).thenReturn(userAtRevision);
        when(entityManager.find(User.class, 1L)).thenReturn(currentUser);
        
        // When
        User result = service.rollbackUser(1L, 5);
        
        // Then
        assertEquals("old", result.getUsername());
        assertEquals("old@example.com", result.getEmail());
    }
}
```

### Integration тесты

```java
@SpringBootTest
class RollbackIntegrationTest {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private BasicRollbackService service;
    
    @Test
    void shouldRollbackUser() {
        // Given
        User user = new User("test", "test@example.com");
        entityManager.persist(user);
        entityManager.flush();
        
        // Изменение пользователя
        user.setUsername("modified");
        entityManager.merge(user);
        entityManager.flush();
        
        // When
        User rolledBackUser = service.rollbackUser(user.getId(), 1);
        
        // Then
        assertEquals("test", rolledBackUser.getUsername());
    }
}
```

## Заключение

Revision rollback предоставляет мощные возможности для восстановления данных в Hibernate Envers:

- **Восстановление**: Возврат сущностей к предыдущим состояниям
- **Валидация**: Проверка возможности и безопасности отката
- **Метаданные**: Сохранение информации об откате
- **Связанные сущности**: Обработка зависимостей при откате
- **Безопасность**: Обработка ошибок и исключений

При работе с откатом ревизий важно помнить о валидации, обработке ошибок и сохранении целостности данных. 