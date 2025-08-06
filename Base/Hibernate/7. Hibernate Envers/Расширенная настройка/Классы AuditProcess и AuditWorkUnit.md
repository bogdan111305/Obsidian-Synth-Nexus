# Классы AuditProcess и AuditWorkUnit

Классы `AuditProcess` и `AuditWorkUnit` являются ключевыми компонентами внутренней архитектуры Hibernate Envers. Они отвечают за управление процессом аудита и обработку единиц работы.

## Обзор архитектуры

```
Entity Change → AuditWorkUnit → AuditProcess → Database
```

## Класс AuditProcess

`AuditProcess` является основным координатором процесса аудита. Он управляет жизненным циклом аудита и координирует работу различных компонентов.

### Основные методы

```java
public class AuditProcess {
    
    // Обработка единицы работы
    public void addWorkUnit(AuditWorkUnit workUnit);
    
    // Выполнение аудита
    public void doBeforeTransactionCompletion();
    
    // Очистка после транзакции
    public void doAfterTransactionCompletion(boolean success);
    
    // Получение информации о ревизии
    public Object getCurrentRevisionData(boolean persist);
}
```

### Жизненный цикл AuditProcess

```java
@Component
public class CustomAuditProcess extends AuditProcess {
    
    private final List<AuditWorkUnit> workUnits = new ArrayList<>();
    private final AuditRevisionGenerator revisionGenerator;
    
    @Override
    public void addWorkUnit(AuditWorkUnit workUnit) {
        // Логирование добавления единицы работы
        log.debug("Добавлена единица работы: {}", workUnit.getClass().getSimpleName());
        
        // Валидация единицы работы
        validateWorkUnit(workUnit);
        
        workUnits.add(workUnit);
    }
    
    @Override
    public void doBeforeTransactionCompletion() {
        log.info("Начало завершения транзакции аудита");
        
        // Создание ревизии
        Object revision = revisionGenerator.generate();
        
        // Обработка всех единиц работы
        for (AuditWorkUnit workUnit : workUnits) {
            try {
                workUnit.perform(session, revision);
            } catch (Exception e) {
                log.error("Ошибка при выполнении единицы работы", e);
                throw new AuditException("Ошибка аудита", e);
            }
        }
    }
    
    @Override
    public void doAfterTransactionCompletion(boolean success) {
        if (success) {
            log.info("Транзакция аудита завершена успешно");
            workUnits.clear();
        } else {
            log.warn("Транзакция аудита откачена");
            // Очистка ресурсов при откате
            cleanupOnRollback();
        }
    }
    
    private void validateWorkUnit(AuditWorkUnit workUnit) {
        if (workUnit == null) {
            throw new IllegalArgumentException("Единица работы не может быть null");
        }
        
        if (workUnit.getEntityName() == null) {
            throw new IllegalArgumentException("Имя сущности не может быть null");
        }
    }
    
    private void cleanupOnRollback() {
        // Очистка ресурсов при откате транзакции
        workUnits.clear();
    }
}
```

## Класс AuditWorkUnit

`AuditWorkUnit` представляет единицу работы аудита - конкретное изменение сущности, которое должно быть зафиксировано в аудите.

### Типы AuditWorkUnit

#### 1. AddWorkUnit

```java
public class AddWorkUnit extends AbstractAuditWorkUnit {
    
    private final Object entity;
    private final Object[] state;
    
    public AddWorkUnit(Session session, String entityName, 
                      AuditEnversListener enversListener, 
                      Serializable id, Object entity, Object[] state) {
        super(session, entityName, enversListener, id);
        this.entity = entity;
        this.state = state;
    }
    
    @Override
    public boolean containsWork() {
        return true;
    }
    
    @Override
    public AuditWorkUnit merge(AuditWorkUnit other) {
        // Логика слияния с другими единицами работы
        if (other instanceof AddWorkUnit) {
            return this; // Добавление остается добавлением
        }
        return null; // Нельзя объединить с другими типами
    }
    
    @Override
    public void perform(Session session, Object revision) {
        // Создание записи в аудит-таблице
        String auditTableName = getAuditTableName();
        
        // Подготовка данных для вставки
        Object[] auditState = getAuditState();
        
        // Выполнение вставки
        session.createSQLQuery("INSERT INTO " + auditTableName + 
                             " (id, rev, revtype, ...) VALUES (?, ?, ?, ...)")
               .setParameter(1, getId())
               .setParameter(2, revision)
               .setParameter(3, RevisionType.ADD)
               .executeUpdate();
    }
    
    private Object[] getAuditState() {
        // Преобразование состояния сущности в аудит-формат
        return enversListener.getAuditState(entity, state);
    }
}
```

