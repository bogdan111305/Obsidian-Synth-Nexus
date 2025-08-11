# Класс PartTreeJpaQuery

## Обзор

`PartTreeJpaQuery` - это ключевой класс в Spring Data JPA, который отвечает за создание и выполнение запросов на основе анализа имени метода (PartTree). Этот класс является основой для автоматической генерации JPQL запросов из имен методов репозитория, таких как `findByEmail`, `findByUsernameAndActiveTrue` и т.д.

## Архитектура и принципы работы

### Базовая структура
```java
public class PartTreeJpaQuery extends AbstractJpaQuery {
    
    private final PartTree partTree;
    private final JpaMetamodel jpaMetamodel;
    private final QueryMethod queryMethod;
    
    public PartTreeJpaQuery(QueryMethod method, RepositoryMetadata metadata, EntityManager entityManager) {
        super(method, metadata, entityManager);
        this.partTree = new PartTree(method.getName(), metadata.getDomainType());
        this.jpaMetamodel = JpaMetamodel.of(entityManager.getMetamodel());
        this.queryMethod = method;
    }
    
    @Override
    public Object execute(Object[] parameters) {
        // Создание JPQL запроса
        String jpql = createQuery();
        
        // Создание TypedQuery
        TypedQuery<?> query = entityManager.createQuery(jpql, getDomainClass());
        
        // Установка параметров
        setParameters(query, parameters);
        
        // Выполнение запроса
        return executeQuery(query);
    }
    
    @Override
    protected String createQuery() {
        return generateJpql();
    }
}
```

### Иерархия классов
```
AbstractJpaQuery (абстрактный базовый класс)
    ↓
PartTreeJpaQuery (для методов с PartTree)
    ↓
CustomPartTreeJpaQuery (кастомные расширения)
```

## Детальная реализация

### Создание PartTree
```java
public class PartTreeParser {
    
    public static PartTree parse(String methodName, Class<?> domainType) {
        // Удаление префикса (find, get, read, query, search, count, exists, delete, remove)
        String queryMethod = extractQueryMethod(methodName);
        
        // Парсинг частей запроса
        List<Part> parts = parseParts(queryMethod, domainType);
        
        // Создание PartTree
        return new PartTree(parts, domainType);
    }
    
    private static String extractQueryMethod(String methodName) {
        String[] prefixes = {"find", "get", "read", "query", "search", "count", "exists", "delete", "remove"};
        
        for (String prefix : prefixes) {
            if (methodName.startsWith(prefix)) {
                return methodName.substring(prefix.length());
            }
        }
        
        return methodName;
    }
    
    private static List<Part> parseParts(String queryMethod, Class<?> domainType) {
        List<Part> parts = new ArrayList<>();
        
        // Разделение на части по And/Or
        String[] partStrings = queryMethod.split("(And|Or)");
        
        for (String partString : partStrings) {
            Part part = parsePart(partString, domainType);
            parts.add(part);
        }
        
        return parts;
    }
    
    private static Part parsePart(String partString, Class<?> domainType) {
        // Анализ части запроса
        PropertyPath propertyPath = PropertyPath.from(partString, domainType);
        Type type = determineType(partString);
        
        return new Part(propertyPath, type);
    }
    
    private static Type determineType(String partString) {
        if (partString.endsWith("IgnoreCase")) {
            return Type.IGNORE_CASE;
        } else if (partString.endsWith("Not")) {
            return Type.NOT;
        } else if (partString.endsWith("Like")) {
            return Type.LIKE;
        } else if (partString.endsWith("Containing")) {
            return Type.CONTAINING;
        } else if (partString.endsWith("StartingWith")) {
            return Type.STARTING_WITH;
        } else if (partString.endsWith("EndingWith")) {
            return Type.ENDING_WITH;
        } else if (partString.endsWith("Between")) {
            return Type.BETWEEN;
        } else if (partString.endsWith("LessThan")) {
            return Type.LESS_THAN;
        } else if (partString.endsWith("GreaterThan")) {
            return Type.GREATER_THAN;
        } else if (partString.endsWith("LessThanEqual")) {
            return Type.LESS_THAN_EQUAL;
        } else if (partString.endsWith("GreaterThanEqual")) {
            return Type.GREATER_THAN_EQUAL;
        } else if (partString.endsWith("In")) {
            return Type.IN;
        } else if (partString.endsWith("NotIn")) {
            return Type.NOT_IN;
        } else if (partString.endsWith("IsNull")) {
            return Type.IS_NULL;
        } else if (partString.endsWith("IsNotNull")) {
            return Type.IS_NOT_NULL;
        } else if (partString.endsWith("True")) {
            return Type.TRUE;
        } else if (partString.endsWith("False")) {
            return Type.FALSE;
        } else {
            return Type.EQUALS;
        }
    }
}
```

