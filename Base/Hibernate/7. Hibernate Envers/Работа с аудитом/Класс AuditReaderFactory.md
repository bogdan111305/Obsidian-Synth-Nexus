# Класс AuditReaderFactory

`AuditReaderFactory` является фабрикой для создания экземпляров `AuditReader` - основного API для чтения аудита в Hibernate Envers. Этот класс предоставляет методы для получения настроенного экземпляра `AuditReader`.

## Обзор класса

```java
public class AuditReaderFactory {
    
    // Получение AuditReader для EntityManager
    public static AuditReader get(EntityManager entityManager);
    
    // Получение AuditReader для EntityManager с кастомной ревизией
    public static AuditReader get(EntityManager entityManager, Class<?> revisionEntityClass);
    
    // Получение AuditReader для Session
    public static AuditReader get(Session session);
    
    // Получение AuditReader для Session с кастомной ревизией
    public static AuditReader get(Session session, Class<?> revisionEntityClass);
}
```

## Основные методы

### get(EntityManager entityManager)

Создает `AuditReader` для работы с `EntityManager`.

```java
@Service
public class AuditService {
    
    @Autowired
    private EntityManager entityManager;
    
    public AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Number> getRevisions(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        return auditReader.getRevisions(entityName, id);
    }
    
    public Object findEntity(String entityName, Serializable id, Number revision) {
        AuditReader auditReader = getAuditReader();
        return auditReader.find(entityName, id, revision);
    }
}
```

### get(EntityManager entityManager, Class<?> revisionEntityClass)

Создает `AuditReader` с кастомной ревизией.

```java
@Service
public class CustomAuditService {
    
    @Autowired
    private EntityManager entityManager;
    
    public AuditReader getCustomAuditReader() {
        return AuditReaderFactory.get(entityManager, CustomRevisionEntity.class);
    }
    
    public CustomRevisionEntity getRevisionInfo(Number revision) {
        AuditReader auditReader = getCustomAuditReader();
        return auditReader.findRevision(CustomRevisionEntity.class, revision);
    }
    
    public List<CustomRevisionEntity> getRevisionsWithInfo(String entityName, Serializable id) {
        AuditReader auditReader = getCustomAuditReader();
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> auditReader.findRevision(CustomRevisionEntity.class, rev))
            .collect(Collectors.toList());
    }
}
```

### get(Session session)

Создает `AuditReader` для работы с Hibernate `Session`.

```java
@Service
public class SessionBasedAuditService {
    
    @Autowired
    private SessionFactory sessionFactory;
    
    public AuditReader getAuditReader() {
        Session session = sessionFactory.getCurrentSession();
        return AuditReaderFactory.get(session);
    }
    
    public List<Object> getAuditHistory(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> auditReader.find(entityName, id, rev))
            .collect(Collectors.toList());
    }
}
```

## Конфигурация AuditReader

### Базовая конфигурация

```java
@Configuration
public class AuditReaderConfig {
    
    @Bean
    public AuditReaderFactory auditReaderFactory() {
        return new AuditReaderFactory();
    }
    
    @Bean
    public AuditReader auditReader(EntityManager entityManager) {
        return AuditReaderFactory.get(entityManager);
    }
}
```

### Конфигурация с кастомной ревизией

```java
@Configuration
public class CustomAuditReaderConfig {
    
    @Bean
    public AuditReader customAuditReader(EntityManager entityManager) {
        return AuditReaderFactory.get(entityManager, ExtendedRevisionEntity.class);
    }
    
    @Bean
    public AuditReader securityAuditReader(EntityManager entityManager) {
        return AuditReaderFactory.get(entityManager, SecurityRevisionEntity.class);
    }
}
```

## Практические примеры использования

### 1. Базовый сервис аудита