#### 2. ModWorkUnit

```java
public class ModWorkUnit extends AbstractAuditWorkUnit {
    
    private final Object[] oldState;
    private final Object[] newState;
    
    public ModWorkUnit(Session session, String entityName, 
                      AuditEnversListener enversListener, 
                      Serializable id, Object[] oldState, Object[] newState) {
        super(session, entityName, enversListener, id);
        this.oldState = oldState;
        this.newState = newState;
    }
    
    @Override
    public boolean containsWork() {
        // Проверяем, есть ли реальные изменения
        return !Arrays.equals(oldState, newState);
    }
    
    @Override
    public void perform(Session session, Object revision) {
        if (!containsWork()) {
            return; // Нет изменений для аудита
        }
        
        // Создание записи об изменении
        String auditTableName = getAuditTableName();
        
        // Подготовка данных
        Object[] auditState = getAuditState();
        
        // Вставка записи об изменении
        session.createSQLQuery("INSERT INTO " + auditTableName + 
                             " (id, rev, revtype, ...) VALUES (?, ?, ?, ...)")
               .setParameter(1, getId())
               .setParameter(2, revision)
               .setParameter(3, RevisionType.MOD)
               .executeUpdate();
    }
}
```

#### 3. DelWorkUnit

```java
public class DelWorkUnit extends AbstractAuditWorkUnit {
    
    private final Object[] state;
    
    public DelWorkUnit(Session session, String entityName, 
                      AuditEnversListener enversListener, 
                      Serializable id, Object[] state) {
        super(session, entityName, enversListener, id);
        this.state = state;
    }
    
    @Override
    public boolean containsWork() {
        return true; // Удаление всегда требует аудита
    }
    
    @Override
    public void perform(Session session, Object revision) {
        // Создание записи об удалении
        String auditTableName = getAuditTableName();
        
        // Сохранение состояния перед удалением
        Object[] auditState = getAuditState();
        
        // Вставка записи об удалении
        session.createSQLQuery("INSERT INTO " + auditTableName + 
                             " (id, rev, revtype, ...) VALUES (?, ?, ?, ...)")
               .setParameter(1, getId())
               .setParameter(2, revision)
               .setParameter(3, RevisionType.DEL)
               .executeUpdate();
    }
}
```

## Кастомизация AuditProcess

### Создание кастомного AuditProcess

```java
@Component
public class CustomAuditProcess extends AuditProcess {
    
    private final AuditMetricsService metricsService;
    private final AuditNotificationService notificationService;
    
    public CustomAuditProcess(AuditMetricsService metricsService,
                            AuditNotificationService notificationService) {
        this.metricsService = metricsService;
        this.notificationService = notificationService;
    }
    
    @Override
    public void addWorkUnit(AuditWorkUnit workUnit) {
        // Сбор метрик
        metricsService.recordWorkUnit(workUnit);
        
        // Проверка критических изменений
        if (isCriticalChange(workUnit)) {
            notificationService.notifyCriticalChange(workUnit);
        }
        
        super.addWorkUnit(workUnit);
    }
    
    @Override
    public void doBeforeTransactionCompletion() {
        long startTime = System.currentTimeMillis();
        
        try {
            super.doBeforeTransactionCompletion();
            
            // Запись метрик производительности
            long duration = System.currentTimeMillis() - startTime;
            metricsService.recordAuditDuration(duration);
            
        } catch (Exception e) {
            metricsService.recordAuditError(e);
            throw e;
        }
    }
    
    private boolean isCriticalChange(AuditWorkUnit workUnit) {
        // Логика определения критических изменений
        return workUnit.getEntityName().equals("User") ||
               workUnit.getEntityName().equals("Transaction");
    }
}
```

### Кастомизация AuditWorkUnit

