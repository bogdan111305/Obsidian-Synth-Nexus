# Custom revinfo table

Кастомная таблица ревизий позволяет расширить стандартную таблицу `REVINFO` дополнительными полями для хранения метаинформации о ревизиях. Это мощный инструмент для интеграции аудита с бизнес-логикой.

## Обзор концепции

По умолчанию Hibernate Envers создает таблицу `REVINFO` с двумя полями:
- `REV` - номер ревизии
- `REVTSTMP` - временная метка

Кастомная таблица ревизий позволяет добавить дополнительные поля, такие как:
- Пользователь, внесший изменения
- IP адрес
- Комментарий к изменениям
- Тип операции
- Дополнительные метаданные

## Аннотация @RevisionEntity

### Базовая структура

```java
@Entity
@Table(name = "REVINFO")
@RevisionEntity(CustomRevisionListener.class)
public class CustomRevisionEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "REV")
    private Long rev;
    
    @Column(name = "REVTSTMP")
    private Long timestamp;
    
    @Column(name = "USERNAME")
    private String username;
    
    @Column(name = "IP_ADDRESS")
    private String ipAddress;
    
    @Column(name = "USER_AGENT")
    private String userAgent;
    
    @Column(name = "COMMENT")
    private String comment;
    
    // Конструкторы, геттеры, сеттеры...
}
```

### Расширенная структура

```java
@Entity
@Table(name = "REVINFO")
@RevisionEntity(CustomRevisionListener.class)
public class ExtendedRevisionEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "REV")
    private Long rev;
    
    @Column(name = "REVTSTMP")
    private Long timestamp;
    
    // Информация о пользователе
    @Column(name = "USERNAME")
    private String username;
    
    @Column(name = "USER_ID")
    private Long userId;
    
    @Column(name = "USER_ROLE")
    private String userRole;
    
    // Информация о сессии
    @Column(name = "IP_ADDRESS")
    private String ipAddress;
    
    @Column(name = "USER_AGENT")
    private String userAgent;
    
    @Column(name = "SESSION_ID")
    private String sessionId;
    
    // Бизнес-информация
    @Column(name = "COMMENT")
    private String comment;
    
    @Column(name = "OPERATION_TYPE")
    @Enumerated(EnumType.STRING)
    private OperationType operationType;
    
    @Column(name = "MODULE")
    private String module;
    
    @Column(name = "PRIORITY")
    private Integer priority;
    
    // Метаданные
    @Column(name = "ENVIRONMENT")
    private String environment;
    
    @Column(name = "VERSION")
    private String version;
    
    @Column(name = "ADDITIONAL_DATA", columnDefinition = "TEXT")
    private String additionalData;
    
    // Конструкторы
    public ExtendedRevisionEntity() {
        this.timestamp = System.currentTimeMillis();
        this.environment = getCurrentEnvironment();
        this.version = getApplicationVersion();
    }
    
    // Геттеры и сеттеры...
    
    private String getCurrentEnvironment() {
        return System.getProperty("spring.profiles.active", "default");
    }
    
    private String getApplicationVersion() {
        return "1.0.0"; // Можно получать из конфигурации
    }
}
```

## Класс RevisionListener

### Базовый RevisionListener

```java
@Component
public class CustomRevisionListener implements RevisionListener {
    
    @Override
    public void newRevision(Object revisionEntity) {
        CustomRevisionEntity revision = (CustomRevisionEntity) revisionEntity;
        
        // Установка информации о пользователе
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            revision.setUsername(auth.getName());
        } else {
            revision.setUsername("system");
        }
        
        // Установка IP адреса
        try {
            HttpServletRequest request = ((ServletRequestAttributes) 
                RequestContextHolder.currentRequestAttributes()).getRequest();
            revision.setIpAddress(getClientIpAddress(request));
            revision.setUserAgent(request.getHeader("User-Agent"));
        } catch (Exception e) {
            revision.setIpAddress("unknown");
            revision.setUserAgent("unknown");
        }
    }
    
    private String getClientIpAddress(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        
        String xRealIp = request.getHeader("X-Real-IP");
        if (xRealIp != null && !xRealIp.isEmpty()) {
            return xRealIp;
        }
        
        return request.getRemoteAddr();
    }
}
```

### Расширенный RevisionListener

