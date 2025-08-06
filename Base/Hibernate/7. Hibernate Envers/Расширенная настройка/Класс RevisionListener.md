# Класс RevisionListener

Интерфейс `RevisionListener` является ключевым компонентом для кастомизации процесса создания ревизий в Hibernate Envers. Он позволяет добавлять дополнительную логику при создании новых ревизий.

## Обзор интерфейса

```java
public interface RevisionListener {
    void newRevision(Object revisionEntity);
}
```

## Основные методы

### newRevision

Основной метод интерфейса, который вызывается при создании новой ревизии.

```java
@Override
public void newRevision(Object revisionEntity) {
    // Логика обработки новой ревизии
}
```

## Типы RevisionListener

### 1. Базовый RevisionListener

```java
@Component
public class BasicRevisionListener implements RevisionListener {
    
    private static final Logger log = LoggerFactory.getLogger(BasicRevisionListener.class);
    
    @Override
    public void newRevision(Object revisionEntity) {
        if (revisionEntity instanceof CustomRevisionEntity) {
            CustomRevisionEntity revision = (CustomRevisionEntity) revisionEntity;
            
            // Установка временной метки
            revision.setTimestamp(System.currentTimeMillis());
            
            // Установка информации о пользователе
            setUserInfo(revision);
            
            // Логирование
            log.info("Создана новая ревизия: {}", revision.getRev());
        }
    }
    
    private void setUserInfo(CustomRevisionEntity revision) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            revision.setUsername(auth.getName());
        } else {
            revision.setUsername("system");
        }
    }
}
```

### 2. Расширенный RevisionListener

```java
@Component
public class ExtendedRevisionListener implements RevisionListener {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private AuditMetadataService metadataService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Override
    public void newRevision(Object revisionEntity) {
        if (revisionEntity instanceof ExtendedRevisionEntity) {
            ExtendedRevisionEntity revision = (ExtendedRevisionEntity) revisionEntity;
            
            // Установка базовой информации
            setBasicInfo(revision);
            
            // Установка информации о пользователе
            setUserInfo(revision);
            
            // Установка информации о сессии
            setSessionInfo(revision);
            
            // Установка бизнес-информации
            setBusinessInfo(revision);
            
            // Уведомления
            sendNotifications(revision);
        }
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
            try {
                User user = userService.findByUsername(auth.getName());
                if (user != null) {
                    revision.setUserId(user.getId());
                    revision.setUserRole(user.getRole().name());
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
        // Получение информации из контекста
        revision.setComment(AuditContext.getCurrentComment());
        revision.setOperationType(AuditContext.getCurrentOperationType());
        revision.setModule(AuditContext.getCurrentModule());
        
        // Установка приоритета
        revision.setPriority(determinePriority(revision.getOperationType()));
        
        // Дополнительные метаданные
        Map<String, Object> metadata = metadataService.getCurrentMetadata();
        revision.setAdditionalData(JsonUtils.toJson(metadata));
    }
    
    private void sendNotifications(ExtendedRevisionEntity revision) {
        // Уведомление о критических изменениях
        if (isCriticalChange(revision)) {
            notificationService.notifyCriticalChange(revision);
        }
        
        // Уведомление администраторов
        if (isAdminChange(revision)) {
            notificationService.notifyAdmins(revision);
        }
    }
    
    private boolean isCriticalChange(ExtendedRevisionEntity revision) {
        return revision.getOperationType() == OperationType.CRITICAL ||
               revision.getPriority() == 1;
    }
    
    private boolean isAdminChange(ExtendedRevisionEntity revision) {
        return "ADMIN".equals(revision.getUserRole()) ||
               "SYSTEM".equals(revision.getUserRole());
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
    
    private String getCurrentEnvironment() {
        return System.getProperty("spring.profiles.active", "default");
    }
    
    private String getApplicationVersion() {
        return "1.0.0";
    }
}
```

## Специализированные RevisionListener

### 1. SecurityRevisionListener

