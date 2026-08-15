# Методы класса AuditReader

`AuditReader` является основным API для чтения аудита в Hibernate Envers. Этот класс предоставляет множество методов для получения и анализа данных аудита.

## Обзор класса

```java
public interface AuditReader {
    
    // Основные методы поиска
    Object find(String entityName, Serializable id, Number revision);
    Object find(Class<?> cls, Serializable id, Number revision);
    
    // Получение ревизий
    List<Number> getRevisions(String entityName, Serializable id);
    List<Number> getRevisions(Class<?> cls, Serializable id);
    
    // Получение информации о ревизии
    <T> T findRevision(Class<T> revisionEntityClass, Number revision);
    
    // Создание запросов
    AuditQuery createQuery();
    
    // Проверка существования
    boolean isEntityClassAudited(Class<?> entityClass);
    boolean isEntityNameAudited(String entityName);
}
```

## Основные методы поиска

### find(String entityName, Serializable id, Number revision)

Поиск сущности по имени, ID и номеру ревизии.

```java
@Service
public class BasicAuditReaderService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public Object findEntity(String entityName, Serializable id, Number revision) {
        AuditReader auditReader = getAuditReader();
        return auditReader.find(entityName, id, revision);
    }
    
    public User findUser(Long userId, Number revision) {
        return (User) findEntity("User", userId, revision);
    }
    
    public Product findProduct(Long productId, Number revision) {
        return (Product) findEntity("Product", productId, revision);
    }
    
    public Order findOrder(Long orderId, Number revision) {
        return (Order) findEntity("Order", orderId, revision);
    }
}
```

### find(Class<?> cls, Serializable id, Number revision)

Поиск сущности по классу, ID и номеру ревизии.

```java
@Service
public class TypedAuditReaderService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public <T> T findEntity(Class<T> entityClass, Serializable id, Number revision) {
        AuditReader auditReader = getAuditReader();
        return auditReader.find(entityClass, id, revision);
    }
    
    public User findUser(Long userId, Number revision) {
        return findEntity(User.class, userId, revision);
    }
    
    public Product findProduct(Long productId, Number revision) {
        return findEntity(Product.class, productId, revision);
    }
    
    public List<User> findUsersAtRevision(List<Long> userIds, Number revision) {
        return userIds.stream()
            .map(userId -> findEntity(User.class, userId, revision))
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

## Получение ревизий

### getRevisions(String entityName, Serializable id)

Получение списка ревизий для сущности по имени.

```java
@Service
public class RevisionService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Number> getRevisions(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        return auditReader.getRevisions(entityName, id);
    }
    
    public List<Number> getUserRevisions(Long userId) {
        return getRevisions("User", userId);
    }
    
    public List<Number> getProductRevisions(Long productId) {
        return getRevisions("Product", productId);
    }
    
    public List<Number> getOrderRevisions(Long orderId) {
        return getRevisions("Order", orderId);
    }
    
    public Number getLatestRevision(String entityName, Serializable id) {
        List<Number> revisions = getRevisions(entityName, id);
        return revisions.isEmpty() ? null : revisions.get(revisions.size() - 1);
    }
    
    public Number getFirstRevision(String entityName, Serializable id) {
        List<Number> revisions = getRevisions(entityName, id);
        return revisions.isEmpty() ? null : revisions.get(0);
    }
}
```

### getRevisions(Class<?> cls, Serializable id)

Получение списка ревизий для сущности по классу.

```java
@Service
public class TypedRevisionService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public <T> List<Number> getRevisions(Class<T> entityClass, Serializable id) {
        AuditReader auditReader = getAuditReader();
        return auditReader.getRevisions(entityClass, id);
    }
    
    public List<Number> getUserRevisions(Long userId) {
        return getRevisions(User.class, userId);
    }
    
    public List<Number> getProductRevisions(Long productId) {
        return getRevisions(Product.class, productId);
    }
    
    public Map<Long, List<Number>> getRevisionsForMultipleUsers(List<Long> userIds) {
        return userIds.stream()
            .collect(Collectors.toMap(
                userId -> userId,
                userId -> getRevisions(User.class, userId)
            ));
    }
}
```

## Получение информации о ревизии

### findRevision(Class<T> revisionEntityClass, Number revision)

Получение информации о конкретной ревизии.

```java
@Service
public class RevisionInfoService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager, CustomRevisionEntity.class);
    }
    
    public CustomRevisionEntity getRevisionInfo(Number revision) {
        AuditReader auditReader = getAuditReader();
        return auditReader.findRevision(CustomRevisionEntity.class, revision);
    }
    
    public List<CustomRevisionEntity> getRevisionsInfo(List<Number> revisions) {
        AuditReader auditReader = getAuditReader();
        return revisions.stream()
            .map(rev -> auditReader.findRevision(CustomRevisionEntity.class, rev))
            .collect(Collectors.toList());
    }
    
    public List<CustomRevisionEntity> getRevisionsInfoForEntity(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        return getRevisionsInfo(revisions);
    }
    
    public CustomRevisionEntity getLatestRevisionInfo(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        
        if (revisions.isEmpty()) {
            return null;
        }
        
        Number latestRevision = revisions.get(revisions.size() - 1);
        return auditReader.findRevision(CustomRevisionEntity.class, latestRevision);
    }
}
```

## Создание запросов

### createQuery()

Создание запроса к аудиту.

```java
@Service
public class AuditQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Object> findEntitiesAtRevision(String entityName, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(entityName, revision);
        
        return query.getResultList();
    }
    
    public List<User> findUsersAtRevision(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision);
        
        return query.getResultList();
    }
    
    public List<Object> findEntitiesModifiedInRevision(String entityName, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesModifiedAtRevision(entityName, revision);
        
        return query.getResultList();
    }
    
    public List<User> findUsersModifiedInRevision(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesModifiedAtRevision(User.class, revision);
        
        return query.getResultList();
    }
}
```

## Проверка существования

### isEntityClassAudited(Class<?> entityClass)

Проверка, является ли класс аудируемым.

```java
@Service
public class AuditValidationService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public boolean isEntityAudited(Class<?> entityClass) {
        AuditReader auditReader = getAuditReader();
        return auditReader.isEntityClassAudited(entityClass);
    }
    
    public boolean isEntityAudited(String entityName) {
        AuditReader auditReader = getAuditReader();
        return auditReader.isEntityNameAudited(entityName);
    }
    
    public List<Class<?>> getAuditedClasses(List<Class<?>> classes) {
        return classes.stream()
            .filter(this::isEntityAudited)
            .collect(Collectors.toList());
    }
    
    public void validateAuditedEntity(Class<?> entityClass) {
        if (!isEntityAudited(entityClass)) {
            throw new IllegalArgumentException(
                "Класс " + entityClass.getSimpleName() + " не является аудируемым"
            );
        }
    }
}
```

## Расширенные методы

### Работа с коллекциями и ассоциациями

```java
@Service
public class AssociationAuditService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Object> getEntityWithAssociations(String entityName, Serializable id, Number revision) {
        AuditReader auditReader = getAuditReader();
        Object entity = auditReader.find(entityName, id, revision);
        