```java
@Service
@Transactional
public class BasicAuditService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager);
    }
    
    public List<Object> getEntityHistory(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> auditReader.find(entityName, id, rev))
            .collect(Collectors.toList());
    }
    
    public Object getEntityAtRevision(String entityName, Serializable id, Number revision) {
        AuditReader auditReader = getAuditReader();
        return auditReader.find(entityName, id, revision);
    }
    
    public List<Number> getRevisionsForEntity(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        return auditReader.getRevisions(entityName, id);
    }
}
```

### 2. Расширенный сервис аудита

```java
@Service
@Transactional
public class ExtendedAuditService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager, ExtendedRevisionEntity.class);
    }
    
    public List<ExtendedRevisionEntity> getRevisionsWithMetadata(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> auditReader.findRevision(ExtendedRevisionEntity.class, rev))
            .collect(Collectors.toList());
    }
    
    public List<ExtendedRevisionEntity> getRevisionsByUser(String username) {
        AuditReader auditReader = getAuditReader();
        
        // Кастомный запрос для получения ревизий по пользователю
        String query = "SELECT r FROM ExtendedRevisionEntity r WHERE r.username = :username ORDER BY r.rev DESC";
        
        return entityManager.createQuery(query, ExtendedRevisionEntity.class)
            .setParameter("username", username)
            .getResultList();
    }
    
    public List<ExtendedRevisionEntity> getRevisionsByDateRange(Date from, Date to) {
        AuditReader auditReader = getAuditReader();
        
        String query = "SELECT r FROM ExtendedRevisionEntity r " +
                      "WHERE r.timestamp BETWEEN :from AND :to " +
                      "ORDER BY r.rev DESC";
        
        return entityManager.createQuery(query, ExtendedRevisionEntity.class)
            .setParameter("from", from.getTime())
            .setParameter("to", to.getTime())
            .getResultList();
    }
}
```

### 3. Специализированный сервис аудита

```java
@Service
@Transactional
public class SecurityAuditService {
    
    @Autowired
    private EntityManager entityManager;
    
    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(entityManager, SecurityRevisionEntity.class);
    }
    
    public List<SecurityRevisionEntity> getSecurityAuditTrail(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader();
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> auditReader.findRevision(SecurityRevisionEntity.class, rev))
            .filter(revision -> revision.getSecurityLevel() == SecurityLevel.HIGH)
            .collect(Collectors.toList());
    }
    
    public List<SecurityRevisionEntity> getSuspiciousActivity() {
        AuditReader auditReader = getAuditReader();
        
        String query = "SELECT r FROM SecurityRevisionEntity r " +
                      "WHERE r.riskScore > :threshold " +
                      "ORDER BY r.riskScore DESC";
        
        return entityManager.createQuery(query, SecurityRevisionEntity.class)
            .setParameter("threshold", 50)
            .getResultList();
    }
}
```

## Интеграция с Spring

### Конфигурация через Spring Boot

```java
@Configuration
public class AuditReaderConfiguration {
    
    @Bean
    @Primary
    public AuditReader defaultAuditReader(EntityManager entityManager) {
        return AuditReaderFactory.get(entityManager);
    }
    
    @Bean
    @Qualifier("customAuditReader")
    public AuditReader customAuditReader(EntityManager entityManager) {
        return AuditReaderFactory.get(entityManager, CustomRevisionEntity.class);
    }
    
    @Bean
    @Qualifier("securityAuditReader")
    public AuditReader securityAuditReader(EntityManager entityManager) {
        return AuditReaderFactory.get(entityManager, SecurityRevisionEntity.class);
    }
}
```

### Использование в сервисах

```java
@Service
public class MultiAuditService {
    
    @Autowired
    @Qualifier("defaultAuditReader")
    private AuditReader defaultAuditReader;
    
    @Autowired
    @Qualifier("customAuditReader")
    private AuditReader customAuditReader;
    
    @Autowired
    @Qualifier("securityAuditReader")
    private AuditReader securityAuditReader;
    
    public Object getEntityHistory(String entityName, Serializable id) {
        List<Number> revisions = defaultAuditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> defaultAuditReader.find(entityName, id, rev))
            .collect(Collectors.toList());
    }
    
    public List<CustomRevisionEntity> getCustomRevisions(String entityName, Serializable id) {
        List<Number> revisions = customAuditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> customAuditReader.findRevision(CustomRevisionEntity.class, rev))
            .collect(Collectors.toList());
    }
    
    public List<SecurityRevisionEntity> getSecurityRevisions(String entityName, Serializable id) {
        List<Number> revisions = securityAuditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> securityAuditReader.findRevision(SecurityRevisionEntity.class, rev))
            .collect(Collectors.toList());
    }
}
```

