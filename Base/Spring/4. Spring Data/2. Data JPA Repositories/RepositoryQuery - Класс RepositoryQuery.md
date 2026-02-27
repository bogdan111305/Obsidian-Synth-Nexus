# Класс RepositoryQuery

## Обзор

`RepositoryQuery` - это абстрактный базовый класс в Spring Data JPA, который представляет собой фундаментальную концепцию для выполнения запросов в репозиториях. Этот класс является основой для всех типов запросов в Spring Data JPA и обеспечивает единообразный интерфейс для выполнения различных типов запросов.

## Архитектура и иерархия

### Базовая структура
```java
public abstract class RepositoryQuery {
    
    private final QueryMethod method;
    private final RepositoryMetadata metadata;
    private final EntityManager entityManager;
    
    public RepositoryQuery(QueryMethod method, RepositoryMetadata metadata, EntityManager entityManager) {
        this.method = method;
        this.metadata = metadata;
        this.entityManager = entityManager;
    }
    
    public abstract Object execute(Object[] parameters);
    
    protected QueryMethod getQueryMethod() {
        return method;
    }
    
    protected RepositoryMetadata getRepositoryMetadata() {
        return metadata;
    }
    
    protected EntityManager getEntityManager() {
        return entityManager;
    }
}
```

### Иерархия классов
```
RepositoryQuery (абстрактный базовый класс)
    ↓
JpaQuery (специфичный для JPA)
    ↓
PartTreeJpaQuery (для методов с PartTree)
NamedQueryJpaQuery (для @NamedQuery)
@QueryJpaQuery (для @Query)
StoredProcedureJpaQuery (для хранимых процедур)
```

## Детальная реализация

### Базовый класс RepositoryQuery
```java
public abstract class RepositoryQuery {
    
    private final QueryMethod method;
    private final RepositoryMetadata metadata;
    private final EntityManager entityManager;
    private final QueryMethodFactory queryMethodFactory;
    
    public RepositoryQuery(QueryMethod method, RepositoryMetadata metadata, EntityManager entityManager) {
        this.method = method;
        this.metadata = metadata;
        this.entityManager = entityManager;
        this.queryMethodFactory = new QueryMethodFactory(entityManager);
    }
    
    /**
     * Основной метод выполнения запроса
     */
    public abstract Object execute(Object[] parameters);
    
    /**
     * Получение строки запроса
     */
    public abstract String getQueryString();
    
    /**
     * Проверка, является ли запрос нативным
     */
    public abstract boolean isNativeQuery();
    
    /**
     * Получение параметров запроса
     */
    public List<Parameter> getParameters() {
        return method.getParameters();
    }
    
    /**
     * Получение возвращаемого типа
     */
    public Class<?> getReturnType() {
        return method.getReturnType();
    }
    
    /**
     * Получение имени метода
     */
    public String getMethodName() {
        return method.getName();
    }
    
    /**
     * Создание запроса
     */
    protected Query createQuery(String queryString) {
        if (isNativeQuery()) {
            return entityManager.createNativeQuery(queryString);
        } else {
            return entityManager.createQuery(queryString, getReturnType());
        }
    }
    
    /**
     * Установка параметров запроса
     */
    protected void setParameters(Query query, Object[] parameters) {
        List<Parameter> methodParameters = getParameters();
        
        for (int i = 0; i < methodParameters.size(); i++) {
            Parameter parameter = methodParameters.get(i);
            Object value = parameters[i];
            
            if (parameter.hasName()) {
                query.setParameter(parameter.getName(), value);
            } else {
                query.setParameter(i + 1, value);
            }
        }
    }
    
    /**
     * Выполнение запроса и обработка результата
     */
    protected Object executeQuery(Query query) {
        Class<?> returnType = getReturnType();
        
        if (returnType.equals(void.class)) {
            return query.executeUpdate();
        } else if (returnType.equals(Optional.class)) {
            List<?> results = query.getResultList();
            return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
        } else if (returnType.equals(List.class)) {
            return query.getResultList();
        } else if (returnType.equals(Page.class)) {
            return createPage(query);
        } else if (returnType.equals(Slice.class)) {
            return createSlice(query);
        } else {
            List<?> results = query.getResultList();
            return results.isEmpty() ? null : results.get(0);
        }
    }
}
```