        // Загрузка связанных сущностей
        loadAssociations(entity, auditReader, revision);
        
        return Arrays.asList(entity);
    }
    
    private void loadAssociations(Object entity, AuditReader auditReader, Number revision) {
        if (entity instanceof User) {
            User user = (User) entity;
            // Загрузка связанных сущностей пользователя
            loadUserAssociations(user, auditReader, revision);
        }
    }
    
    private void loadUserAssociations(User user, AuditReader auditReader, Number revision) {
        // Загрузка заказов пользователя
        if (user.getOrders() != null) {
            for (Order order : user.getOrders()) {
                Order auditedOrder = auditReader.find(Order.class, order.getId(), revision);
                if (auditedOrder != null) {
                    order.setItems(auditedOrder.getItems());
                }
            }
        }
    }
}
```

### Работа с вложенными объектами

```java
@Service
public class NestedObjectAuditService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public Object getEntityWithNestedObjects(String entityName, Serializable id, Number revision) {
        AuditReader auditReader = getAuditReader();
        Object entity = auditReader.find(entityName, id, revision);
        
        // Обработка вложенных объектов
        processNestedObjects(entity, auditReader, revision);
        
        return entity;
    }
    
    private void processNestedObjects(Object entity, AuditReader auditReader, Number revision) {
        if (entity instanceof Customer) {
            Customer customer = (Customer) entity;
            
            // Обработка адресов
            if (customer.getBillingAddress() != null) {
                // Адреса являются @Embeddable, поэтому обрабатываются автоматически
            }
            
            if (customer.getShippingAddress() != null) {
                // Адреса являются @Embeddable, поэтому обрабатываются автоматически
            }
        }
    }
}
```

## Обработка ошибок

### Безопасное чтение аудита

```java
@Service
public class SafeAuditReaderService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        try {
            return AuditReaderFactory.get(entityManager);
        } catch (Exception e) {
            log.error("Ошибка при создании AuditReader", e);
            throw new AuditException("Не удалось создать AuditReader", e);
        }
    }
    
    public Optional<Object> findEntitySafely(String entityName, Serializable id, Number revision) {
        try {
            AuditReader auditReader = getAuditReader();
            Object entity = auditReader.find(entityName, id, revision);
            return Optional.ofNullable(entity);
        } catch (Exception e) {
            log.error("Ошибка при поиске сущности в аудите", e);
            return Optional.empty();
        }
    }
    
    public Optional<List<Number>> getRevisionsSafely(String entityName, Serializable id) {
        try {
            AuditReader auditReader = getAuditReader();
            List<Number> revisions = auditReader.getRevisions(entityName, id);
            return Optional.of(revisions);
        } catch (Exception e) {
            log.error("Ошибка при получении ревизий", e);
            return Optional.empty();
        }
    }
    
    public Optional<Object> getLatestVersionSafely(String entityName, Serializable id) {
        Optional<List<Number>> revisionsOpt = getRevisionsSafely(entityName, id);
        
        if (revisionsOpt.isEmpty() || revisionsOpt.get().isEmpty()) {
            return Optional.empty();
        }
        
        Number latestRevision = revisionsOpt.get().get(revisionsOpt.get().size() - 1);
        return findEntitySafely(entityName, id, latestRevision);
    }
}
```

## Производительность

### Кэширование результатов

```java
@Service
public class CachedAuditReaderService {
    
