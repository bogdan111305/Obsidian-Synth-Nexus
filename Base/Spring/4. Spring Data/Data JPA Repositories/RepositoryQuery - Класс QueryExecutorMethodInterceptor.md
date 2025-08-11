# Класс QueryExecutorMethodInterceptor

## Обзор

`QueryExecutorMethodInterceptor` - это ключевой класс в Spring Data JPA, который отвечает за перехват и выполнение методов репозитория. Этот интерцептор является частью механизма проксирования и обеспечивает автоматическое создание и выполнение запросов на основе сигнатуры методов репозитория.

## Архитектура и принципы работы

### Основная структура
```java
public class QueryExecutorMethodInterceptor implements MethodInterceptor {
    
    private final RepositoryFactory repositoryFactory;
    private final Map<Method, QueryMethod> queryMethodCache = new ConcurrentHashMap<>();
    private final EntityManager entityManager;
    
    public QueryExecutorMethodInterceptor(RepositoryFactory repositoryFactory, 
                                       EntityManager entityManager) {
        this.repositoryFactory = repositoryFactory;
        this.entityManager = entityManager;
    }
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        Object[] arguments = invocation.getArguments();
        
        // Получение или создание QueryMethod
        QueryMethod queryMethod = getOrCreateQueryMethod(method);
        
        // Выполнение запроса
        return executeQuery(queryMethod, arguments);
    }
}
```

### Иерархия классов
```
MethodInterceptor (Spring AOP)
    ↓
QueryExecutorMethodInterceptor (Spring Data JPA)
    ↓
JpaQueryMethod (специфичный для JPA)
    ↓
PartTreeJpaQuery (для методов с PartTree)
NamedQueryJpaQuery (для @NamedQuery)
@QueryJpaQuery (для @Query)
```

## Детальная реализация

### Создание QueryMethod
```java
public class QueryMethodFactory {
    
    private final EntityManager entityManager;
    private final JpaMetamodel jpaMetamodel;
    
    public QueryMethodFactory(EntityManager entityManager) {
        this.entityManager = entityManager;
        this.jpaMetamodel = JpaMetamodel.of(entityManager.getMetamodel());
    }
    
    public QueryMethod createQueryMethod(Method method, RepositoryMetadata metadata) {
        // Анализ метода
        MethodSignature signature = new MethodSignature(method);
        
        // Определение типа запроса
        QueryMethodType queryType = determineQueryType(signature);
        
        switch (queryType) {
            case PART_TREE:
                return new PartTreeJpaQuery(method, metadata, jpaMetamodel);
            case NAMED_QUERY:
                return new NamedQueryJpaQuery(method, metadata, jpaMetamodel);
            case ANNOTATED_QUERY:
                return new AnnotatedQueryJpaQuery(method, metadata, jpaMetamodel);
            case CUSTOM:
                return new CustomQueryJpaQuery(method, metadata, jpaMetamodel);
            default:
                throw new UnsupportedOperationException("Unknown query type: " + queryType);
        }
    }
    
    private QueryMethodType determineQueryType(MethodSignature signature) {
        // Проверка наличия @Query аннотации
        if (signature.hasQueryAnnotation()) {
            return QueryMethodType.ANNOTATED_QUERY;
        }
        
        // Проверка наличия @NamedQuery
        if (signature.hasNamedQueryAnnotation()) {
            return QueryMethodType.NAMED_QUERY;
        }
        
        // Проверка соответствия PartTree паттерну
        if (signature.matchesPartTreePattern()) {
            return QueryMethodType.PART_TREE;
        }
        
        return QueryMethodType.CUSTOM;
    }
}
```

### Кэширование QueryMethod
```java
public class QueryMethodCache {
    
    private final Map<Method, QueryMethod> cache = new ConcurrentHashMap<>();
    private final QueryMethodFactory factory;
    
    public QueryMethodCache(QueryMethodFactory factory) {
        this.factory = factory;
    }
    
    public QueryMethod getOrCreateQueryMethod(Method method, RepositoryMetadata metadata) {
        return cache.computeIfAbsent(method, 
            m -> factory.createQueryMethod(m, metadata));
    }
    
    public void clearCache() {
        cache.clear();
    }
    
    public int getCacheSize() {
        return cache.size();
    }
}
```