### JpaQuery - специфичная для JPA реализация
```java
public abstract class JpaQuery extends RepositoryQuery {
    
    private final JpaMetamodel jpaMetamodel;
    private final QueryMethodFactory queryMethodFactory;
    
    public JpaQuery(QueryMethod method, RepositoryMetadata metadata, EntityManager entityManager) {
        super(method, metadata, entityManager);
        this.jpaMetamodel = JpaMetamodel.of(entityManager.getMetamodel());
        this.queryMethodFactory = new QueryMethodFactory(entityManager);
    }
    
    /**
     * Получение имени сущности
     */
    protected String getEntityName() {
        return metadata.getDomainType().getSimpleName();
    }
    
    /**
     * Получение полного имени сущности
     */
    protected String getFullEntityName() {
        return metadata.getDomainType().getName();
    }
    
    /**
     * Проверка, является ли запрос нативным
     */
    @Override
    public boolean isNativeQuery() {
        return method.isNativeQuery();
    }
    
    /**
     * Создание TypedQuery для JPQL
     */
    protected <T> TypedQuery<T> createTypedQuery(String jpql, Class<T> resultType) {
        return entityManager.createQuery(jpql, resultType);
    }
    
    /**
     * Создание Query для нативных запросов
     */
    protected Query createNativeQuery(String sql) {
        return entityManager.createNativeQuery(sql);
    }
    
    /**
     * Обработка результатов с учетом типа возврата
     */
    protected Object handleResult(Object result) {
        Class<?> returnType = getReturnType();
        
        if (returnType.equals(Optional.class)) {
            return result == null ? Optional.empty() : Optional.of(result);
        } else if (returnType.equals(List.class)) {
            if (result instanceof List) {
                return result;
            }
            return Collections.singletonList(result);
        } else {
            if (result instanceof List) {
                List<?> list = (List<?>) result;
                return list.isEmpty() ? null : list.get(0);
            }
            return result;
        }
    }
}
```

## Типы запросов

### PartTreeJpaQuery
```java
public class PartTreeJpaQuery extends JpaQuery {
    
    private final PartTree partTree;
    
    public PartTreeJpaQuery(QueryMethod method, RepositoryMetadata metadata, EntityManager entityManager) {
        super(method, metadata, entityManager);
        this.partTree = new PartTree(method.getName(), metadata.getDomainType());
    }
    
    @Override
    public String getQueryString() {
        StringBuilder jpql = new StringBuilder();
        jpql.append("SELECT e FROM ").append(getEntityName()).append(" e");
        
        if (partTree.hasPredicate()) {
            jpql.append(" WHERE ").append(generateWhereClause());
        }
        
        if (partTree.hasSort()) {
            jpql.append(" ORDER BY ").append(generateOrderByClause());
        }
        
        return jpql.toString();
    }
    
    @Override
    public Object execute(Object[] parameters) {
        String jpql = getQueryString();
        TypedQuery<?> query = createTypedQuery(jpql, getReturnType());
        
        setParameters(query, parameters);
        
        return handleResult(query.getResultList());
    }
    
    private String generateWhereClause() {
        // Генерация WHERE условия на основе PartTree
        return partTree.getPredicate().toString();
    }
    
    private String generateOrderByClause() {
        // Генерация ORDER BY на основе PartTree
        return partTree.getSort().toString();
    }
}
```

### NamedQueryJpaQuery
```java
public class NamedQueryJpaQuery extends JpaQuery {
    
    private final String namedQueryName;
    
    public NamedQueryJpaQuery(QueryMethod method, RepositoryMetadata metadata, EntityManager entityManager) {
        super(method, metadata, entityManager);
        this.namedQueryName = method.getNamedQueryName();
    }
    
    @Override
    public String getQueryString() {
        // Получение запроса из @NamedQuery
        return method.getNamedQueryString();
    }
    
    @Override
    public Object execute(Object[] parameters) {
        Query query = entityManager.createNamedQuery(namedQueryName);
        
        setParameters(query, parameters);
        
        return handleResult(query.getResultList());
    }
}
```

### AnnotatedQueryJpaQuery
```java
public class AnnotatedQueryJpaQuery extends JpaQuery {
    
    private final String queryString;
    
    public AnnotatedQueryJpaQuery(QueryMethod method, RepositoryMetadata metadata, EntityManager entityManager) {
        super(method, metadata, entityManager);
        this.queryString = method.getQueryString();
    }
    
    @Override
    public String getQueryString() {
        return queryString;
    }
    
    @Override
    public Object execute(Object[] parameters) {
        Query query = createQuery(queryString);
        
        setParameters(query, parameters);
        
        return handleResult(query.getResultList());
    }
}
```