### Генерация JPQL
```java
public class JpqlGenerator {
    
    public static String generateJpql(PartTree partTree, Class<?> domainType) {
        StringBuilder jpql = new StringBuilder();
        
        // SELECT часть
        jpql.append("SELECT e FROM ").append(domainType.getSimpleName()).append(" e");
        
        // WHERE часть
        if (partTree.hasPredicate()) {
            jpql.append(" WHERE ").append(generateWhereClause(partTree));
        }
        
        // ORDER BY часть
        if (partTree.hasSort()) {
            jpql.append(" ORDER BY ").append(generateOrderByClause(partTree));
        }
        
        return jpql.toString();
    }
    
    private static String generateWhereClause(PartTree partTree) {
        StringBuilder whereClause = new StringBuilder();
        List<Part> parts = partTree.getParts();
        
        for (int i = 0; i < parts.size(); i++) {
            Part part = parts.get(i);
            
            if (i > 0) {
                // Добавление логического оператора
                whereClause.append(" ").append(part.getConnector()).append(" ");
            }
            
            whereClause.append(generatePartClause(part));
        }
        
        return whereClause.toString();
    }
    
    private static String generatePartClause(Part part) {
        PropertyPath propertyPath = part.getPropertyPath();
        Type type = part.getType();
        
        switch (type) {
            case EQUALS:
                return "e." + propertyPath.toDotPath() + " = :" + propertyPath.getLeafProperty();
            case NOT_EQUALS:
                return "e." + propertyPath.toDotPath() + " != :" + propertyPath.getLeafProperty();
            case LIKE:
                return "e." + propertyPath.toDotPath() + " LIKE :" + propertyPath.getLeafProperty();
            case NOT_LIKE:
                return "e." + propertyPath.toDotPath() + " NOT LIKE :" + propertyPath.getLeafProperty();
            case CONTAINING:
                return "e." + propertyPath.toDotPath() + " LIKE %:" + propertyPath.getLeafProperty() + "%";
            case NOT_CONTAINING:
                return "e." + propertyPath.toDotPath() + " NOT LIKE %:" + propertyPath.getLeafProperty() + "%";
            case STARTING_WITH:
                return "e." + propertyPath.toDotPath() + " LIKE :" + propertyPath.getLeafProperty() + "%";
            case ENDING_WITH:
                return "e." + propertyPath.toDotPath() + " LIKE %:" + propertyPath.getLeafProperty();
            case BETWEEN:
                return "e." + propertyPath.toDotPath() + " BETWEEN :" + propertyPath.getLeafProperty() + "Start AND :" + propertyPath.getLeafProperty() + "End";
            case LESS_THAN:
                return "e." + propertyPath.toDotPath() + " < :" + propertyPath.getLeafProperty();
            case GREATER_THAN:
                return "e." + propertyPath.toDotPath() + " > :" + propertyPath.getLeafProperty();
            case LESS_THAN_EQUAL:
                return "e." + propertyPath.toDotPath() + " <= :" + propertyPath.getLeafProperty();
            case GREATER_THAN_EQUAL:
                return "e." + propertyPath.toDotPath() + " >= :" + propertyPath.getLeafProperty();
            case IN:
                return "e." + propertyPath.toDotPath() + " IN :" + propertyPath.getLeafProperty();
            case NOT_IN:
                return "e." + propertyPath.toDotPath() + " NOT IN :" + propertyPath.getLeafProperty();
            case IS_NULL:
                return "e." + propertyPath.toDotPath() + " IS NULL";
            case IS_NOT_NULL:
                return "e." + propertyPath.toDotPath() + " IS NOT NULL";
            case TRUE:
                return "e." + propertyPath.toDotPath() + " = true";
            case FALSE:
                return "e." + propertyPath.toDotPath() + " = false";
            default:
                return "e." + propertyPath.toDotPath() + " = :" + propertyPath.getLeafProperty();
        }
    }
    
    private static String generateOrderByClause(PartTree partTree) {
        Sort sort = partTree.getSort();
        StringBuilder orderByClause = new StringBuilder();
        
        for (Sort.Order order : sort) {
            if (orderByClause.length() > 0) {
                orderByClause.append(", ");
            }
            
            orderByClause.append("e.").append(order.getProperty());
            
            if (order.isDescending()) {
                orderByClause.append(" DESC");
            } else {
                orderByClause.append(" ASC");
            }
        }
        
        return orderByClause.toString();
    }
}
```