```java
@Component
public class ExtendedRevisionListener implements RevisionListener {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private AuditMetadataService metadataService;
    
    @Override
    public void newRevision(Object revisionEntity) {
        ExtendedRevisionEntity revision = (ExtendedRevisionEntity) revisionEntity;
        
        // Базовая информация
        setBasicInfo(revision);
        
        // Информация о пользователе
        setUserInfo(revision);
        
        // Информация о сессии
        setSessionInfo(revision);
        
        // Бизнес-информация
        setBusinessInfo(revision);
        
        // Метаданные
        setMetadata(revision);
    }
    
    private void setBasicInfo(ExtendedRevisionEntity revision) {
        revision.setTimestamp(System.currentTimeMillis());
        revision.setEnvironment(getCurrentEnvironment());
        revision.setVersion(getApplicationVersion());
    }
    
    private void setUserInfo(ExtendedRevisionEntity revision) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            revision.setUsername(auth.getName());
            
            // Получение дополнительной информации о пользователе
            if (auth.getPrincipal() instanceof UserDetails) {
                UserDetails userDetails = (UserDetails) auth.getPrincipal();
                revision.setUserRole(userDetails.getAuthorities().stream()
                    .map(GrantedAuthority::getAuthority)
                    .collect(Collectors.joining(",")));
            }
            
            // Получение ID пользователя из базы данных
            try {
                User user = userService.findByUsername(auth.getName());
                if (user != null) {
                    revision.setUserId(user.getId());
                }
            } catch (Exception e) {
                log.warn("Не удалось получить информацию о пользователе", e);
            }
        } else {
            revision.setUsername("system");
            revision.setUserRole("SYSTEM");
        }
    }
    
    private void setSessionInfo(ExtendedRevisionEntity revision) {
        try {
            HttpServletRequest request = ((ServletRequestAttributes) 
                RequestContextHolder.currentRequestAttributes()).getRequest();
            
            revision.setIpAddress(getClientIpAddress(request));
            revision.setUserAgent(request.getHeader("User-Agent"));
            revision.setSessionId(request.getSession().getId());
            
        } catch (Exception e) {
            revision.setIpAddress("unknown");
            revision.setUserAgent("unknown");
            revision.setSessionId("unknown");
        }
    }
    
    private void setBusinessInfo(ExtendedRevisionEntity revision) {
        // Определение типа операции
        revision.setOperationType(determineOperationType());
        
        // Определение модуля
        revision.setModule(determineModule());
        
        // Установка приоритета
        revision.setPriority(determinePriority(revision.getOperationType()));
        
        // Комментарий (можно получать из контекста)
        revision.setComment(getCurrentComment());
    }
    
    private void setMetadata(ExtendedRevisionEntity revision) {
        // Дополнительные метаданные
        Map<String, Object> metadata = metadataService.getCurrentMetadata();
        revision.setAdditionalData(JsonUtils.toJson(metadata));
    }
    
    private OperationType determineOperationType() {
        // Логика определения типа операции
        // Можно анализировать текущий контекст или использовать ThreadLocal
        return OperationType.MANUAL;
    }
    
    private String determineModule() {
        // Логика определения модуля
        return "unknown";
    }
    
    private Integer determinePriority(OperationType operationType) {
        switch (operationType) {
            case CRITICAL: return 1;
            case HIGH: return 2;
            case MEDIUM: return 3;
            case LOW: return 4;
            default: return 3;
        }
    }
    
    private String getCurrentComment() {
        // Получение комментария из контекста
        return AuditContext.getCurrentComment();
    }
    
    private String getClientIpAddress(HttpServletRequest request) {
        // Логика получения IP адреса...
        return request.getRemoteAddr();
    }
    
    private String getCurrentEnvironment() {
        return System.getProperty("spring.profiles.active", "default");
    }
    
    private String getApplicationVersion() {
        return "1.0.0";
    }
}
```

## Контекст аудита

### AuditContext для передачи метаданных

```java
public class AuditContext {
    
    private static final ThreadLocal<String> commentHolder = new ThreadLocal<>();
    private static final ThreadLocal<OperationType> operationTypeHolder = new ThreadLocal<>();
    private static final ThreadLocal<String> moduleHolder = new ThreadLocal<>();
    private static final ThreadLocal<Map<String, Object>> metadataHolder = new ThreadLocal<>();
    
    public static void setComment(String comment) {
        commentHolder.set(comment);
    }
    
    public static String getCurrentComment() {
        String comment = commentHolder.get();
        commentHolder.remove();
        return comment;
    }
    
    public static void setOperationType(OperationType operationType) {
        operationTypeHolder.set(operationType);
    }
    
    public static OperationType getCurrentOperationType() {
        OperationType type = operationTypeHolder.get();
        operationTypeHolder.remove();
        return type;
    }
    
    public static void setModule(String module) {
        moduleHolder.set(module);
    }
    
    public static String getCurrentModule() {
        String module = moduleHolder.get();
        moduleHolder.remove();
        return module;
    }
    
    public static void setMetadata(String key, Object value) {
        Map<String, Object> metadata = metadataHolder.get();
        if (metadata == null) {
            metadata = new HashMap<>();
            metadataHolder.set(metadata);
        }
        metadata.put(key, value);
    }
    
    public static Map<String, Object> getCurrentMetadata() {
        Map<String, Object> metadata = metadataHolder.get();
        metadataHolder.remove();
        return metadata != null ? metadata : new HashMap<>();
    }
    
    public static void clear() {
        commentHolder.remove();
        operationTypeHolder.remove();
        moduleHolder.remove();
        metadataHolder.remove();
    }
}
```