```java
public class CustomAddWorkUnit extends AddWorkUnit {
    
    private final AuditEnhancer auditEnhancer;
    
    public CustomAddWorkUnit(Session session, String entityName, 
                           AuditEnversListener enversListener, 
                           Serializable id, Object entity, Object[] state,
                           AuditEnhancer auditEnhancer) {
        super(session, entityName, enversListener, id, entity, state);
        this.auditEnhancer = auditEnhancer;
    }
    
    @Override
    public void perform(Session session, Object revision) {
        // Улучшение данных перед аудитом
        Object enhancedEntity = auditEnhancer.enhance(entity);
        
        // Выполнение стандартного аудита
        super.perform(session, revision);
        
        // Дополнительная логика
        performCustomAudit(session, revision, enhancedEntity);
    }
    
    private void performCustomAudit(Session session, Object revision, Object entity) {
        // Кастомная логика аудита
        if (entity instanceof User) {
            User user = (User) entity;
            
            // Аудит дополнительной информации
            session.createSQLQuery("INSERT INTO user_audit_extended " +
                                 "(user_id, rev, ip_address, user_agent) " +
                                 "VALUES (?, ?, ?, ?)")
                   .setParameter(1, user.getId())
                   .setParameter(2, revision)
                   .setParameter(3, getCurrentIpAddress())
                   .setParameter(4, getCurrentUserAgent())
                   .executeUpdate();
        }
    }
    
    private String getCurrentIpAddress() {
        // Получение IP адреса текущего пользователя
        return SecurityContextHolder.getContext().getAuthentication() != null ?
               SecurityContextHolder.getContext().getAuthentication().getDetails().toString() : 
               "unknown";
    }
    
    private String getCurrentUserAgent() {
        // Получение User-Agent
        HttpServletRequest request = ((ServletRequestAttributes) 
            RequestContextHolder.currentRequestAttributes()).getRequest();
        return request.getHeader("User-Agent");
    }
}
```

## Интеграция с Spring

### Конфигурация кастомного AuditProcess

```java
@Configuration
public class CustomEnversConfig {
    
    @Bean
    public AuditProcess customAuditProcess(AuditMetricsService metricsService,
                                         AuditNotificationService notificationService) {
        return new CustomAuditProcess(metricsService, notificationService);
    }
    
    @Bean
    public AuditWorkUnitFactory customWorkUnitFactory() {
        return new CustomAuditWorkUnitFactory();
    }
}

@Component
public class CustomAuditWorkUnitFactory implements AuditWorkUnitFactory {
    
    @Autowired
    private AuditEnhancer auditEnhancer;
    
    @Override
    public AuditWorkUnit createAddWorkUnit(Session session, String entityName,
                                         AuditEnversListener enversListener,
                                         Serializable id, Object entity, Object[] state) {
        return new CustomAddWorkUnit(session, entityName, enversListener, 
                                   id, entity, state, auditEnhancer);
    }
    
    @Override
    public AuditWorkUnit createModWorkUnit(Session session, String entityName,
                                         AuditEnversListener enversListener,
                                         Serializable id, Object[] oldState, Object[] newState) {
        return new CustomModWorkUnit(session, entityName, enversListener, 
                                   id, oldState, newState);
    }
    
    @Override
    public AuditWorkUnit createDelWorkUnit(Session session, String entityName,
                                         AuditEnversListener enversListener,
                                         Serializable id, Object[] state) {
        return new CustomDelWorkUnit(session, entityName, enversListener, 
                                   id, state);
    }
}
```

## Мониторинг и метрики

### Сбор метрик AuditProcess

```java
@Component
public class AuditMetricsService {
    
    private final MeterRegistry meterRegistry;
    private final Counter workUnitCounter;
    private final Timer auditTimer;
    private final Counter errorCounter;
    
    public AuditMetricsService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.workUnitCounter = Counter.builder("envers.workunits")
                                     .description("Количество единиц работы аудита")
                                     .register(meterRegistry);
        this.auditTimer = Timer.builder("envers.audit.duration")
                              .description("Время выполнения аудита")
                              .register(meterRegistry);
        this.errorCounter = Counter.builder("envers.errors")
                                  .description("Количество ошибок аудита")
                                  .register(meterRegistry);
    }
    
    public void recordWorkUnit(AuditWorkUnit workUnit) {
        workUnitCounter.increment();
        
        // Метрики по типам сущностей
        Counter.builder("envers.workunits.by.entity")
               .tag("entity", workUnit.getEntityName())
               .tag("type", workUnit.getClass().getSimpleName())
               .register(meterRegistry)
               .increment();
    }
    
    public void recordAuditDuration(long durationMs) {
        auditTimer.record(durationMs, TimeUnit.MILLISECONDS);
    }
    
    public void recordAuditError(Exception e) {
        errorCounter.increment();
        
        // Метрики по типам ошибок
        Counter.builder("envers.errors.by.type")
               .tag("error", e.getClass().getSimpleName())
               .register(meterRegistry)
               .increment();
    }
}
```