## Типы запросов

### Простые запросы
```java
// findByEmail
public List<User> findByEmail(String email) {
    // Генерируется: SELECT e FROM User e WHERE e.email = :email
}

// findByUsername
public Optional<User> findByUsername(String username) {
    // Генерируется: SELECT e FROM User e WHERE e.username = :username
}

// findById
public User findById(Long id) {
    // Генерируется: SELECT e FROM User e WHERE e.id = :id
}
```

### Сложные запросы
```java
// findByEmailAndUsername
public User findByEmailAndUsername(String email, String username) {
    // Генерируется: SELECT e FROM User e WHERE e.email = :email AND e.username = :username
}

// findByEmailOrUsername
public List<User> findByEmailOrUsername(String email, String username) {
    // Генерируется: SELECT e FROM User e WHERE e.email = :email OR e.username = :username
}

// findByEmailAndUsernameAndActiveTrue
public List<User> findByEmailAndUsernameAndActiveTrue(String email, String username) {
    // Генерируется: SELECT e FROM User e WHERE e.email = :email AND e.username = :username AND e.active = true
}
```

### Запросы с операторами
```java
// findByEmailLike
public List<User> findByEmailLike(String emailPattern) {
    // Генерируется: SELECT e FROM User e WHERE e.email LIKE :emailPattern
}

// findByEmailContaining
public List<User> findByEmailContaining(String emailPart) {
    // Генерируется: SELECT e FROM User e WHERE e.email LIKE %:emailPart%
}

// findByEmailStartingWith
public List<User> findByEmailStartingWith(String emailPrefix) {
    // Генерируется: SELECT e FROM User e WHERE e.email LIKE :emailPrefix%
}

// findByEmailEndingWith
public List<User> findByEmailEndingWith(String emailSuffix) {
    // Генерируется: SELECT e FROM User e WHERE e.email LIKE %:emailSuffix
}
```

### Запросы с сравнениями
```java
// findByAgeGreaterThan
public List<User> findByAgeGreaterThan(int age) {
    // Генерируется: SELECT e FROM User e WHERE e.age > :age
}

// findByAgeLessThan
public List<User> findByAgeLessThan(int age) {
    // Генерируется: SELECT e FROM User e WHERE e.age < :age
}

// findByAgeBetween
public List<User> findByAgeBetween(int minAge, int maxAge) {
    // Генерируется: SELECT e FROM User e WHERE e.age BETWEEN :minAge AND :maxAge
}

// findByAgeGreaterThanEqual
public List<User> findByAgeGreaterThanEqual(int age) {
    // Генерируется: SELECT e FROM User e WHERE e.age >= :age
}
```

### Запросы с коллекциями
```java
// findByRoleIn
public List<User> findByRoleIn(List<Role> roles) {
    // Генерируется: SELECT e FROM User e WHERE e.role IN :roles
}

// findByRoleNotIn
public List<User> findByRoleNotIn(List<Role> roles) {
    // Генерируется: SELECT e FROM User e WHERE e.role NOT IN :roles
}
```

### Запросы с null проверками
```java
// findByEmailIsNull
public List<User> findByEmailIsNull() {
    // Генерируется: SELECT e FROM User e WHERE e.email IS NULL
}

// findByEmailIsNotNull
public List<User> findByEmailIsNotNull() {
    // Генерируется: SELECT e FROM User e WHERE e.email IS NOT NULL
}
```