## Обработка параметров

### Анализ параметров
```java
public class ParameterAnalyzer {
    
    public static List<ParameterMetadata> analyzeParameters(Method method) {
        List<ParameterMetadata> parameters = new ArrayList<>();
        Parameter[] methodParameters = method.getParameters();
        
        for (int i = 0; i < methodParameters.length; i++) {
            Parameter parameter = methodParameters[i];
            
            ParameterMetadata metadata = new ParameterMetadata(
                parameter.getName(),
                parameter.getType(),
                parameter.getAnnotations(),
                i,
                extractParameterName(parameter)
            );
            
            parameters.add(metadata);
        }
        
        return parameters;
    }
    
    private static String extractParameterName(Parameter parameter) {
        // Проверка @Param аннотации
        Param paramAnnotation = parameter.getAnnotation(Param.class);
        if (paramAnnotation != null) {
            return paramAnnotation.value();
        }
        
        // Использование имени параметра
        return parameter.getName();
    }
}
```

### Установка параметров
```java
public class ParameterBinder {
    
    public static void bindParameters(Query query, List<Parameter> parameters, Object[] values) {
        for (int i = 0; i < parameters.size(); i++) {
            Parameter parameter = parameters.get(i);
            Object value = values[i];
            
            if (parameter.hasName()) {
                query.setParameter(parameter.getName(), value);
            } else {
                query.setParameter(i + 1, value);
            }
        }
    }
    
    public static void bindParametersWithType(Query query, List<Parameter> parameters, Object[] values) {
        for (int i = 0; i < parameters.size(); i++) {
            Parameter parameter = parameters.get(i);
            Object value = values[i];
            
            if (parameter.hasName()) {
                query.setParameter(parameter.getName(), value, parameter.getType());
            } else {
                query.setParameter(i + 1, value, parameter.getType());
            }
        }
    }
}
```

## Обработка результатов

### Обработка различных типов возврата
```java
public class ResultHandler {
    
    public static Object handleResult(Object result, Class<?> returnType) {
        if (returnType.equals(void.class)) {
            return null;
        } else if (returnType.equals(Optional.class)) {
            return handleOptionalResult(result);
        } else if (returnType.equals(List.class)) {
            return handleListResult(result);
        } else if (returnType.equals(Page.class)) {
            return handlePageResult(result);
        } else if (returnType.equals(Slice.class)) {
            return handleSliceResult(result);
        } else {
            return handleSingleResult(result);
        }
    }
    
    private static Object handleOptionalResult(Object result) {
        if (result == null) {
            return Optional.empty();
        }
        return Optional.of(result);
    }
    
    private static Object handleListResult(Object result) {
        if (result instanceof List) {
            return result;
        }
        return Collections.singletonList(result);
    }
    
    private static Object handleSingleResult(Object result) {
        if (result instanceof List) {
            List<?> list = (List<?>) result;
            return list.isEmpty() ? null : list.get(0);
        }
        return result;
    }
    
    private static Object handlePageResult(Object result) {
        // Обработка пагинации
        if (result instanceof Page) {
            return result;
        }
        // Создание Page из List
        return new PageImpl<>((List<?>) result);
    }
    
    private static Object handleSliceResult(Object result) {
        // Обработка Slice
        if (result instanceof Slice) {
            return result;
        }
        // Создание Slice из List
        return new SliceImpl<>((List<?>) result);
    }
}
```

### Обработка пагинации
```java
public class PaginationHandler {
    
    public static Page<?> createPage(List<?> content, Pageable pageable, long total) {
        return new PageImpl<>(content, pageable, total);
    }
    
    public static Slice<?> createSlice(List<?> content, Pageable pageable, boolean hasNext) {
        return new SliceImpl<>(content, pageable, hasNext);
    }
    
    public static long countTotal(RepositoryQuery query, Object[] parameters) {
        // Создание count запроса
        String countQuery = generateCountQuery(query);
        Query countQueryObj = query.getEntityManager().createQuery(countQuery);
        
        // Установка параметров
        ParameterBinder.bindParameters(countQueryObj, query.getParameters(), parameters);
        
        // Выполнение
        return ((Number) countQueryObj.getSingleResult()).longValue();
    }
    
    private static String generateCountQuery(RepositoryQuery query) {
        String originalQuery = query.getQueryString();
        
        // Извлечение FROM части
        int fromIndex = originalQuery.indexOf("FROM");
        if (fromIndex == -1) {
            throw new IllegalArgumentException("Invalid query: " + originalQuery);
        }
        
        // Создание count запроса
        return "SELECT COUNT(*) " + originalQuery.substring(fromIndex);
    }
}
```

