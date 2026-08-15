# Класс PartTreeJpaQuery

## Обзор

`PartTreeJpaQuery` - это класс в Spring Data JPA, который отвечает за создание и выполнение запросов на основе анализа имени метода репозитория. Он парсит имя метода и генерирует соответствующий JPQL запрос.

## Основные возможности

### Парсинг имени метода
- Анализ префиксов (find, get, read, query, count, exists)
- Извлечение свойств сущности
- Определение операторов сравнения
- Генерация JPQL запроса

### Структура класса

```java
public class PartTreeJpaQuery implements RepositoryQuery {
    
    private final PartTree partTree;
    private final EntityManager entityManager;
    private final Class<?> domainClass;
    private final QueryMethod queryMethod;
    
    public PartTreeJpaQuery(PartTree partTree, EntityManager entityManager, 
                           Class<?> domainClass) {
        this.partTree = partTree;
        this.entityManager = entityManager;
        this.domainClass = domainClass;
    }
}
```

## Анализ PartTree

### Структура PartTree
```java
public class PartTree {
    
    private final Subject subject;
    private final Predicate predicate;
    
    public PartTree(String methodName, Class<?> domainClass) {
        this.subject = new Subject(methodName);
        this.predicate = new Predicate(methodName, domainClass);
    }
}
```

### Subject (подлежащее)
```java
public class Subject {
    
    private final String subject;
    
    public Subject(String methodName) {
        // Извлечение подлежащего из имени метода
        // find, get, read, query, count, exists
        this.subject = extractSubject(methodName);
    }
    
    private String extractSubject(String methodName) {
        if (methodName.startsWith("find")) return "find";
        if (methodName.startsWith("get")) return "get";
        if (methodName.startsWith("read")) return "read";
        if (methodName.startsWith("query")) return "query";
        if (methodName.startsWith("count")) return "count";
        if (methodName.startsWith("exists")) return "exists";
        return "find"; // по умолчанию
    }
}
```

### Predicate (предикат)
```java
public class Predicate {
    
    private final List<Part> parts;
    
    public Predicate(String methodName, Class<?> domainClass) {
        this.parts = parseParts(methodName, domainClass);
    }
    
    private List<Part> parseParts(String methodName, Class<?> domainClass) {
        // Парсинг частей предиката
        // findByEmailAndName -> [Part(email), Part(name)]
        return Arrays.stream(methodName.split("And|Or"))
                    .map(part -> new Part(part, domainClass))
                    .collect(Collectors.toList());
    }
}
```

## Генерация JPQL

### Базовый шаблон
```java
public class JpqlQueryGenerator {
    
    public String generateQuery(PartTree partTree, Class<?> domainClass) {
        StringBuilder jpql = new StringBuilder();
        
        // SELECT clause
        jpql.append("SELECT ");
        if (partTree.getSubject().isCount()) {
            jpql.append("COUNT(");
        }
        jpql.append("e");
        if (partTree.getSubject().isCount()) {
            jpql.append(")");
        }
        
        // FROM clause
        jpql.append(" FROM ").append(domainClass.getSimpleName()).append(" e");
        
        // WHERE clause
        if (!partTree.getPredicate().getParts().isEmpty()) {
            jpql.append(" WHERE ");
            jpql.append(generateWhereClause(partTree.getPredicate()));
        }
        
        return jpql.toString();
    }
}
```

### Генерация WHERE clause
```java
private String generateWhereClause(Predicate predicate) {
    return predicate.getParts().stream()
                   .map(this::generatePartCondition)
                   .collect(Collectors.joining(" AND "));
}

private String generatePartCondition(Part part) {
    StringBuilder condition = new StringBuilder();
    
    condition.append("e.").append(part.getProperty());
    
    switch (part.getOperator()) {
        case EQUALS:
            condition.append(" = ?");
            break;
        case LIKE:
            condition.append(" LIKE ?");
            break;
        case GREATER_THAN:
            condition.append(" > ?");
            break;
        case LESS_THAN:
            condition.append(" < ?");
            break;
        case IS_NULL:
            condition.append(" IS NULL");
            break;
        case IS_NOT_NULL:
            condition.append(" IS NOT NULL");
            break;
    }
    
    return condition.toString();
}
```

## Поддерживаемые операторы