### Запросы с булевыми значениями
```java
// findByActiveTrue
public List<User> findByActiveTrue() {
    // Генерируется: SELECT e FROM User e WHERE e.active = true
}

// findByActiveFalse
public List<User> findByActiveFalse() {
    // Генерируется: SELECT e FROM User e WHERE e.active = false
}
```

## Обработка параметров

### Установка параметров
```java
public class ParameterBinder {
    
    public static void bindParameters(TypedQuery<?> query, PartTree partTree, Object[] parameters) {
        List<Part> parts = partTree.getParts();
        
        for (int i = 0; i < parts.size(); i++) {
            Part part = parts.get(i);
            Object parameter = parameters[i];
            
            if (part.getType() == Type.BETWEEN) {
                // Обработка BETWEEN с двумя параметрами
                Object[] betweenParams = (Object[]) parameter;
                query.setParameter(part.getPropertyPath().getLeafProperty() + "Start", betweenParams[0]);
                query.setParameter(part.getPropertyPath().getLeafProperty() + "End", betweenParams[1]);
            } else {
                // Обычная установка параметра
                query.setParameter(part.getPropertyPath().getLeafProperty(), parameter);
            }
        }
    }
    
    public static void bindParametersWithType(TypedQuery<?> query, PartTree partTree, Object[] parameters) {
        List<Part> parts = partTree.getParts();
        
        for (int i = 0; i < parts.size(); i++) {
            Part part = parts.get(i);
            Object parameter = parameters[i];
            
            if (part.getType() == Type.BETWEEN) {
                Object[] betweenParams = (Object[]) parameter;
                query.setParameter(part.getPropertyPath().getLeafProperty() + "Start", betweenParams[0], part.getPropertyType());
                query.setParameter(part.getPropertyPath().getLeafProperty() + "End", betweenParams[1], part.getPropertyType());
            } else {
                query.setParameter(part.getPropertyPath().getLeafProperty(), parameter, part.getPropertyType());
            }
        }
    }
}
```

### Валидация параметров
```java
public class ParameterValidator {
    
    public static void validateParameters(PartTree partTree, Object[] parameters) {
        List<Part> parts = partTree.getParts();
        
        if (parameters.length != parts.size()) {
            throw new IllegalArgumentException(
                "Expected " + parts.size() + " parameters, but got " + parameters.length);
        }
        
        for (int i = 0; i < parts.size(); i++) {
            Part part = parts.get(i);
            Object parameter = parameters[i];
            
            validateParameter(part, parameter);
        }
    }
    
    private static void validateParameter(Part part, Object parameter) {
        if (part.getType() == Type.BETWEEN) {
            if (!(parameter instanceof Object[])) {
                throw new IllegalArgumentException("BETWEEN requires Object[] parameter");
            }
            Object[] betweenParams = (Object[]) parameter;
            if (betweenParams.length != 2) {
                throw new IllegalArgumentException("BETWEEN requires exactly 2 parameters");
            }
        }
        
        if (part.getType() == Type.IN || part.getType() == Type.NOT_IN) {
            if (!(parameter instanceof Collection)) {
                throw new IllegalArgumentException("IN/NOT_IN requires Collection parameter");
            }
        }
    }
}
```

## Оптимизация производительности

### Кэширование запросов
```java
public class PartTreeQueryCache {
    
    private final Map<String, TypedQuery<?>> queryCache = new ConcurrentHashMap<>();
    private final Map<String, PartTree> partTreeCache = new ConcurrentHashMap<>();
    
    public TypedQuery<?> getOrCreateQuery(String methodName, Class<?> domainType, EntityManager entityManager) {
        String cacheKey = generateCacheKey(methodName, domainType);
        
        return queryCache.computeIfAbsent(cacheKey, key -> {
            PartTree partTree = getOrCreatePartTree(methodName, domainType);
            String jpql = JpqlGenerator.generateJpql(partTree, domainType);
            return entityManager.createQuery(jpql, domainType);
        });
    }
    
    public PartTree getOrCreatePartTree(String methodName, Class<?> domainType) {
        String cacheKey = methodName + ":" + domainType.getName();
        
        return partTreeCache.computeIfAbsent(cacheKey, key -> 
            PartTreeParser.parse(methodName, domainType));
    }
    
    private String generateCacheKey(String methodName, Class<?> domainType) {
        return methodName + ":" + domainType.getName();
    }
}
```