## Оптимизация производительности

### Батчинг единиц работы

```java
@Component
public class BatchedAuditProcess extends AuditProcess {
    
    private final int batchSize = 100;
    private final List<AuditWorkUnit> batch = new ArrayList<>();
    
    @Override
    public void addWorkUnit(AuditWorkUnit workUnit) {
        batch.add(workUnit);
        
        if (batch.size() >= batchSize) {
            processBatch();
        }
    }
    
    @Override
    public void doBeforeTransactionCompletion() {
        // Обработка оставшихся единиц работы
        if (!batch.isEmpty()) {
            processBatch();
        }
    }
    
    private void processBatch() {
        if (batch.isEmpty()) {
            return;
        }
        
        // Группировка по типам операций для оптимизации
        Map<String, List<AuditWorkUnit>> groupedByEntity = batch.stream()
            .collect(Collectors.groupingBy(AuditWorkUnit::getEntityName));
        
        for (Map.Entry<String, List<AuditWorkUnit>> entry : groupedByEntity.entrySet()) {
            processEntityBatch(entry.getKey(), entry.getValue());
        }
        
        batch.clear();
    }
    
    private void processEntityBatch(String entityName, List<AuditWorkUnit> workUnits) {
        // Оптимизированная обработка батча для одной сущности
        String auditTableName = getAuditTableName(entityName);
        
        // Подготовка batch insert
        StringBuilder sql = new StringBuilder();
        sql.append("INSERT INTO ").append(auditTableName);
        sql.append(" (id, rev, revtype, ...) VALUES ");
        
        List<Object[]> parameters = new ArrayList<>();
        
        for (AuditWorkUnit workUnit : workUnits) {
            parameters.add(new Object[]{workUnit.getId(), workUnit.getRevision(), 
                                      workUnit.getRevisionType()});
        }
        
        // Выполнение batch insert
        executeBatchInsert(sql.toString(), parameters);
    }
}
```

## Тестирование

### Unit тесты для AuditWorkUnit

```java
@ExtendWith(MockitoExtension.class)
class CustomAddWorkUnitTest {
    
    @Mock
    private Session session;
    
    @Mock
    private AuditEnversListener enversListener;
    
    @Mock
    private AuditEnhancer auditEnhancer;
    
    @Test
    void perform_ShouldExecuteCustomAudit() {
        // Given
        User user = new User("test", "test@example.com");
        Object[] state = new Object[]{user.getName(), user.getEmail()};
        Object revision = new Object();
        
        CustomAddWorkUnit workUnit = new CustomAddWorkUnit(
            session, "User", enversListener, 1L, user, state, auditEnhancer
        );
        
        when(auditEnhancer.enhance(user)).thenReturn(user);
        
        // When
        workUnit.perform(session, revision);
        
        // Then
        verify(session).createSQLQuery(contains("user_audit_extended"));
        verify(auditEnhancer).enhance(user);
    }
}
```

### Integration тесты для AuditProcess

```java
@SpringBootTest
class CustomAuditProcessIntegrationTest {
    
    @Autowired
    private EntityManager entityManager;
    
    @Autowired
    private AuditMetricsService metricsService;
    
    @Test
    void shouldRecordMetricsOnAudit() {
        // Given
        User user = new User("test", "test@example.com");
        
        // When
        entityManager.persist(user);
        entityManager.flush();
        
        // Then
        // Проверяем, что метрики были записаны
        // (через моки или прямую проверку)
    }
}
```

## Заключение

Классы `AuditProcess` и `AuditWorkUnit` являются фундаментальными компонентами архитектуры Hibernate Envers. Понимание их работы позволяет:

- Кастомизировать процесс аудита под специфические требования
- Оптимизировать производительность аудита
- Добавлять дополнительную логику и метрики
- Интегрировать аудит с бизнес-процессами
- Обеспечивать надежность и отказоустойчивость

При работе с этими классами важно помнить о производительности, правильной обработке ошибок и соблюдении принципов ACID транзакций. 