## Использование в коде

### Пример использования AuditContext

```java
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(User user) {
        // Установка контекста аудита
        AuditContext.setComment("Создание нового пользователя");
        AuditContext.setOperationType(OperationType.CREATE);
        AuditContext.setModule("USER_MANAGEMENT");
        AuditContext.setMetadata("source", "admin_panel");
        
        try {
            return userRepository.save(user);
        } finally {
            AuditContext.clear();
        }
    }
    
    public User updateUser(Long id, User user) {
        // Установка контекста аудита
        AuditContext.setComment("Обновление данных пользователя");
        AuditContext.setOperationType(OperationType.UPDATE);
        AuditContext.setModule("USER_MANAGEMENT");
        AuditContext.setMetadata("user_id", id);
        
        try {
            User existingUser = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));
            
            // Обновление полей
            existingUser.setName(user.getName());
            existingUser.setEmail(user.getEmail());
            
            return userRepository.save(existingUser);
        } finally {
            AuditContext.clear();
        }
    }
    
    public void deleteUser(Long id) {
        // Установка контекста аудита
        AuditContext.setComment("Удаление пользователя");
        AuditContext.setOperationType(OperationType.DELETE);
        AuditContext.setModule("USER_MANAGEMENT");
        AuditContext.setMetadata("user_id", id);
        
        try {
            userRepository.deleteById(id);
        } finally {
            AuditContext.clear();
        }
    }
}
```

## Конфигурация

### Настройка в application.properties

```properties
# Включение кастомной таблицы ревизий
hibernate.envers.revision_entity=com.example.ExtendedRevisionEntity
hibernate.envers.revision_listener=com.example.ExtendedRevisionListener

# Дополнительные настройки
hibernate.envers.audit_table_suffix=_AUDIT
hibernate.envers.revision_field_name=REV
hibernate.envers.revision_type_field_name=REVTYPE
```

### Конфигурация через Java

```java
@Configuration
public class EnversConfig {
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        
        Properties properties = new Properties();
        properties.setProperty("hibernate.envers.revision_entity", 
                             "com.example.ExtendedRevisionEntity");
        properties.setProperty("hibernate.envers.revision_listener", 
                             "com.example.ExtendedRevisionListener");
        
        em.setJpaProperties(properties);
        return em;
    }
}
```

## Чтение кастомных ревизий

### Получение информации о ревизии

```java
@Service
public class AuditService {
    
    @Autowired
    private AuditReaderFactory auditReaderFactory;
    
    public ExtendedRevisionEntity getRevisionInfo(Long revision) {
        AuditReader auditReader = auditReaderFactory.get(entityManager, ExtendedRevisionEntity.class);
        return auditReader.findRevision(ExtendedRevisionEntity.class, revision);
    }
    
    public List<ExtendedRevisionEntity> getRevisionsForEntity(String entityName, Serializable id) {
        AuditReader auditReader = auditReaderFactory.get(entityManager, ExtendedRevisionEntity.class);
        
        List<Number> revisions = auditReader.getRevisions(entityName, id);
        return revisions.stream()
            .map(rev -> auditReader.findRevision(ExtendedRevisionEntity.class, rev))
            .collect(Collectors.toList());
    }
    
    public List<ExtendedRevisionEntity> getRevisionsByUser(String username) {
        // Кастомный запрос для получения ревизий по пользователю
        String query = "SELECT r FROM ExtendedRevisionEntity r WHERE r.username = :username ORDER BY r.rev DESC";
        
        return entityManager.createQuery(query, ExtendedRevisionEntity.class)
            .setParameter("username", username)
            .getResultList();
    }
    
    public List<ExtendedRevisionEntity> getRevisionsByDateRange(Date from, Date to) {
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

## Практические примеры

### Аудит с бизнес-логикой

```java
@Entity
@Table(name = "REVINFO")
@RevisionEntity(BusinessRevisionListener.class)
public class BusinessRevisionEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "REV")
    private Long rev;
    
    @Column(name = "REVTSTMP")
    private Long timestamp;
    
    // Бизнес-поля
    @Column(name = "DEPARTMENT")
    private String department;
    
    @Column(name = "PROJECT")
    private String project;
    
    @Column(name = "APPROVAL_STATUS")
    @Enumerated(EnumType.STRING)
    private ApprovalStatus approvalStatus;
    
    @Column(name = "APPROVED_BY")
    private String approvedBy;
    
    @Column(name = "APPROVAL_DATE")
    private Date approvalDate;
    
    @Column(name = "RISK_LEVEL")
    @Enumerated(EnumType.STRING)
    private RiskLevel riskLevel;
    
    // Конструкторы, геттеры, сеттеры...
}