### Выполнение запросов
```java
public class QueryExecutor {
    
    private final EntityManager entityManager;
    private final QueryMethodFactory queryMethodFactory;
    
    public QueryExecutor(EntityManager entityManager) {
        this.entityManager = entityManager;
        this.queryMethodFactory = new QueryMethodFactory(entityManager);
    }
    
    public Object executeQuery(QueryMethod queryMethod, Object[] arguments) {
        // Создание запроса
        Query query = createQuery(queryMethod, arguments);
        
        // Установка параметров
        setParameters(query, queryMethod, arguments);
        
        // Выполнение запроса
        return executeQuery(query, queryMethod.getReturnType());
    }
    
    private Query createQuery(QueryMethod queryMethod, Object[] arguments) {
        String queryString = queryMethod.getQueryString();
        
        if (queryMethod.isNativeQuery()) {
            return entityManager.createNativeQuery(queryString);
        } else {
            return entityManager.createQuery(queryString, queryMethod.getReturnType());
        }
    }
    
    private void setParameters(Query query, QueryMethod queryMethod, Object[] arguments) {
        List<Parameter> parameters = queryMethod.getParameters();
        
        for (int i = 0; i < parameters.size(); i++) {
            Parameter parameter = parameters.get(i);
            Object value = arguments[i];
            
            if (parameter.hasName()) {
                query.setParameter(parameter.getName(), value);
            } else {
                query.setParameter(i + 1, value);
            }
        }
    }
    
    private Object executeQuery(Query query, Class<?> returnType) {
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

## Анализ методов

### Анализ сигнатуры метода
```java
public class MethodSignatureAnalyzer {
    
    public static MethodSignature analyze(Method method) {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        Class<?> returnType = method.getReturnType();
        Annotation[] annotations = method.getAnnotations();
        
        return new MethodSignature(
            methodName,
            parameterTypes,
            returnType,
            annotations,
            method.getGenericParameterTypes(),
            method.getGenericReturnType()
        );
    }
    
    public static boolean isQueryMethod(Method method) {
        String methodName = method.getName();
        
        // Проверка стандартных префиксов
        return methodName.startsWith("find") ||
               methodName.startsWith("get") ||
               methodName.startsWith("read") ||
               methodName.startsWith("query") ||
               methodName.startsWith("search") ||
               methodName.startsWith("count") ||
               methodName.startsWith("exists") ||
               methodName.startsWith("delete") ||
               methodName.startsWith("remove");
    }
    
    public static QueryType determineQueryType(Method method) {
        // Проверка аннотаций
        if (method.isAnnotationPresent(Query.class)) {
            return QueryType.ANNOTATED;
        }
        
        if (method.isAnnotationPresent(NamedQuery.class)) {
            return QueryType.NAMED;
        }
        
        // Проверка имени метода
        String methodName = method.getName();
        if (methodName.startsWith("find") || methodName.startsWith("get")) {
            return QueryType.PART_TREE;
        }
        
        return QueryType.CUSTOM;
    }
}
```

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

## Типы запросов

### PartTree запросы
```java
public class PartTreeQueryExecutor {
    
    public Object executePartTreeQuery(PartTreeJpaQuery queryMethod, Object[] arguments) {
        // Парсинг PartTree
        PartTree partTree = queryMethod.getPartTree();
        
        // Генерация JPQL
        String jpql = generateJpql(partTree);
        
        // Создание запроса
        TypedQuery<?> query = entityManager.createQuery(jpql, queryMethod.getReturnType());
        
        // Установка параметров
        setPartTreeParameters(query, partTree, arguments);
        
        // Выполнение
        return executeQuery(query, queryMethod.getReturnType());
    }
    
    private String generateJpql(PartTree partTree) {
        StringBuilder jpql = new StringBuilder();
        jpql.append("SELECT e FROM ").append(getEntityName()).append(" e");
        
        if (partTree.hasPredicate()) {
            jpql.append(" WHERE ").append(generateWhereClause(partTree));
        }
        
        if (partTree.hasSort()) {
            jpql.append(" ORDER BY ").append(generateOrderByClause(partTree));
        }
        
        return jpql.toString();
    }
}
```

### NamedQuery запросы
```java
public class NamedQueryExecutor {
    