    @Autowired
    private EntityManager entityManager;
    
    private final Map<String, Object> entityCache = new ConcurrentHashMap<>();
    private final Map<String, List<Number>> revisionCache = new ConcurrentHashMap<>();
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public Object findEntityWithCache(String entityName, Serializable id, Number revision) {
        String cacheKey = entityName + "_" + id + "_" + revision;
        
        return entityCache.computeIfAbsent(cacheKey, key -> {
            AuditReader auditReader = getAuditReader();
            return auditReader.find(entityName, id, revision);
        });
    }
    
    public List<Number> getRevisionsWithCache(String entityName, Serializable id) {
        String cacheKey = entityName + "_" + id + "_revisions";
        
        return revisionCache.computeIfAbsent(cacheKey, key -> {
            AuditReader auditReader = getAuditReader();
            return auditReader.getRevisions(entityName, id);
        });
    }
    
    public void clearCache() {
        entityCache.clear();
        revisionCache.clear();
    }
    
    public void clearEntityCache(String entityName, Serializable id) {
        String prefix = entityName + "_" + id + "_";
        entityCache.keySet().removeIf(key -> key.startsWith(prefix));
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class AuditReaderServiceTest {
    
    @Mock
    private EntityManager entityManager;
    
    @Mock
    private AuditReader auditReader;
    
    @InjectMocks
    private BasicAuditReaderService service;
    
    @Test
    void findEntity_ShouldReturnEntity() {
        // Given
        User expectedUser = new User("test", "test@example.com");
        when(auditReader.find("User", 1L, 5)).thenReturn(expectedUser);
        
        // When
        User result = service.findUser(1L, 5);
        
        // Then
        assertEquals(expectedUser, result);
    }
    
    @Test
    void getRevisions_ShouldReturnRevisionList() {
        // Given
        List<Number> expectedRevisions = Arrays.asList(1, 3, 5);
        when(auditReader.getRevisions("User", 1L)).thenReturn(expectedRevisions);
        
        // When
        List<Number> result = service.getUserRevisions(1L);
        
        // Then
        assertEquals(expectedRevisions, result);
    }
}
```

### Integration тесты

```java
@SpringBootTest
class AuditReaderIntegrationTest {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private BasicAuditReaderService service;
    
    @Test
    void shouldFindEntityAtRevision() {
        // Given
        User user = new User("test", "test@example.com");
        entityManager.persist(user);
        entityManager.flush();
        
        // When
        User foundUser = service.findUser(user.getId(), 1);
        
        // Then
        assertNotNull(foundUser);
        assertEquals("test", foundUser.getUsername());
    }
}
```

## Заключение

Методы класса `AuditReader` предоставляют мощные возможности для работы с аудитом в Hibernate Envers:

- **Поиск**: Гибкие методы поиска сущностей по различным критериям
- **Ревизии**: Получение и управление ревизиями
- **Запросы**: Создание сложных запросов к аудиту
- **Валидация**: Проверка аудируемости сущностей
- **Производительность**: Кэширование и оптимизация

При работе с `AuditReader` важно помнить о производительности, правильной обработке ошибок и эффективном использовании памяти. 