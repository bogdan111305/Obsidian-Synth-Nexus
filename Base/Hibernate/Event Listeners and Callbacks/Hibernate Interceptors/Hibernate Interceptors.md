# Hibernate Interceptors

Hibernate Interceptors предоставляют мощный механизм для перехвата и модификации операций с сущностями на низком уровне. Они позволяют добавлять дополнительную логику без изменения основного кода сущностей.

## Содержание

1. [Основы интерцепторов](#основы-интерцепторов)
2. [Интерфейс Interceptor](#интерфейс-interceptor)
3. [EmptyInterceptor (deprecated)](#emptyinterceptor-deprecated)
4. [Регистрация интерцепторов](#регистрация-интерцепторов)
5. [Жизненный цикл интерцептора](#жизненный-цикл-интерцептора)
6. [Практические примеры](#практические-примеры)
7. [Лучшие практики](#лучшие-практики)

## Основы интерцепторов

### Что такое Hibernate Interceptors?

Hibernate Interceptors - это компоненты, которые позволяют перехватывать операции с сущностями на различных этапах их жизненного цикла. Они работают на уровне Hibernate Session и предоставляют более низкоуровневый контроль, чем Event Listeners.

### Зачем нужны интерцепторы?

- **Аудит** - отслеживание изменений сущностей
- **Валидация** - проверка данных перед сохранением
- **Логирование** - запись операций с сущностями
- **Модификация данных** - изменение значений перед сохранением
- **Безопасность** - проверка прав доступа

## Интерфейс Interceptor

### Основной интерфейс

```java
public interface Interceptor {
    
    /**
     * Вызывается при загрузке сущности
     */
    default boolean onLoad(Object entity, Object id, Object[] state, 
                          String[] propertyNames, Type[] types) throws CallbackException {
        return false;
    }
    
    /**
     * Вызывается при сохранении новой сущности
     */
    default boolean onSave(Object entity, Object id, Object[] state, 
                          String[] propertyNames, Type[] types) throws CallbackException {
        return false;
    }
    
    /**
     * Вызывается при обновлении сущности
     */
    default boolean onFlushDirty(Object entity, Object id, Object[] currentState, 
                                Object[] previousState, String[] propertyNames, 
                                Type[] types) throws CallbackException {
        return false;
    }
    
    /**
     * Вызывается при удалении сущности
     */
    default boolean onDelete(Object entity, Object id, Object[] state, 
                           String[] propertyNames, Type[] types) throws CallbackException {
        return false;
    }
    
    /**
     * Вызывается при создании коллекции
     */
    default boolean onCollectionRecreate(Object collection, Object key) throws CallbackException {
        return false;
    }
    
    /**
     * Вызывается при удалении коллекции
     */
    default boolean onCollectionRemove(Object collection, Object key) throws CallbackException {
        return false;
    }
    
    /**
     * Вызывается при обновлении коллекции
     */
    default boolean onCollectionUpdate(Object collection, Object key) throws CallbackException {
        return false;
    }
    
    /**
     * Вызывается перед выполнением SQL
     */
    default String onPrepareStatement(String sql) {
        return sql;
    }
}
```

### Возвращаемые значения

- **true** - данные были изменены интерцептором
- **false** - данные не изменялись

## EmptyInterceptor (deprecated)

### Базовая реализация

```java
/**
 * @deprecated Начиная с Hibernate 6.0
 * Используйте интерфейс Interceptor напрямую
 */
@Deprecated
public class EmptyInterceptor implements Interceptor {
    
    @Override
    public boolean onLoad(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        return false;
    }
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        return false;
    }
    
    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState, 
                               Object[] previousState, String[] propertyNames, 
                               Type[] types) throws CallbackException {
        return false;
    }
    
    @Override
    public boolean onDelete(Object entity, Object id, Object[] state, 
                          String[] propertyNames, Type[] types) throws CallbackException {
        return false;
    }
}
```

### Современный подход

```java
public class ModernInterceptor implements Interceptor {
    
    @Override
    public boolean onLoad(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        // Логика загрузки
        return false;
    }
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        // Логика сохранения
        return false;
    }
}
```

## Регистрация интерцепторов

### Глобальная регистрация

```java
// Регистрация для всех сессий
Configuration configuration = new Configuration();
configuration.setInterceptor(new GlobalInterceptor());

// Или через SessionFactory
SessionFactory sessionFactory = configuration.buildSessionFactory();
sessionFactory.withOptions().interceptor(new GlobalInterceptor());
```

### Регистрация для конкретной сессии

```java
// Для конкретной сессии
Session session = sessionFactory.openSession();
session = session.withOptions().interceptor(new SessionInterceptor()).openSession();

// Или через SessionBuilder
Session session = sessionFactory.withOptions()
    .interceptor(new SessionInterceptor())
    .openSession();
```

### Регистрация через Spring

```java
@Configuration
public class HibernateConfig {
    
    @Bean
    public LocalSessionFactoryBean sessionFactory() {
        LocalSessionFactoryBean sessionFactory = new LocalSessionFactoryBean();
        
        // Устанавливаем интерцептор
        sessionFactory.setEntityInterceptor(new GlobalInterceptor());
        
        return sessionFactory;
    }
}
```

## Жизненный цикл интерцептора

### Последовательность выполнения

```
Entity Operation
    ↓
Interceptor.onSave/onLoad/onFlushDirty/onDelete
    ↓
Database Operation
    ↓
Interceptor.onCollectionRecreate/onCollectionRemove/onCollectionUpdate
    ↓
Transaction Commit/Rollback
```

### Пример реализации

```java
@Component
public class AuditInterceptor implements Interceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(AuditInterceptor.class);
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
            
            logger.info("Entity {} created with id: {}", 
                       entity.getClass().getSimpleName(), id);
            
            return true; // Данные изменены
        }
        
        return false;
    }
    
    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState, 
                               Object[] previousState, String[] propertyNames, 
                               Type[] types) throws CallbackException {
        
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
            
            logger.info("Entity {} updated with id: {}", 
                       entity.getClass().getSimpleName(), id);
            
            return true; // Данные изменены
        }
        
        return false;
    }
    
    @Override
    public boolean onDelete(Object entity, Object id, Object[] state, 
                          String[] propertyNames, Type[] types) throws CallbackException {
        
        logger.info("Entity {} deleted with id: {}", 
                   entity.getClass().getSimpleName(), id);
        
        return false;
    }
    
    private String getCurrentUser() {
        // Получение текущего пользователя
        return "system";
    }
}
```

## Практические примеры

### 1. Интерцептор для аудита

```java
public class AuditInterceptor implements Interceptor {
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        if (entity instanceof BaseEntity) {
            BaseEntity baseEntity = (BaseEntity) entity;
            baseEntity.setCreatedAt(LocalDateTime.now());
            baseEntity.setCreatedBy(getCurrentUsername());
            
            return true;
        }
        
        return false;
    }
    
    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState, 
                               Object[] previousState, String[] propertyNames, 
                               Type[] types) throws CallbackException {
        
        if (entity instanceof BaseEntity) {
            BaseEntity baseEntity = (BaseEntity) entity;
            baseEntity.setUpdatedAt(LocalDateTime.now());
            baseEntity.setUpdatedBy(getCurrentUsername());
            
            return true;
        }
        
        return false;
    }
    
    private String getCurrentUsername() {
        try {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            return auth != null ? auth.getName() : "system";
        } catch (Exception e) {
            return "system";
        }
    }
}
```

### 2. Интерцептор для валидации

```java
public class ValidationInterceptor implements Interceptor {
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        validateEntity(entity);
        return false;
    }
    
    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState, 
                               Object[] previousState, String[] propertyNames, 
                               Type[] types) throws CallbackException {
        
        validateEntity(entity);
        return false;
    }
    
    private void validateEntity(Object entity) {
        if (entity instanceof Validatable) {
            Validatable validatable = (Validatable) entity;
            Set<ConstraintViolation<Validatable>> violations = 
                Validation.buildDefaultValidatorFactory().getValidator().validate(validatable);
            
            if (!violations.isEmpty()) {
                throw new ValidationException("Entity validation failed: " + violations);
            }
        }
    }
}
```

### 3. Интерцептор для логирования

```java
public class LoggingInterceptor implements Interceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        logger.info("Saving entity: {} with id: {}", 
                   entity.getClass().getSimpleName(), id);
        
        return false;
    }
    
    @Override
    public boolean onLoad(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        logger.debug("Loading entity: {} with id: {}", 
                    entity.getClass().getSimpleName(), id);
        
        return false;
    }
    
    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState, 
                               Object[] previousState, String[] propertyNames, 
                               Type[] types) throws CallbackException {
        
        logger.info("Updating entity: {} with id: {}", 
                   entity.getClass().getSimpleName(), id);
        
        // Логируем изменения
        for (int i = 0; i < propertyNames.length; i++) {
            if (!Objects.equals(currentState[i], previousState[i])) {
                logger.debug("Property {} changed from {} to {}", 
                           propertyNames[i], previousState[i], currentState[i]);
            }
        }
        
        return false;
    }
    
    @Override
    public boolean onDelete(Object entity, Object id, Object[] state, 
                          String[] propertyNames, Type[] types) throws CallbackException {
        
        logger.info("Deleting entity: {} with id: {}", 
                   entity.getClass().getSimpleName(), id);
        
        return false;
    }
}
```

### 4. Интерцептор для шифрования

```java
public class EncryptionInterceptor implements Interceptor {
    
    private final EncryptionService encryptionService;
    
    public EncryptionInterceptor(EncryptionService encryptionService) {
        this.encryptionService = encryptionService;
    }
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        if (entity instanceof Encryptable) {
            Encryptable encryptable = (Encryptable) entity;
            
            for (int i = 0; i < propertyNames.length; i++) {
                if (encryptable.isEncrypted(propertyNames[i]) && state[i] != null) {
                    state[i] = encryptionService.encrypt(state[i].toString());
                }
            }
            
            return true;
        }
        
        return false;
    }
    
    @Override
    public boolean onLoad(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        if (entity instanceof Encryptable) {
            Encryptable encryptable = (Encryptable) entity;
            
            for (int i = 0; i < propertyNames.length; i++) {
                if (encryptable.isEncrypted(propertyNames[i]) && state[i] != null) {
                    state[i] = encryptionService.decrypt(state[i].toString());
                }
            }
            
            return true;
        }
        
        return false;
    }
}
```

### 5. Интерцептор для кэширования

```java
public class CacheInterceptor implements Interceptor {
    
    private final CacheManager cacheManager;
    
    public CacheInterceptor(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }
    
    @Override
    public boolean onLoad(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        // Проверяем кэш перед загрузкой из БД
        String cacheKey = entity.getClass().getSimpleName() + ":" + id;
        Cache cache = cacheManager.getCache("entities");
        
        if (cache != null) {
            Cache.ValueWrapper cached = cache.get(cacheKey);
            if (cached != null) {
                // Возвращаем кэшированную версию
                return false;
            }
        }
        
        return false;
    }
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        // Кэшируем после сохранения
        String cacheKey = entity.getClass().getSimpleName() + ":" + id;
        Cache cache = cacheManager.getCache("entities");
        
        if (cache != null) {
            cache.put(cacheKey, entity);
        }
        
        return false;
    }
    
    @Override
    public boolean onDelete(Object entity, Object id, Object[] state, 
                          String[] propertyNames, Type[] types) throws CallbackException {
        
        // Удаляем из кэша
        String cacheKey = entity.getClass().getSimpleName() + ":" + id;
        Cache cache = cacheManager.getCache("entities");
        
        if (cache != null) {
            cache.evict(cacheKey);
        }
        
        return false;
    }
}
```

## Лучшие практики

### 1. Правильный выбор между Interceptors и Event Listeners

```java
// Используйте Interceptors для:
// - Низкоуровневого контроля
// - Модификации данных перед сохранением
// - Глобальной логики для всех сущностей

// Используйте Event Listeners для:
// - Специфичных событий
// - Более высокоуровневой логики
// - Интеграции с внешними системами
```

### 2. Производительность

```java
public class PerformanceInterceptor implements Interceptor {
    
    @Override
    public boolean onLoad(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        // Избегайте тяжелых операций в интерцепторах
        // Используйте асинхронную обработку для логирования
        
        CompletableFuture.runAsync(() -> {
            logEntityLoad(entity, id);
        });
        
        return false;
    }
    
    private void logEntityLoad(Object entity, Object id) {
        // Тяжелая операция логирования
    }
}
```

### 3. Обработка исключений

```java
public class SafeInterceptor implements Interceptor {
    
    @Override
    public boolean onSave(Object entity, Object id, Object[] state, 
                         String[] propertyNames, Type[] types) throws CallbackException {
        
        try {
            // Логика интерцептора
            return false;
        } catch (Exception e) {
            // Логируем ошибку, но не прерываем операцию
            logger.error("Error in interceptor: {}", e.getMessage(), e);
            return false;
        }
    }
}
```

### 4. Тестирование интерцепторов

```java
@SpringBootTest
class AuditInterceptorTest {
    
    @Autowired
    private SessionFactory sessionFactory;
    
    @Test
    void shouldAddAuditFieldsOnSave() {
        // Создаем сессию с интерцептором
        Session session = sessionFactory.withOptions()
            .interceptor(new AuditInterceptor())
            .openSession();
        
        try {
            session.beginTransaction();
            
            User user = new User("test@example.com");
            session.save(user);
            
            session.flush();
            
            // Проверяем, что аудит поля установлены
            assertThat(user.getCreatedAt()).isNotNull();
            assertThat(user.getCreatedBy()).isNotNull();
            
            session.getTransaction().commit();
        } finally {
            session.close();
        }
    }
}
```

### 5. Конфигурация через Spring

```java
@Configuration
public class HibernateInterceptorConfig {
    
    @Bean
    public LocalSessionFactoryBean sessionFactory(DataSource dataSource) {
        LocalSessionFactoryBean sessionFactory = new LocalSessionFactoryBean();
        sessionFactory.setDataSource(dataSource);
        sessionFactory.setPackagesToScan("com.example.entity");
        
        // Устанавливаем интерцептор
        sessionFactory.setEntityInterceptor(new AuditInterceptor());
        
        return sessionFactory;
    }
    
    @Bean
    public AuditInterceptor auditInterceptor() {
        return new AuditInterceptor();
    }
}
```

## Связанные темы

- [Entity Callbacks](../Entity Callbacks/README.md)
- [Listener Callbacks](../Listener Callbacks/README.md)
- [Hibernate Event Listeners](../Hibernate Event Listeners/README.md)
- [Hibernate Event model](../Hibernate Event Listeners/Hibernate Event model.md) 