## Оптимизация производительности

### Кэширование запросов
```java
public class QueryCache {
    
    private final Map<String, Query> queryCache = new ConcurrentHashMap<>();
    private final Map<Method, RepositoryQuery> queryMethodCache = new ConcurrentHashMap<>();
    
    public Query getOrCreateQuery(String queryString, RepositoryQuery repositoryQuery) {
        String cacheKey = generateCacheKey(queryString, repositoryQuery);
        
        return queryCache.computeIfAbsent(cacheKey, key -> {
            if (repositoryQuery.isNativeQuery()) {
                return repositoryQuery.getEntityManager().createNativeQuery(queryString);
            } else {
                return repositoryQuery.getEntityManager().createQuery(queryString, repositoryQuery.getReturnType());
            }
        });
    }
    
    public RepositoryQuery getOrCreateRepositoryQuery(Method method, RepositoryMetadata metadata, EntityManager entityManager) {
        return queryMethodCache.computeIfAbsent(method, 
            m -> createRepositoryQuery(m, metadata, entityManager));
    }
    
    private RepositoryQuery createRepositoryQuery(Method method, RepositoryMetadata metadata, EntityManager entityManager) {
        QueryMethod queryMethod = new QueryMethod(method, metadata);
        
        if (queryMethod.hasQueryAnnotation()) {
            return new AnnotatedQueryJpaQuery(queryMethod, metadata, entityManager);
        } else if (queryMethod.hasNamedQueryAnnotation()) {
            return new NamedQueryJpaQuery(queryMethod, metadata, entityManager);
        } else {
            return new PartTreeJpaQuery(queryMethod, metadata, entityManager);
        }
    }
    
    private String generateCacheKey(String queryString, RepositoryQuery repositoryQuery) {
        return queryString + ":" + repositoryQuery.getReturnType().getName();
    }
}
```

### Батчинг запросов
```java
public class BatchQueryExecutor {
    
    private final EntityManager entityManager;
    private final int batchSize;
    
    public BatchQueryExecutor(EntityManager entityManager, int batchSize) {
        this.entityManager = entityManager;
        this.batchSize = batchSize;
    }
    
    public List<Object> executeBatchQueries(List<RepositoryQuery> queries, List<Object[]> parameters) {
        List<Object> results = new ArrayList<>();
        
        for (int i = 0; i < queries.size(); i += batchSize) {
            int endIndex = Math.min(i + batchSize, queries.size());
            List<RepositoryQuery> batch = queries.subList(i, endIndex);
            List<Object[]> batchParams = parameters.subList(i, endIndex);
            
            List<Object> batchResults = executeBatch(batch, batchParams);
            results.addAll(batchResults);
        }
        
        return results;
    }
    
    private List<Object> executeBatch(List<RepositoryQuery> queries, List<Object[]> parameters) {
        return queries.stream()
            .map(query -> query.execute(parameters.get(queries.indexOf(query))))
            .collect(Collectors.toList());
    }
}
```

## Обработка ошибок

### Обработка исключений
```java
public class QueryExceptionHandler {
    
    public static Object handleQueryExecution(RepositoryQuery query, Object[] parameters) {
        try {
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
    
    private static Object handleNoResultException(RepositoryQuery query) {
        Class<?> returnType = query.getReturnType();
        
        if (returnType.equals(Optional.class)) {
            return Optional.empty();
        } else if (returnType.equals(List.class)) {
            return Collections.emptyList();
        } else {
            return null;
        }
    }
    
    private static Object handleNonUniqueResultException(RepositoryQuery query) {
        throw new IncorrectResultSizeDataAccessException(
            "Query returned more than one result", 1, Integer.MAX_VALUE);
    }
    
    private static Object handlePersistenceException(PersistenceException e, RepositoryQuery query) {
        throw new DataAccessException("Error executing query: " + query.getMethodName(), e);
    }
    
    private static Object handleGenericException(Exception e, RepositoryQuery query) {
        throw new DataAccessException("Unexpected error executing query: " + query.getMethodName(), e);
    }
}
```