    public Object executeNamedQuery(NamedQueryJpaQuery queryMethod, Object[] arguments) {
        // Получение имени запроса
        String queryName = queryMethod.getNamedQueryName();
        
        // Создание запроса
        Query query = entityManager.createNamedQuery(queryName);
        
        // Установка параметров
        setNamedQueryParameters(query, queryMethod, arguments);
        
        // Выполнение
        return executeQuery(query, queryMethod.getReturnType());
    }
    
    private void setNamedQueryParameters(Query query, NamedQueryJpaQuery queryMethod, Object[] arguments) {
        List<Parameter> parameters = queryMethod.getParameters();
        
        for (int i = 0; i < parameters.size(); i++) {
            Parameter parameter = parameters.get(i);
            Object value = arguments[i];
            
            query.setParameter(parameter.getName(), value);
        }
    }
}
```

### @Query запросы
```java
public class AnnotatedQueryExecutor {
    
    public Object executeAnnotatedQuery(AnnotatedQueryJpaQuery queryMethod, Object[] arguments) {
        // Получение запроса из аннотации
        String queryString = queryMethod.getQueryString();
        
        // Создание запроса
        Query query = createQuery(queryString, queryMethod);
        
        // Установка параметров
        setAnnotatedQueryParameters(query, queryMethod, arguments);
        
        // Выполнение
        return executeQuery(query, queryMethod.getReturnType());
    }
    