### Операторы сравнения
```java
public enum Operator {
    EQUALS("Equals"),
    NOT_EQUALS("NotEquals"),
    LIKE("Like"),
    NOT_LIKE("NotLike"),
    GREATER_THAN("GreaterThan"),
    GREATER_THAN_EQUALS("GreaterThanEquals"),
    LESS_THAN("LessThan"),
    LESS_THAN_EQUALS("LessThanEquals"),
    IS_NULL("IsNull"),
    IS_NOT_NULL("IsNotNull"),
    IN("In"),
    NOT_IN("NotIn"),
    BETWEEN("Between"),
    REGEX("Regex");
}
```

### Операторы логические
```java
public enum LogicalOperator {
    AND("And"),
    OR("Or"),
    NOT("Not");
}
```

## Обработка параметров

### Установка параметров
```java
public class ParameterBinder {
    
    public void bindParameters(Query query, PartTree partTree, Object[] arguments) {
        List<Part> parts = partTree.getPredicate().getParts();
        
        for (int i = 0; i < parts.size(); i++) {
            Part part = parts.get(i);
            Object value = arguments[i];
            
            if (part.getOperator() == Operator.LIKE && value instanceof String) {
                value = "%" + value + "%";
            }
            
            query.setParameter(i + 1, value);
        }
    }
}
```

### Валидация параметров
```java
public class ParameterValidator {
    
    public void validateParameters(PartTree partTree, Object[] arguments) {
        List<Part> parts = partTree.getPredicate().getParts();
        
        if (arguments.length != parts.size()) {
            throw new IllegalArgumentException(
                "Expected " + parts.size() + " parameters, but got " + arguments.length);
        }
        
        for (int i = 0; i < parts.size(); i++) {
            Part part = parts.get(i);
            Object value = arguments[i];
            
            validateParameter(part, value);
        }
    }
    
    private void validateParameter(Part part, Object value) {
        if (part.getOperator() == Operator.IS_NULL || 
            part.getOperator() == Operator.IS_NOT_NULL) {
            if (value != null) {
                throw new IllegalArgumentException(
                    "Parameter should be null for " + part.getOperator());
            }
        }
    }
}
```

## Кэширование запросов

### Кэш сгенерированных запросов
```java
public class PartTreeQueryCache {
    
    private static final Map<String, String> queryCache = new ConcurrentHashMap<>();
    
    public static String getOrCreateQuery(String methodName, Class<?> domainClass) {
        String cacheKey = methodName + "_" + domainClass.getName();
        
        return queryCache.computeIfAbsent(cacheKey, key -> {
            PartTree partTree = new PartTree(methodName, domainClass);
            return new JpqlQueryGenerator().generateQuery(partTree, domainClass);
        });
    }
}
```

## Производительность

### Оптимизация парсинга
```java
public class OptimizedPartTreeParser {
    
    private static final Pattern METHOD_PATTERN = 
        Pattern.compile("^(find|get|read|query|count|exists)(By|And|Or|OrderBy)(.+)$");
    
    public PartTree parseOptimized(String methodName, Class<?> domainClass) {
        Matcher matcher = METHOD_PATTERN.matcher(methodName);
        
        if (matcher.matches()) {
            String subject = matcher.group(1);
            String predicate = matcher.group(3);
            
            return new PartTree(subject, predicate, domainClass);
        }
        
        throw new IllegalArgumentException("Invalid method name: " + methodName);
    }
}
```

## Логирование и отладка

### Включение логирования
```properties
logging.level.org.springframework.data.repository.query=DEBUG
logging.level.org.springframework.data.repository.query.PartTree=DEBUG
```

### Отладочная информация
```java
public class PartTreeDebugger {
    
    public void debugPartTree(PartTree partTree) {
        System.out.println("Subject: " + partTree.getSubject());
        System.out.println("Predicate parts:");
        
        partTree.getPredicate().getParts().forEach(part -> {
            System.out.println("  - Property: " + part.getProperty());
            System.out.println("    Operator: " + part.getOperator());
            System.out.println("    Type: " + part.getPropertyType());
        });
    }
}
```

## Лучшие практики

1. **Используйте кэширование** для сгенерированных запросов
2. **Следуйте соглашениям** именования методов
3. **Валидируйте параметры** перед выполнением
4. **Мониторьте производительность** парсинга
5. **Используйте логирование** для отладки сложных запросов
6. **Тестируйте граничные случаи** в unit-тестах 