## Тестирование

### Unit тестирование
```java
@ExtendWith(MockitoExtension.class)
class RepositoryQueryTest {
    
    @Mock
    private EntityManager entityManager;
    
    @Mock
    private TypedQuery<User> typedQuery;
    
    @Mock
    private QueryMethod queryMethod;
    
    @Test
    void testQueryExecution() {
        // given
        RepositoryMetadata metadata = mock(RepositoryMetadata.class);
        when(metadata.getDomainType()).thenReturn(User.class);
        
        when(queryMethod.getName()).thenReturn("findByEmail");
        when(queryMethod.getReturnType()).thenReturn(List.class);
        when(queryMethod.getParameters()).thenReturn(Collections.emptyList());
        
        when(entityManager.createQuery(anyString(), eq(User.class))).thenReturn(typedQuery);
        when(typedQuery.getResultList()).thenReturn(Arrays.asList(new User()));
        
        RepositoryQuery repositoryQuery = new TestRepositoryQuery(queryMethod, metadata, entityManager);
        
        // when
        Object result = repositoryQuery.execute(new Object[]{"test@example.com"});
        
        // then
        assertNotNull(result);
        assertTrue(result instanceof List);
        verify(entityManager).createQuery(contains("SELECT"), eq(User.class));
    }
}

class TestRepositoryQuery extends RepositoryQuery {
    
    public TestRepositoryQuery(QueryMethod method, RepositoryMetadata metadata, EntityManager entityManager) {
        super(method, metadata, entityManager);
    }
    
    @Override
    public Object execute(Object[] parameters) {
        String queryString = getQueryString();
        Query query = createQuery(queryString);
        setParameters(query, parameters);
        return executeQuery(query);
    }
    
    @Override
    public String getQueryString() {
        return "SELECT e FROM User e WHERE e.email = :email";
    }
    
    @Override
    public boolean isNativeQuery() {
        return false;
    }
}
```

### Интеграционное тестирование
```java
@DataJpaTest
class RepositoryQueryIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testQueryExecution_Integration() {
        // given
        User user = new User();
        user.setEmail("test@example.com");
        user.setUsername("testuser");
        userRepository.save(user);
        
        // when
        List<User> users = userRepository.findByEmail("test@example.com");
        Optional<User> userOpt = userRepository.findByUsername("testuser");
        
        // then
        assertEquals(1, users.size());
        assertTrue(userOpt.isPresent());
        assertEquals("testuser", userOpt.get().getUsername());
    }
}
```

## Лучшие практики

### 1. Кэширование запросов
```java
// Используйте кэширование для улучшения производительности
@Configuration
public class OptimizedJpaConfig {
    
    @Bean
    public QueryCache queryCache() {
        return new QueryCache();
    }
}
```

### 2. Обработка ошибок
```java
// Всегда обрабатывайте исключения при выполнении запросов
try {
    return repositoryQuery.execute(parameters);
} catch (NoResultException e) {
    return handleNoResult(repositoryQuery);
} catch (NonUniqueResultException e) {
    throw new IncorrectResultSizeDataAccessException("Multiple results found", 1, Integer.MAX_VALUE);
}
```

### 3. Оптимизация запросов
```java
// Используйте батчинг для множественных запросов
@Transactional
public List<User> findUsersInBatch(List<String> emails) {
    return batchQueryExecutor.executeBatchQueries(
        emails.stream().map(this::createFindByEmailQuery).collect(Collectors.toList()),
        emails.stream().map(email -> new Object[]{email}).collect(Collectors.toList())
    );
}
```

### 4. Мониторинг производительности
```java
@Component
public class QueryPerformanceMonitor {
    
    @EventListener(ApplicationReadyEvent.class)
    public void monitorQueryPerformance() {
        // Логика мониторинга производительности запросов
    }
}
```

## Заключение

`RepositoryQuery` является фундаментальным классом в Spring Data JPA, который обеспечивает:
- Единообразный интерфейс для выполнения различных типов запросов
- Поддержку различных типов запросов (PartTree, NamedQuery, @Query)
- Кэширование для улучшения производительности
- Обработку ошибок и исключений
- Гибкость в настройке и кастомизации

Правильное понимание и использование этого класса позволяет создавать эффективные и надежные репозитории в Spring приложениях.
