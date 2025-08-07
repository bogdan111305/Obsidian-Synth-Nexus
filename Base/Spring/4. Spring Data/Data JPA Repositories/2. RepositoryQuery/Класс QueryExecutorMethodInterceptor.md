# Класс QueryExecutorMethodInterceptor

## Обзор

`QueryExecutorMethodInterceptor` - это ключевой класс в Spring Data JPA, который отвечает за перехват вызовов методов репозитория и их выполнение через соответствующие Query объекты.

## Основные возможности

### Перехват вызовов методов
- Анализ сигнатуры метода
- Определение типа операции
- Создание и выполнение Query
- Обработка результатов

### Структура класса

```java
public class QueryExecutorMethodInterceptor implements MethodInterceptor {
    
    private final RepositoryFactory repositoryFactory;
    private final QueryMethod queryMethod;
    private final QueryExecutor queryExecutor;
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // Логика перехвата и выполнения
        return queryExecutor.execute(queryMethod, invocation.getArguments());
    }
}
```

## Жизненный цикл

### 1. Инициализация
```java
public QueryExecutorMethodInterceptor(RepositoryFactory repositoryFactory) {
    this.repositoryFactory = repositoryFactory;
    this.queryExecutor = new DefaultQueryExecutor();
}
```

### 2. Анализ метода
```java
public QueryMethod analyzeMethod(Method method) {
    String methodName = method.getName();
    Class<?>[] parameterTypes = method.getParameterTypes();
    Class<?> returnType = method.getReturnType();
    
    return new QueryMethod(method, methodName, parameterTypes, returnType);
}
```

### 3. Создание Query
```java
public Query createQuery(QueryMethod queryMethod) {
    if (queryMethod.isNamedQuery()) {
        return createNamedQuery(queryMethod);
    } else if (queryMethod.isQueryAnnotation()) {
        return createAnnotatedQuery(queryMethod);
    } else {
        return createDerivedQuery(queryMethod);
    }
}
```

## Типы Query

### Named Query
```java
private Query createNamedQuery(QueryMethod queryMethod) {
    String queryName = queryMethod.getNamedQueryName();
    return entityManager.createNamedQuery(queryName, queryMethod.getReturnType());
}
```

### @Query Annotation
```java
private Query createAnnotatedQuery(QueryMethod queryMethod) {
    String queryString = queryMethod.getQueryString();
    Query query = entityManager.createQuery(queryString);
    
    // Установка параметров
    setQueryParameters(query, queryMethod, arguments);
    
    return query;
}
```

### Derived Query
```java
private Query createDerivedQuery(QueryMethod queryMethod) {
    PartTree partTree = new PartTree(queryMethod.getMethodName(), 
                                   queryMethod.getDomainClass());
    
    return new PartTreeJpaQuery(partTree, entityManager, 
                               queryMethod.getDomainClass());
}
```

## Обработка параметров

### Позиционные параметры
```java
private void setPositionalParameters(Query query, Object[] arguments) {
    for (int i = 0; i < arguments.length; i++) {
        query.setParameter(i + 1, arguments[i]);
    }
}
```

### Именованные параметры
```java
private void setNamedParameters(Query query, QueryMethod queryMethod, 
                              Object[] arguments) {
    Parameter[] parameters = queryMethod.getParameters();
    
    for (int i = 0; i < parameters.length; i++) {
        Parameter parameter = parameters[i];
        String paramName = parameter.getName();
        Object value = arguments[i];
        
        query.setParameter(paramName, value);
    }
}
```

## Выполнение Query

### Типы возвращаемых значений
```java
public Object executeQuery(Query query, QueryMethod queryMethod) {
    Class<?> returnType = queryMethod.getReturnType();
    
    if (returnType == List.class) {
        return query.getResultList();
    } else if (returnType == Optional.class) {
        List<?> results = query.getResultList();
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    } else if (returnType == long.class || returnType == Long.class) {
        return query.getSingleResult();
    } else {
        return query.getSingleResult();
    }
}
```

### Обработка исключений
```java
public Object executeSafely(QueryMethod queryMethod, Object[] arguments) {
    try {
        Query query = createQuery(queryMethod);
        return executeQuery(query, queryMethod);
    } catch (NoResultException e) {
        if (queryMethod.isOptionalReturnType()) {
            return Optional.empty();
        }
        throw e;
    } catch (NonUniqueResultException e) {
        throw new IncorrectResultSizeDataAccessException(
            "Query returned more than one result", 1, e);
    }
}
```

## Кэширование

### Кэш Query объектов
```java
public class QueryCache {
    
    private static final Map<Method, Query> queryCache = new ConcurrentHashMap<>();
    
    public static Query getOrCreateQuery(Method method, EntityManager entityManager) {
        return queryCache.computeIfAbsent(method, 
            m -> createQuery(m, entityManager));
    }
}
```

### Кэш QueryMethod
```java
public class QueryMethodCache {
    
    private static final Map<Method, QueryMethod> methodCache = new ConcurrentHashMap<>();
    
    public static QueryMethod getOrCreateQueryMethod(Method method) {
        return methodCache.computeIfAbsent(method, 
            QueryMethod::new);
    }
}
```

## Производительность

### Оптимизация создания Query
```java
public class OptimizedQueryExecutor {
    
    private final QueryCache queryCache;
    private final QueryMethodCache methodCache;
    
    public Object execute(Method method, Object[] arguments) {
        QueryMethod queryMethod = methodCache.getOrCreateQueryMethod(method);
        Query query = queryCache.getOrCreateQuery(method, entityManager);
        
        setParameters(query, queryMethod, arguments);
        return executeQuery(query, queryMethod);
    }
}
```

### Мониторинг производительности
```java
@Component
public class QueryPerformanceMonitor {
    
    private final Map<String, Long> executionTimes = new ConcurrentHashMap<>();
    
    public Object monitorExecution(String methodName, Supplier<Object> execution) {
        long startTime = System.currentTimeMillis();
        try {
            return execution.get();
        } finally {
            long executionTime = System.currentTimeMillis() - startTime;
            executionTimes.merge(methodName, executionTime, Long::sum);
        }
    }
}
```

## Логирование и отладка

### Включение логирования
```properties
logging.level.org.springframework.data.jpa.repository.query=DEBUG
logging.level.org.springframework.data.jpa.repository.support=DEBUG
```

### Отладочная информация
```java
public class QueryDebugInterceptor implements MethodInterceptor {
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        Object[] arguments = invocation.getArguments();
        
        System.out.println("Executing method: " + method.getName());
        System.out.println("Arguments: " + Arrays.toString(arguments));
        
        Object result = invocation.proceed();
        
        System.out.println("Result: " + result);
        return result;
    }
}
```

## Лучшие практики

1. **Используйте кэширование** для Query объектов
2. **Мониторьте производительность** выполнения запросов
3. **Обрабатывайте исключения** корректно
4. **Используйте логирование** для отладки
5. **Оптимизируйте создание Query** для часто используемых методов
6. **Тестируйте различные типы Query** в unit-тестах 