### Батчинг запросов
```java
public class BatchPartTreeQueryExecutor {
    
    private final EntityManager entityManager;
    private final int batchSize;
    
    public BatchPartTreeQueryExecutor(EntityManager entityManager, int batchSize) {
        this.entityManager = entityManager;
        this.batchSize = batchSize;
    }
    
    public List<Object> executeBatchQueries(List<PartTreeJpaQuery> queries, List<Object[]> parameters) {
        List<Object> results = new ArrayList<>();
        
        for (int i = 0; i < queries.size(); i += batchSize) {
            int endIndex = Math.min(i + batchSize, queries.size());
            List<PartTreeJpaQuery> batch = queries.subList(i, endIndex);
            List<Object[]> batchParams = parameters.subList(i, endIndex);
            
            List<Object> batchResults = executeBatch(batch, batchParams);
            results.addAll(batchResults);
        }
        
        return results;
    }
    
    private List<Object> executeBatch(List<PartTreeJpaQuery> queries, List<Object[]> parameters) {
        return queries.stream()
            .map(query -> query.execute(parameters.get(queries.indexOf(query))))
            .collect(Collectors.toList());
    }
}
```

## Обработка ошибок

### Обработка исключений
```java
public class PartTreeQueryExceptionHandler {
    
    public static Object handleQueryExecution(PartTreeJpaQuery query, Object[] parameters) {
        try {
            // Валидация параметров
            ParameterValidator.validateParameters(query.getPartTree(), parameters);
            
            // Выполнение запроса
            return query.execute(parameters);
        } catch (NoResultException e) {
            return handleNoResultException(query);
        } catch (NonUniqueResultException e) {
            return handleNonUniqueResultException(query);
        } catch (PersistenceException e) {
            return handlePersistenceException(e, query);
        } catch (Exception e) {
            return handleGenericException(e, query);
        }
    }
    
    private static Object handleNoResultException(PartTreeJpaQuery query) {
        Class<?> returnType = query.getReturnType();
        
        if (returnType.equals(Optional.class)) {
            return Optional.empty();
        } else if (returnType.equals(List.class)) {
            return Collections.emptyList();
        } else {
            return null;
        }
    }
    
    private static Object handleNonUniqueResultException(PartTreeJpaQuery query) {
        throw new IncorrectResultSizeDataAccessException(
            "Query returned more than one result", 1, Integer.MAX_VALUE);
    }
    
    private static Object handlePersistenceException(PersistenceException e, PartTreeJpaQuery query) {
        throw new DataAccessException("Error executing PartTree query: " + query.getMethodName(), e);
    }
    
    private static Object handleGenericException(Exception e, PartTreeJpaQuery query) {
        throw new DataAccessException("Unexpected error executing PartTree query: " + query.getMethodName(), e);
    }
}
```

## Тестирование