```java
@Component
public class SecurityRevisionListener implements RevisionListener {
    
    @Autowired
    private SecurityAuditService securityAuditService;
    
    @Override
    public void newRevision(Object revisionEntity) {
        if (revisionEntity instanceof SecurityRevisionEntity) {
            SecurityRevisionEntity revision = (SecurityRevisionEntity) revisionEntity;
            
            // Базовая информация
            revision.setTimestamp(System.currentTimeMillis());
            
            // Информация о пользователе
            setUserSecurityInfo(revision);
            
            // Информация о сессии
            setSessionSecurityInfo(revision);
            
            // Анализ безопасности
            analyzeSecurity(revision);
            
            // Логирование безопасности
            securityAuditService.logSecurityEvent(revision);
        }
    }
    
    private void setUserSecurityInfo(SecurityRevisionEntity revision) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            revision.setUsername(auth.getName());
            revision.setUserRoles(getUserRoles(auth));
            revision.setPermissions(getUserPermissions(auth));
        } else {
            revision.setUsername("anonymous");
            revision.setUserRoles("ANONYMOUS");
            revision.setPermissions("NONE");
        }
    }
    
    private void setSessionSecurityInfo(SecurityRevisionEntity revision) {
        try {
            HttpServletRequest request = ((ServletRequestAttributes) 
                RequestContextHolder.currentRequestAttributes()).getRequest();
            
            revision.setIpAddress(getClientIpAddress(request));
            revision.setUserAgent(request.getHeader("User-Agent"));
            revision.setSessionId(request.getSession().getId());
            revision.setRequestUri(request.getRequestURI());
            revision.setRequestMethod(request.getMethod());
            
        } catch (Exception e) {
            revision.setIpAddress("unknown");
            revision.setUserAgent("unknown");
            revision.setSessionId("unknown");
        }
    }
    
    private void analyzeSecurity(SecurityRevisionEntity revision) {
        // Анализ подозрительной активности
        if (isSuspiciousActivity(revision)) {
            revision.setSecurityLevel(SecurityLevel.HIGH);
            revision.setRiskScore(calculateRiskScore(revision));
        } else {
            revision.setSecurityLevel(SecurityLevel.NORMAL);
            revision.setRiskScore(0);
        }
    }
    
    private boolean isSuspiciousActivity(SecurityRevisionEntity revision) {
        // Проверка подозрительной активности
        return "unknown".equals(revision.getIpAddress()) ||
               revision.getIpAddress().startsWith("192.168.") ||
               "anonymous".equals(revision.getUsername());
    }
    
    private int calculateRiskScore(SecurityRevisionEntity revision) {
        int score = 0;
        
        if ("anonymous".equals(revision.getUsername())) score += 50;
        if ("unknown".equals(revision.getIpAddress())) score += 30;
        if (revision.getIpAddress().startsWith("192.168.")) score += 20;
        
        return Math.min(score, 100);
    }
    
    private String getUserRoles(Authentication auth) {
        return auth.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.joining(","));
    }
    
    private String getUserPermissions(Authentication auth) {
        // Получение разрешений пользователя
        return "READ,WRITE"; // Упрощенный пример
    }
}
```

### 2. BusinessRevisionListener

```java
@Component
public class BusinessRevisionListener implements RevisionListener {
    
    @Autowired
    private BusinessRulesService businessRulesService;
    
    @Autowired
    private ApprovalService approvalService;
    
    @Override
    public void newRevision(Object revisionEntity) {
        if (revisionEntity instanceof BusinessRevisionEntity) {
            BusinessRevisionEntity revision = (BusinessRevisionEntity) revisionEntity;
            
            // Базовая информация
            revision.setTimestamp(System.currentTimeMillis());
            
            // Бизнес-информация
            setBusinessInfo(revision);
            
            // Применение бизнес-правил
            applyBusinessRules(revision);
            
            // Проверка необходимости одобрения
            checkApprovalRequirements(revision);
        }
    }
    
    private void setBusinessInfo(BusinessRevisionEntity revision) {
        // Определение отдела
        revision.setDepartment(determineDepartment());
        
        // Определение проекта
        revision.setProject(determineProject());
        
        // Определение типа операции
        revision.setOperationType(determineOperationType());
        
        // Определение уровня риска
        revision.setRiskLevel(determineRiskLevel(revision));
    }
    
    private void applyBusinessRules(BusinessRevisionEntity revision) {
        // Применение бизнес-правил
        BusinessRuleResult result = businessRulesService.applyRules(revision);
        
        revision.setComplianceStatus(result.getComplianceStatus());
        revision.setViolations(result.getViolations());
        revision.setRecommendations(result.getRecommendations());
    }
    
    private void checkApprovalRequirements(BusinessRevisionEntity revision) {
        if (requiresApproval(revision)) {
            revision.setApprovalStatus(ApprovalStatus.PENDING);
            revision.setApprovalRequired(true);
            
            // Создание запроса на одобрение
            approvalService.createApprovalRequest(revision);
        } else {
            revision.setApprovalStatus(ApprovalStatus.AUTO_APPROVED);
            revision.setApprovalRequired(false);
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
    
    private OperationType determineOperationType() {
        // Логика определения типа операции
        return OperationType.MANUAL;
    }
    
    private RiskLevel determineRiskLevel(BusinessRevisionEntity revision) {
        // Логика определения уровня риска
        if (revision.getOperationType() == OperationType.CRITICAL) {
            return RiskLevel.HIGH;
        } else if (revision.getOperationType() == OperationType.HIGH) {
            return RiskLevel.MEDIUM;
        } else {
            return RiskLevel.LOW;
        }
    }
    
    private boolean requiresApproval(BusinessRevisionEntity revision) {
        // Логика определения необходимости одобрения
        return revision.getRiskLevel() == RiskLevel.HIGH ||
               revision.getOperationType() == OperationType.CRITICAL ||
               revision.getComplianceStatus() == ComplianceStatus.VIOLATION;
    }
}
```