@Component
public class BusinessRevisionListener implements RevisionListener {
    
    @Override
    public void newRevision(Object revisionEntity) {
        BusinessRevisionEntity revision = (BusinessRevisionEntity) revisionEntity;
        
        // Установка базовой информации
        revision.setTimestamp(System.currentTimeMillis());
        
        // Определение отдела
        revision.setDepartment(determineDepartment());
        
        // Определение проекта
        revision.setProject(determineProject());
        
        // Определение уровня риска
        revision.setRiskLevel(determineRiskLevel());
        
        // Проверка необходимости одобрения
        if (requiresApproval(revision)) {
            revision.setApprovalStatus(ApprovalStatus.PENDING);
        } else {
            revision.setApprovalStatus(ApprovalStatus.AUTO_APPROVED);
            revision.setApprovedBy("system");
            revision.setApprovalDate(new Date());
        }
    }
    
    private String determineDepartment() {
        // Логика определения отдела
        return "IT";
    }
    
    private String determineProject() {
        // Логика определения проекта
        return "CRM_SYSTEM";
    }
    
    private RiskLevel determineRiskLevel() {
        // Логика определения уровня риска
        return RiskLevel.MEDIUM;
    }
    
    private boolean requiresApproval(BusinessRevisionEntity revision) {
        // Логика определения необходимости одобрения
        return revision.getRiskLevel() == RiskLevel.HIGH;
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class CustomRevisionListenerTest {
    
    @Test
    void newRevision_ShouldSetUserInfo() {
        // Given
        CustomRevisionEntity revision = new CustomRevisionEntity();
        CustomRevisionListener listener = new CustomRevisionListener();
        
        // Mock SecurityContext
        Authentication auth = mock(Authentication.class);
        when(auth.isAuthenticated()).thenReturn(true);
        when(auth.getName()).thenReturn("testuser");
        
        SecurityContext context = mock(SecurityContext.class);
        when(context.getAuthentication()).thenReturn(auth);
        
        SecurityContextHolder.setContext(context);
        
        // When
        listener.newRevision(revision);
        
        // Then
        assertEquals("testuser", revision.getUsername());
    }
}
```

### Integration тесты

```java
@SpringBootTest
@TestPropertySource(properties = {
    "hibernate.envers.revision_entity=com.example.CustomRevisionEntity",
    "hibernate.envers.revision_listener=com.example.CustomRevisionListener"
})
class CustomRevisionEntityIntegrationTest {
    
    @Autowired
    private EntityManager entityManager;
    
    @Test
    void shouldCreateCustomRevision() {
        // Given
        User user = new User("test", "test@example.com");
        
        // When
        entityManager.persist(user);
        entityManager.flush();
        
        // Then
        // Проверяем, что создалась кастомная ревизия
        List<CustomRevisionEntity> revisions = entityManager
            .createQuery("SELECT r FROM CustomRevisionEntity r", CustomRevisionEntity.class)
            .getResultList();
        
        assertFalse(revisions.isEmpty());
        assertNotNull(revisions.get(0).getUsername());
    }
}
```

## Заключение

Кастомная таблица ревизий предоставляет мощные возможности для расширения функциональности аудита в Hibernate Envers:

- **Гибкость**: Возможность добавления любых полей для метаинформации
- **Интеграция**: Легкая интеграция с бизнес-логикой приложения
- **Отслеживаемость**: Полная информация о том, кто, когда и зачем внес изменения
- **Соответствие**: Возможность соответствия требованиям регуляторов
- **Аналитика**: Богатые данные для анализа и отчетности

При создании кастомной таблицы ревизий важно помнить о производительности, правильной обработке контекста и соблюдении принципов безопасности. 