    private Query createQuery(String queryString, AnnotatedQueryJpaQuery queryMethod) {
        if (queryMethod.isNativeQuery()) {
            return entityManager.createNativeQuery(queryString);
        } else {
            return entityManager.createQuery(queryString, queryMethod.getReturnType());
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
    
    public static long countTotal(QueryMethod queryMethod, Object[] arguments) {
        // Создание count запроса
        String countQuery = generateCountQuery(queryMethod);
        Query countQueryObj = entityManager.createQuery(countQuery);
        
        // Установка параметров
        setParameters(countQueryObj, queryMethod, arguments);
        
        // Выполнение
        return ((Number) countQueryObj.getSingleResult()).longValue();
    }
}
```

## Оптимизация производительности

### Кэширование запросов
```java
public class QueryCache {
    
    private final Map<String, Query> queryCache = new ConcurrentHashMap<>();
    private final Map<Method, QueryMethod> methodCache = new ConcurrentHashMap<>();
    
    public Query getOrCreateQuery(String queryString, QueryMethod queryMethod) {
        String cacheKey = generateCacheKey(queryString, queryMethod);
        
        return queryCache.computeIfAbsent(cacheKey, key -> {
            if (queryMethod.isNativeQuery()) {
                return entityManager.createNativeQuery(queryString);
            } else {
                return entityManager.createQuery(queryString, queryMethod.getReturnType());
            }
        });
    }
    
    public QueryMethod getOrCreateQueryMethod(Method method, RepositoryMetadata metadata) {
        return methodCache.computeIfAbsent(method, 
            m -> queryMethodFactory.createQueryMethod(m, metadata));
    }
    
    private String generateCacheKey(String queryString, QueryMethod queryMethod) {
        return queryString + ":" + queryMethod.getReturnType().getName();
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
    
    public List<Object> executeBatchQueries(List<QueryMethod> queryMethods, List<Object[]> arguments) {
        List<Object> results = new ArrayList<>();
        
        for (int i = 0; i < queryMethods.size(); i += batchSize) {
            int endIndex = Math.min(i + batchSize, queryMethods.size());
            List<QueryMethod> batch = queryMethods.subList(i, endIndex);
            List<Object[]> batchArgs = arguments.subList(i, endIndex);
            
            List<Object> batchResults = executeBatch(batch, batchArgs);
            results.addAll(batchResults);
        }
        
        return results;
    }
    
    private List<Object> executeBatch(List<QueryMethod> queryMethods, List<Object[]> arguments) {
        // Выполнение батча запросов
        return queryMethods.stream()
            .map(queryMethod -> executeQuery(queryMethod, arguments.get(queryMethods.indexOf(queryMethod))))
            .collect(Collectors.toList());
    }
}
```

## Обработка ошибок

### Обработка исключений
```java
public class QueryExceptionHandler {
    
    public static Object handleQueryExecution(MethodInvocation invocation, QueryMethod queryMethod) {
        try {
            return executeQuery(queryMethod, invocation.getArguments());
        } catch (NoResultException e) {
            return handleNoResultException(queryMethod);
        } catch (NonUniqueResultException e) {
            return handleNonUniqueResultException(queryMethod);
        } catch (PersistenceException e) {
            return handlePersistenceException(e, queryMethod);
        } catch (Exception e) {
            return handleGenericException(e, queryMethod);
        }
    }
    
    private static Object handleNoResultException(QueryMethod queryMethod) {
        Class<?> returnType = queryMethod.getReturnType();
        
        if (returnType.equals(Optional.class)) {
            return Optional.empty();
        } else if (returnType.equals(List.class)) {
            return Collections.emptyList();
        } else {
            return null;
        }
    }
    
    private static Object handleNonUniqueResultException(QueryMethod queryMethod) {
        throw new IncorrectResultSizeDataAccessException(
            "Query returned more than one result", 1, Integer.MAX_VALUE);
    }
    
    private static Object handlePersistenceException(PersistenceException e, QueryMethod queryMethod) {
        throw new DataAccessException("Error executing query: " + queryMethod.getName(), e);
    }
}
```

## Тестирование

### Unit тестирование
```java
@ExtendWith(MockitoExtension.class)
class QueryExecutorMethodInterceptorTest {
    
    @Mock
    private EntityManager entityManager;
    
    @Mock
    private TypedQuery<User> typedQuery;
    
    @Mock
    private RepositoryFactory repositoryFactory;
    
    @Test
    void testQueryExecution() throws Throwable {
        // given
        Method method = UserRepository.class.getMethod("findByEmail", String.class);
        MethodInvocation invocation = new MethodInvocation() {
            @Override
            public Method getMethod() { return method; }
            @Override
            public Object[] getArguments() { return new Object[]{"test@example.com"}; }
            @Override
            public Object proceed() { return null; }
        };
        
        when(entityManager.createQuery(anyString(), eq(User.class))).thenReturn(typedQuery);
        when(typedQuery.setParameter(anyString(), any())).thenReturn(typedQuery);
        when(typedQuery.getResultList()).thenReturn(Arrays.asList(new User()));
        
        QueryExecutorMethodInterceptor interceptor = 
            new QueryExecutorMethodInterceptor(repositoryFactory, entityManager);
        
        // when
        Object result = interceptor.invoke(invocation);
        
        // then
        assertNotNull(result);
        assertTrue(result instanceof List);
        verify(entityManager).createQuery(contains("SELECT"), eq(User.class));
    }
}
```

### Интеграционное тестирование
```java
@DataJpaTest
class QueryExecutorMethodInterceptorIntegrationTest {
    
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

### 1. Кэширование QueryMethod
```java
// Используйте кэширование для улучшения производительности
@Configuration
public class OptimizedJpaConfig {
    
    @Bean
    public QueryMethodCache queryMethodCache() {
        return new QueryMethodCache(queryMethodFactory());
    }
}
```

### 2. Обработка ошибок
```java
// Всегда обрабатывайте исключения при выполнении запросов
try {
    return queryExecutor.executeQuery(queryMethod, arguments);
} catch (NoResultException e) {
    return handleNoResult(queryMethod);
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

`QueryExecutorMethodInterceptor` является ключевым компонентом Spring Data JPA, который обеспечивает:
- Автоматическое создание и выполнение запросов
- Поддержку различных типов запросов (PartTree, NamedQuery, @Query)
- Кэширование для улучшения производительности
- Обработку ошибок и исключений
- Гибкость в настройке и кастомизации

Правильное понимание и использование этого интерцептора позволяет создавать эффективные и надежные репозитории в Spring приложениях.