### 3. PerformanceRevisionListener

```java
@Component
public class PerformanceRevisionListener implements RevisionListener {
    
    private final MeterRegistry meterRegistry;
    private final Timer revisionTimer;
    private final Counter revisionCounter;
    
    public PerformanceRevisionListener(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.revisionTimer = Timer.builder("envers.revision.creation")
                                 .description("Время создания ревизии")
                                 .register(meterRegistry);
        this.revisionCounter = Counter.builder("envers.revisions.total")
                                     .description("Общее количество ревизий")
                                     .register(meterRegistry);
    }
    
    @Override
    public void newRevision(Object revisionEntity) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            if (revisionEntity instanceof PerformanceRevisionEntity) {
                PerformanceRevisionEntity revision = (PerformanceRevisionEntity) revisionEntity;
                
                // Базовая информация
                revision.setTimestamp(System.currentTimeMillis());
                
                // Метрики производительности
                setPerformanceMetrics(revision);
                
                // Обновление счетчиков
                updateCounters(revision);
                
            }
        } finally {
            sample.stop(revisionTimer);
        }
    }
    
    private void setPerformanceMetrics(PerformanceRevisionEntity revision) {
        // Время выполнения операции
        revision.setExecutionTime(System.currentTimeMillis() - revision.getTimestamp());
        
        // Использование памяти
        Runtime runtime = Runtime.getRuntime();
        revision.setMemoryUsage(runtime.totalMemory() - runtime.freeMemory());
        
        // Количество потоков
        revision.setThreadCount(Thread.activeCount());
        
        // Информация о системе
        revision.setSystemLoad(getSystemLoad());
        revision.setCpuUsage(getCpuUsage());
    }
    
    private void updateCounters(PerformanceRevisionEntity revision) {
        revisionCounter.increment();
        
        // Счетчики по типам операций
        Counter.builder("envers.revisions.by.type")
               .tag("type", revision.getOperationType().name())
               .register(meterRegistry)
               .increment();
        
        // Счетчики по пользователям
        Counter.builder("envers.revisions.by.user")
               .tag("user", revision.getUsername())
               .register(meterRegistry)
               .increment();
    }
    
    private double getSystemLoad() {
        // Получение нагрузки системы
        return ManagementFactory.getOperatingSystemMXBean().getSystemLoadAverage();
    }
    
    private double getCpuUsage() {
        // Получение использования CPU
        OperatingSystemMXBean osBean = ManagementFactory.getOperatingSystemMXBean();
        if (osBean instanceof com.sun.management.OperatingSystemMXBean) {
            return ((com.sun.management.OperatingSystemMXBean) osBean).getCpuLoad();
        }
        return 0.0;
    }
}
```

## Интеграция с Spring

### Конфигурация RevisionListener

```java
@Configuration
public class RevisionListenerConfig {
    
    @Bean
    public RevisionListener customRevisionListener() {
        return new CustomRevisionListener();
    }
    
    @Bean
    public RevisionListener securityRevisionListener() {
        return new SecurityRevisionListener();
    }
    
    @Bean
    public RevisionListener businessRevisionListener() {
        return new BusinessRevisionListener();
    }
    
    @Bean
    public RevisionListener performanceRevisionListener(MeterRegistry meterRegistry) {
        return new PerformanceRevisionListener(meterRegistry);
    }
}
```