### Unit тестирование
```java
@ExtendWith(MockitoExtension.class)
class PartTreeJpaQueryTest {
    
    @Mock
    private EntityManager entityManager;
    
    @Mock
    private TypedQuery<User> typedQuery;
    
    @Test
    void testQueryGeneration() {
        // given
        Method method = UserRepository.class.getMethod("findByEmail", String.class);
        QueryMethod queryMethod = new QueryMethod(method, mock(RepositoryMetadata.class));
        
        when(entityManager.createQuery(anyString(), eq(User.class))).thenReturn(typedQuery);
        when(typedQuery.setParameter(anyString(), any())).thenReturn(typedQuery);
        when(typedQuery.getResultList()).thenReturn(Arrays.asList(new User()));
        
        PartTreeJpaQuery query = new PartTreeJpaQuery(queryMethod, mock(RepositoryMetadata.class), entityManager);
        
        // when
        Object result = query.execute(new Object[]{"test@example.com"});
        
        // then
        assertNotNull(result);
        assertTrue(result instanceof List);
        verify(entityManager).createQuery(contains("SELECT e FROM User e WHERE e.email = :email"), eq(User.class));
    }
    
    @Test
    void testComplexQueryGeneration() {
        // given
        Method method = UserRepository.class.getMethod("findByEmailAndUsernameAndActiveTrue", String.class, String.class);
        QueryMethod queryMethod = new QueryMethod(method, mock(RepositoryMetadata.class));
        
        when(entityManager.createQuery(anyString(), eq(User.class))).thenReturn(typedQuery);
        when(typedQuery.setParameter(anyString(), any())).thenReturn(typedQuery);
        when(typedQuery.getResultList()).thenReturn(Arrays.asList(new User()));
        
        PartTreeJpaQuery query = new PartTreeJpaQuery(queryMethod, mock(RepositoryMetadata.class), entityManager);
        
        // when
        Object result = query.execute(new Object[]{"test@example.com", "testuser"});
        
        // then
        assertNotNull(result);
        verify(entityManager).createQuery(contains("SELECT e FROM User e WHERE e.email = :email AND e.username = :username AND e.active = true"), eq(User.class));
    }
}
```

### Интеграционное тестирование
```java
@DataJpaTest
class PartTreeJpaQueryIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testSimpleQuery() {
        // given
        User user = new User();
        user.setEmail("test@example.com");
        user.setUsername("testuser");
        userRepository.save(user);
        
        // when
        List<User> users = userRepository.findByEmail("test@example.com");
        
        // then
        assertEquals(1, users.size());
        assertEquals("test@example.com", users.get(0).getEmail());
    }
    
    @Test
    void testComplexQuery() {
        // given
        User user = new User();
        user.setEmail("test@example.com");
        user.setUsername("testuser");
        user.setActive(true);
        userRepository.save(user);
        
        // when
        List<User> users = userRepository.findByEmailAndUsernameAndActiveTrue("test@example.com", "testuser");
        
        // then
        assertEquals(1, users.size());
        assertEquals("test@example.com", users.get(0).getEmail());
        assertEquals("testuser", users.get(0).getUsername());
        assertTrue(users.get(0).isActive());
    }
}
```

## Лучшие практики

### 1. Именование методов
```java
// Правильно - четкие и понятные имена
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
    List<User> findByActiveTrue();
    List<User> findByEmailAndUsername(String email, String username);
}

// Неправильно - неясные имена
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> find(String email); // Неясно, что ищется
    User get(String username); // Неясно, что получается
}
```

### 2. Оптимизация запросов
```java
// Используйте индексы для часто используемых полей
@Entity
@Table(indexes = {
    @Index(name = "idx_user_email", columnList = "email"),
    @Index(name = "idx_user_username", columnList = "username")
})
public class User {
    // поля
}

// Используйте batch операции для множественных запросов
@Transactional
public List<User> findUsersInBatch(List<String> emails) {
    return userRepository.findByEmailIn(emails);
}
```

### 3. Валидация параметров
```java
// Добавляйте валидацию в сервисном слое
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> findByEmail(String email) {
        if (email == null || email.trim().isEmpty()) {
            throw new IllegalArgumentException("Email cannot be null or empty");
        }
        return userRepository.findByEmail(email);
    }
}
```

### 4. Обработка результатов
```java
// Правильная обработка Optional
public Optional<User> findByUsername(String username) {
    try {
        return userRepository.findByUsername(username);
    } catch (Exception e) {
        log.error("Error finding user by username: " + username, e);
        return Optional.empty();
    }
}

// Правильная обработка List
public List<User> findByActiveTrue() {
    try {
        return userRepository.findByActiveTrue();
    } catch (Exception e) {
        log.error("Error finding active users", e);
        return Collections.emptyList();
    }
}
```

## Заключение

`PartTreeJpaQuery` является мощным компонентом Spring Data JPA, который обеспечивает:

- **Автоматическую генерацию JPQL** из имен методов
- **Поддержку сложных запросов** с множественными условиями
- **Оптимизацию производительности** через кэширование
- **Гибкость в настройке** и кастомизации
- **Простоту использования** через конвенции именования

Правильное понимание и использование этого класса позволяет создавать эффективные и читаемые репозитории в Spring приложениях.