## Обработка ошибок

### Безопасное создание AuditReader

```java
@Service
public class SafeAuditService {
    
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
    
    public Optional<List<Object>> getEntityHistory(String entityName, Serializable id) {
        try {
            AuditReader auditReader = getAuditReader();
            List<Number> revisions = auditReader.getRevisions(entityName, id);
            
            List<Object> history = revisions.stream()
                .map(rev -> auditReader.find(entityName, id, rev))
                .collect(Collectors.toList());
            
            return Optional.of(history);
        } catch (Exception e) {
            log.error("Ошибка при получении истории сущности", e);
            return Optional.empty();
        }
    }
}
```

## Производительность

### Кэширование AuditReader

```java
@Service
public class CachedAuditService {
    
    @Autowired
    private EntityManager entityManager;
    
    private final Map<String, AuditReader> auditReaderCache = new ConcurrentHashMap<>();
    
    private AuditReader getAuditReader(String cacheKey) {
        return auditReaderCache.computeIfAbsent(cacheKey, key -> {
            if ("custom".equals(key)) {
                return AuditReaderFactory.get(entityManager, CustomRevisionEntity.class);
            } else if ("security".equals(key)) {
                return AuditReaderFactory.get(entityManager, SecurityRevisionEntity.class);
            } else {
                return AuditReaderFactory.get(entityManager);
            }
        });
    }
    
    public List<Object> getEntityHistory(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader("default");
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> auditReader.find(entityName, id, rev))
            .collect(Collectors.toList());
    }
    
    public List<CustomRevisionEntity> getCustomRevisions(String entityName, Serializable id) {
        AuditReader auditReader = getAuditReader("custom");
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        
        return revisions.stream()
            .map(rev -> auditReader.findRevision(CustomRevisionEntity.class, rev))
            .collect(Collectors.toList());
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class AuditReaderFactoryTest {
    
    @Mock
    private EntityManager entityManager;
    
    @Test
    void get_ShouldCreateAuditReader() {
        // When
        AuditReader auditReader = AuditReaderFactory.get(entityManager);
        
        // Then
        assertNotNull(auditReader);
    }
    
    @Test
    void get_WithCustomRevision_ShouldCreateCustomAuditReader() {
        // When
        AuditReader auditReader = AuditReaderFactory.get(entityManager, CustomRevisionEntity.class);
        
        // Then
        assertNotNull(auditReader);
    }
}
```

### Integration тесты

```java
@SpringBootTest
class AuditReaderFactoryIntegrationTest {
    
    @Autowired
    private EntityManager entityManager;
    
    @Test
    void shouldCreateAuditReader() {
        // When
        AuditReader auditReader = AuditReaderFactory.get(entityManager);
        
        // Then
        assertNotNull(auditReader);
    }
    
    @Test
    void shouldCreateCustomAuditReader() {
        // When
        AuditReader auditReader = AuditReaderFactory.get(entityManager, CustomRevisionEntity.class);
        
        // Then
        assertNotNull(auditReader);
    }
}
```

## Заключение

`AuditReaderFactory` является ключевым компонентом для работы с аудитом в Hibernate Envers. Правильное использование этого класса позволяет:

- Создавать настроенные экземпляры `AuditReader`
- Работать с кастомными ревизиями
- Интегрировать аудит с Spring Framework
- Обеспечивать производительность через кэширование
- Обрабатывать ошибки безопасно

При работе с `AuditReaderFactory` важно помнить о производительности, правильной обработке ошибок и интеграции с существующей архитектурой приложения. 