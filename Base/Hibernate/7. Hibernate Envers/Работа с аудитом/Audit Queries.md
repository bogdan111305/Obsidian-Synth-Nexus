# Audit Queries

Audit Queries предоставляют мощный API для создания сложных запросов к аудиту в Hibernate Envers. Эти запросы позволяют фильтровать, сортировать и агрегировать данные аудита.

## Обзор AuditQuery

```java
public interface AuditQuery {
    
    // Базовые методы
    AuditQuery forEntitiesAtRevision(Class<?> entityClass, Number revision);
    AuditQuery forEntitiesAtRevision(String entityName, Number revision);
    AuditQuery forEntitiesModifiedAtRevision(Class<?> entityClass, Number revision);
    AuditQuery forEntitiesModifiedAtRevision(String entityName, Number revision);
    
    // Фильтрация
    AuditQuery add(AuditCriterion criterion);
    AuditQuery addProjection(AuditProjection projection);
    
    // Сортировка
    AuditQuery addOrder(AuditOrder order);
    
    // Пагинация
    AuditQuery setFirstResult(int firstResult);
    AuditQuery setMaxResults(int maxResults);
    
    // Выполнение
    List<Object> getResultList();
    Object getSingleResult();
    long getResultCount();
}
```

## Базовые запросы

### Запросы к сущностям в определенной ревизии

```java
@Service
public class BasicAuditQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Object> getEntitiesAtRevision(String entityName, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(entityName, revision);
        
        return query.getResultList();
    }
    
    public List<User> getUsersAtRevision(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision);
        
        return query.getResultList();
    }
    
    public List<Product> getProductsAtRevision(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Product.class, revision);
        
        return query.getResultList();
    }
}
```

### Запросы к измененным сущностям

```java
@Service
public class ModifiedEntitiesQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Object> getEntitiesModifiedAtRevision(String entityName, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesModifiedAtRevision(entityName, revision);
        
        return query.getResultList();
    }
    
    public List<User> getUsersModifiedAtRevision(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesModifiedAtRevision(User.class, revision);
        
        return query.getResultList();
    }
    
    public List<Object> getEntitiesModifiedBetweenRevisions(String entityName, 
                                                          Number fromRevision, Number toRevision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesModifiedAtRevision(entityName, fromRevision)
            .add(AuditEntity.revisionNumber().le(toRevision));
        
        return query.getResultList();
    }
}
```

## Фильтрация запросов

### Фильтрация по полям сущности

```java
@Service
public class FilteredAuditQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<User> getUsersByUsername(String username, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .add(AuditEntity.property("username").eq(username));
        
        return query.getResultList();
    }
    
    public List<User> getUsersByEmailPattern(String emailPattern, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .add(AuditEntity.property("email").like(emailPattern));
        
        return query.getResultList();
    }
    
    public List<Product> getProductsByPriceRange(BigDecimal minPrice, BigDecimal maxPrice, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Product.class, revision)
            .add(AuditEntity.property("price").between(minPrice, maxPrice));
        
        return query.getResultList();
    }
    
    public List<User> getActiveUsers(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .add(AuditEntity.property("active").eq(true));
        
        return query.getResultList();
    }
}
```

### Фильтрация по ревизиям

```java
@Service
public class RevisionFilteredQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Object> getEntitiesInRevisionRange(String entityName, Number fromRevision, Number toRevision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(entityName, toRevision)
            .add(AuditEntity.revisionNumber().ge(fromRevision))
            .add(AuditEntity.revisionNumber().le(toRevision));
        
        return query.getResultList();
    }
    
    public List<User> getUsersModifiedAfterRevision(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .add(AuditEntity.revisionNumber().gt(revision));
        
        return query.getResultList();
    }
    
    public List<Object> getEntitiesInDateRange(String entityName, Date fromDate, Date toDate) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(entityName, null)
            .add(AuditEntity.revisionProperty("timestamp").ge(fromDate.getTime()))
            .add(AuditEntity.revisionProperty("timestamp").le(toDate.getTime()));
        
        return query.getResultList();
    }
}
```

## Сложные запросы

### Запросы с множественными условиями