### Использование нескольких RevisionListener

```java
@Component
public class CompositeRevisionListener implements RevisionListener {
    
    private final List<RevisionListener> listeners;
    
    public CompositeRevisionListener(List<RevisionListener> listeners) {
        this.listeners = listeners;
    }
    
    @Override
    public void newRevision(Object revisionEntity) {
        for (RevisionListener listener : listeners) {
            try {
                listener.newRevision(revisionEntity);
            } catch (Exception e) {
                log.error("Ошибка в RevisionListener: {}", listener.getClass().getSimpleName(), e);
                // Не прерываем выполнение других слушателей
            }
        }
    }
}
```

## Обработка ошибок

### Безопасный RevisionListener

```java
@Component
public class SafeRevisionListener implements RevisionListener {
    
    private static final Logger log = LoggerFactory.getLogger(SafeRevisionListener.class);
    
    @Override
    public void newRevision(Object revisionEntity) {
        try {
            processRevision(revisionEntity);
        } catch (Exception e) {
            log.error("Ошибка при обработке ревизии", e);
            // Не прерываем процесс создания ревизии
            handleError(revisionEntity, e);
        }
    }
    
    private void processRevision(Object revisionEntity) {
        if (revisionEntity instanceof CustomRevisionEntity) {
            CustomRevisionEntity revision = (CustomRevisionEntity) revisionEntity;
            
            // Установка базовой информации
            revision.setTimestamp(System.currentTimeMillis());
            
            // Установка информации о пользователе
            setUserInfo(revision);
            
            // Дополнительная обработка
            performAdditionalProcessing(revision);
        }
    }
    
    private void setUserInfo(CustomRevisionEntity revision) {
        try {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth != null && auth.isAuthenticated()) {
                revision.setUsername(auth.getName());
            } else {
                revision.setUsername("system");
            }
        } catch (Exception e) {
            log.warn("Не удалось получить информацию о пользователе", e);
            revision.setUsername("unknown");
        }
    }
    
    private void performAdditionalProcessing(CustomRevisionEntity revision) {
        // Дополнительная обработка
    }
    
    private void handleError(Object revisionEntity, Exception e) {
        // Обработка ошибки
        if (revisionEntity instanceof CustomRevisionEntity) {
            CustomRevisionEntity revision = (CustomRevisionEntity) revisionEntity;
            revision.setUsername("error");
            revision.setComment("Ошибка при создании ревизии: " + e.getMessage());
        }
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class CustomRevisionListenerTest {
    
    @Mock
    private UserService userService;
    
    @InjectMocks
    private CustomRevisionListener listener;
    
    @Test
    void newRevision_ShouldSetUserInfo() {
        // Given
        CustomRevisionEntity revision = new CustomRevisionEntity();
        
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
        assertNotNull(revision.getTimestamp());
    }
    
    @Test
    void newRevision_ShouldHandleNullAuthentication() {
        // Given
        CustomRevisionEntity revision = new CustomRevisionEntity();
        SecurityContextHolder.clearContext();
        
        // When
        listener.newRevision(revision);
        
        // Then
        assertEquals("system", revision.getUsername());
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
class RevisionListenerIntegrationTest {
    
    @Autowired
    private EntityManager entityManager;
    
    @Test
    void shouldTriggerRevisionListenerOnEntitySave() {
        // Given
        User user = new User("test", "test@example.com");
        
        // When
        entityManager.persist(user);
        entityManager.flush();
        
        // Then
        // Проверяем, что RevisionListener сработал
        List<CustomRevisionEntity> revisions = entityManager
            .createQuery("SELECT r FROM CustomRevisionEntity r", CustomRevisionEntity.class)
            .getResultList();
        
        assertFalse(revisions.isEmpty());
        assertNotNull(revisions.get(0).getUsername());
    }
}
```

## Заключение

Класс `RevisionListener` предоставляет мощные возможности для кастомизации процесса создания ревизий в Hibernate Envers:

- **Гибкость**: Возможность добавления любой логики при создании ревизий
- **Интеграция**: Легкая интеграция с бизнес-логикой и внешними системами
- **Безопасность**: Возможность добавления проверок безопасности
- **Мониторинг**: Сбор метрик и статистики
- **Отказоустойчивость**: Правильная обработка ошибок

При создании RevisionListener важно помнить о производительности, правильной обработке ошибок и соблюдении принципов безопасности. 