```java
@Service
public class ComplexAuditQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<User> getUsersByComplexCriteria(String usernamePattern, String emailDomain, 
                                              boolean active, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .add(AuditEntity.property("username").like(usernamePattern))
            .add(AuditEntity.property("email").like("%@" + emailDomain))
            .add(AuditEntity.property("active").eq(active));
        
        return query.getResultList();
    }
    
    public List<Product> getProductsByCategoryAndPrice(String category, BigDecimal minPrice, 
                                                     BigDecimal maxPrice, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Product.class, revision)
            .add(AuditEntity.property("category").eq(category))
            .add(AuditEntity.property("price").between(minPrice, maxPrice));
        
        return query.getResultList();
    }
    
    public List<Order> getOrdersByStatusAndDateRange(String status, Date fromDate, Date toDate, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Order.class, revision)
            .add(AuditEntity.property("status").eq(status))
            .add(AuditEntity.property("orderDate").ge(fromDate))
            .add(AuditEntity.property("orderDate").le(toDate));
        
        return query.getResultList();
    }
}
```

### Запросы с подзапросами

```java
@Service
public class SubqueryAuditQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<User> getUsersWithOrders(Number revision) {
        AuditReader auditReader = getAuditReader();
        
        // Подзапрос для получения пользователей с заказами
        AuditQuery subquery = auditReader.createQuery()
            .forEntitiesAtRevision(Order.class, revision)
            .addProjection(AuditEntity.property("user.id"));
        
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .add(AuditEntity.property("id").in(subquery));
        
        return query.getResultList();
    }
    
    public List<Product> getProductsInPopularCategories(Number revision) {
        AuditReader auditReader = getAuditReader();
        
        // Подзапрос для получения популярных категорий
        AuditQuery subquery = auditReader.createQuery()
            .forEntitiesAtRevision(OrderItem.class, revision)
            .addProjection(AuditEntity.property("product.category"))
            .addGroup(AuditEntity.property("product.category"))
            .addProjection(AuditEntity.count("id"))
            .add(AuditEntity.count("id").ge(10L));
        
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Product.class, revision)
            .add(AuditEntity.property("category").in(subquery));
        
        return query.getResultList();
    }
}
```

## Агрегация данных

### Группировка и агрегация

```java
@Service
public class AggregationAuditQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Object[]> getUserStatistics(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .addProjection(AuditEntity.property("active"))
            .addProjection(AuditEntity.count("id"))
            .addGroup(AuditEntity.property("active"));
        
        return query.getResultList();
    }
    
    public List<Object[]> getProductCategoryStatistics(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Product.class, revision)
            .addProjection(AuditEntity.property("category"))
            .addProjection(AuditEntity.count("id"))
            .addProjection(AuditEntity.avg("price"))
            .addProjection(AuditEntity.max("price"))
            .addProjection(AuditEntity.min("price"))
            .addGroup(AuditEntity.property("category"));
        
        return query.getResultList();
    }
    
    public List<Object[]> getOrderStatisticsByMonth(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Order.class, revision)
            .addProjection(AuditEntity.function("YEAR", AuditEntity.property("orderDate")))
            .addProjection(AuditEntity.function("MONTH", AuditEntity.property("orderDate")))
            .addProjection(AuditEntity.count("id"))
            .addProjection(AuditEntity.sum("totalAmount"))
            .addGroup(AuditEntity.function("YEAR", AuditEntity.property("orderDate")))
            .addGroup(AuditEntity.function("MONTH", AuditEntity.property("orderDate")));
        
        return query.getResultList();
    }
}
```

## Сортировка и пагинация

### Сортировка результатов

```java
@Service
public class SortedAuditQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<User> getUsersSortedByUsername(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .addOrder(AuditEntity.property("username").asc());
        
        return query.getResultList();
    }
    
    public List<Product> getProductsSortedByPrice(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Product.class, revision)
            .addOrder(AuditEntity.property("price").desc());
        
        return query.getResultList();
    }
    
    public List<Order> getOrdersSortedByDateAndAmount(Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Order.class, revision)
            .addOrder(AuditEntity.property("orderDate").desc())
            .addOrder(AuditEntity.property("totalAmount").desc());
        
        return query.getResultList();
    }
}
```

### Пагинация результатов

```java
@Service
public class PaginatedAuditQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<User> getUsersPaginated(Number revision, int page, int size) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(User.class, revision)
            .addOrder(AuditEntity.property("username").asc())
            .setFirstResult(page * size)
            .setMaxResults(size);
        
        return query.getResultList();
    }
    
    public List<Product> getProductsPaginated(Number revision, int page, int size) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(Product.class, revision)
            .addOrder(AuditEntity.property("name").asc())
            .setFirstResult(page * size)
            .setMaxResults(size);
        
        return query.getResultList();
    }
    
    public long getTotalCount(String entityName, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(entityName, revision);
        
        return query.getResultCount();
    }
}
```

## Специализированные запросы

### Запросы для анализа изменений

```java
@Service
public class ChangeAnalysisQueryService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Object[]> getEntityChangeHistory(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesAtRevision(entityName, null)
            .add(AuditEntity.id().eq(id))
            .addProjection(AuditEntity.revisionNumber())
            .addProjection(AuditEntity.revisionType())
            .addProjection(AuditEntity.revisionProperty("timestamp"))
            .addOrder(AuditEntity.revisionNumber().asc());
        
        return query.getResultList();
    }
    
    public List<Object[]> getUsersModifiedByUser(String modifiedBy, Number revision) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesModifiedAtRevision(User.class, revision)
            .add(AuditEntity.revisionProperty("username").eq(modifiedBy));
        
        return query.getResultList();
    }
    
    public List<Object[]> getEntitiesModifiedInTimeRange(String entityName, Date fromDate, Date toDate) {
        AuditReader auditReader = getAuditReader();
        AuditQuery query = auditReader.createQuery()
            .forEntitiesModifiedAtRevision(entityName, null)
            .add(AuditEntity.revisionProperty("timestamp").ge(fromDate.getTime()))
            .add(AuditEntity.revisionProperty("timestamp").le(toDate.getTime()))
            .addProjection(AuditEntity.revisionNumber())
            .addProjection(AuditEntity.revisionType())
            .addProjection(AuditEntity.revisionProperty("username"))
            .addOrder(AuditEntity.revisionProperty("timestamp").desc());
        
        return query.getResultList();
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class AuditQueryServiceTest {
    
    @Mock
    private EntityManager entityManager;
    
    @Mock
    private AuditReader auditReader;
    
    @Mock
    private AuditQuery auditQuery;
    
    @InjectMocks
    private BasicAuditQueryService service;
    
    @Test
    void getEntitiesAtRevision_ShouldReturnEntities() {
        // Given
        List<Object> expectedEntities = Arrays.asList(new User(), new User());
        when(auditQuery.getResultList()).thenReturn(expectedEntities);
        when(auditQuery.forEntitiesAtRevision("User", 5)).thenReturn(auditQuery);
        when(auditReader.createQuery()).thenReturn(auditQuery);
        
        // When
        List<Object> result = service.getEntitiesAtRevision("User", 5);
        
        // Then
        assertEquals(expectedEntities, result);
    }
}
```

### Integration тесты

```java
@SpringBootTest
class AuditQueryIntegrationTest {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private BasicAuditQueryService service;
    
    @Test
    void shouldExecuteAuditQuery() {
        // Given
        User user = new User("test", "test@example.com");
        entityManager.persist(user);
        entityManager.flush();
        
        // When
        List<Object> entities = service.getEntitiesAtRevision("User", 1);
        
        // Then
        assertFalse(entities.isEmpty());
    }
}
```

## Заключение

Audit Queries предоставляют мощные возможности для работы с аудитом в Hibernate Envers:

- **Гибкость**: Создание сложных запросов с множественными условиями
- **Фильтрация**: Точная фильтрация по различным критериям
- **Агрегация**: Группировка и агрегация данных
- **Сортировка**: Гибкая сортировка результатов
- **Пагинация**: Эффективная работа с большими объемами данных
- **Анализ**: Специализированные запросы для анализа изменений

При работе с Audit Queries важно помнить о производительности, правильной обработке ошибок и эффективном использовании ресурсов. 