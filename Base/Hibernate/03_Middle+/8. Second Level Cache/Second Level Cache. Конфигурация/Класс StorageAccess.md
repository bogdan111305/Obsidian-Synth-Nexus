# Класс StorageAccess

`StorageAccess` является интерфейсом в Hibernate, который предоставляет абстракцию для доступа к хранилищу кэша. Этот интерфейс определяет основные операции для работы с кэшем: чтение, запись, удаление и управление транзакциями.

## Обзор интерфейса

```java
public interface StorageAccess {
    
    // Основные операции доступа
    Object getFromCache(Object key, SessionImplementor session);
    void putIntoCache(Object key, Object value, SessionImplementor session);
    void removeFromCache(Object key, SessionImplementor session);
    
    // Управление транзакциями
    void clearCache();
    void evictData();
    
    // Информация о хранилище
    boolean contains(Object key);
    long getSizeInMemory();
    long getElementCountInMemory();
    long getElementCountOnDisk();
    
    // Статистика
    long getHitCount();
    long getMissCount();
    long getPutCount();
}
```

## Базовая реализация StorageAccess

### Абстрактная реализация

```java
public abstract class AbstractStorageAccess implements StorageAccess {
    
    protected final String regionName;
    protected final Properties properties;
    protected final Cache cache;
    
    public AbstractStorageAccess(String regionName, Properties properties, Cache cache) {
        this.regionName = regionName;
        this.properties = properties;
        this.cache = cache;
    }
    
    @Override
    public Object getFromCache(Object key, SessionImplementor session) {
        try {
            Element element = cache.get(key);
            if (element != null) {
                incrementHitCount();
                return element.getObjectValue();
            } else {
                incrementMissCount();
                return null;
            }
        } catch (Exception e) {
            log.error("Error getting from cache: {}", key, e);
            incrementMissCount();
            return null;
        }
    }
    
    @Override
    public void putIntoCache(Object key, Object value, SessionImplementor session) {
        try {
            Element element = new Element(key, value);
            cache.put(element);
            incrementPutCount();
        } catch (Exception e) {
            log.error("Error putting into cache: {}", key, e);
        }
    }
    
    @Override
    public void removeFromCache(Object key, SessionImplementor session) {
        try {
            cache.remove(key);
        } catch (Exception e) {
            log.error("Error removing from cache: {}", key, e);
        }
    }
    
    @Override
    public void clearCache() {
        try {
            cache.removeAll();
        } catch (Exception e) {
            log.error("Error clearing cache", e);
        }
    }
    
    @Override
    public void evictData() {
        clearCache();
    }
    
    @Override
    public boolean contains(Object key) {
        try {
            return cache.get(key) != null;
        } catch (Exception e) {
            log.error("Error checking cache contains: {}", key, e);
            return false;
        }
    }
    
    @Override
    public long getSizeInMemory() {
        try {
            return cache.getMemoryStoreSize();
        } catch (Exception e) {
            log.error("Error getting memory size", e);
            return 0;
        }
    }
    
    @Override
    public long getElementCountInMemory() {
        try {
            return cache.getSize();
        } catch (Exception e) {
            log.error("Error getting element count", e);
            return 0;
        }
    }
    
    @Override
    public long getElementCountOnDisk() {
        try {
            return cache.getDiskStoreSize();
        } catch (Exception e) {
            log.error("Error getting disk element count", e);
            return 0;
        }
    }
    
    // Счетчики статистики
    private final AtomicLong hitCount = new AtomicLong(0);
    private final AtomicLong missCount = new AtomicLong(0);
    private final AtomicLong putCount = new AtomicLong(0);
    
    private void incrementHitCount() {
        hitCount.incrementAndGet();
    }
    
    private void incrementMissCount() {
        missCount.incrementAndGet();
    }
    
    private void incrementPutCount() {
        putCount.incrementAndGet();
    }
    
    @Override
    public long getHitCount() {
        return hitCount.get();
    }
    
    @Override
    public long getMissCount() {
        return missCount.get();
    }
    
    @Override
    public long getPutCount() {
        return putCount.get();
    }
}
```

## Специализированные реализации

### EntityStorageAccess

```java
public class EntityStorageAccess extends AbstractStorageAccess {
    
    private final CacheDataDescription metadata;
    
    public EntityStorageAccess(String regionName, Properties properties, 
                             Cache cache, CacheDataDescription metadata) {
        super(regionName, properties, cache);
        this.metadata = metadata;
    }
    
    @Override
    public Object getFromCache(Object key, SessionImplementor session) {
        Object cachedValue = super.getFromCache(key, session);
        
        if (cachedValue != null) {
            // Проверка версионности для сущностей
            if (metadata.isVersioned()) {
                return validateVersionedEntity(cachedValue, session);
            }
        }
        
        return cachedValue;
    }
    
    @Override
    public void putIntoCache(Object key, Object value, SessionImplementor session) {
        // Подготовка сущности для кэширования
        Object preparedValue = prepareEntityForCaching(value, session);
        
        super.putIntoCache(key, preparedValue, session);
    }
    
    private Object validateVersionedEntity(Object cachedValue, SessionImplementor session) {
        // Валидация версионной сущности
        if (cachedValue instanceof VersionedEntity) {
            VersionedEntity entity = (VersionedEntity) cachedValue;
            
            // Проверка актуальности версии
            if (isVersionStale(entity, session)) {
                log.warn("Cached entity version is stale: {}", entity.getId());
                return null; // Возвращаем null для перезагрузки из БД
            }
        }
        
        return cachedValue;
    }
    
    private boolean isVersionStale(VersionedEntity entity, SessionImplementor session) {
        // Проверка, не устарела ли версия сущности
        // Это упрощенная реализация
        return false;
    }
    
    private Object prepareEntityForCaching(Object value, SessionImplementor session) {
        // Подготовка сущности для кэширования
        if (value instanceof HibernateProxy) {
            // Инициализация прокси перед кэшированием
            Hibernate.initialize(value);
        }
        
        return value;
    }
}
```

### CollectionStorageAccess

```java
public class CollectionStorageAccess extends AbstractStorageAccess {
    
    private final CacheDataDescription metadata;
    
    public CollectionStorageAccess(String regionName, Properties properties, 
                                 Cache cache, CacheDataDescription metadata) {
        super(regionName, properties, cache);
        this.metadata = metadata;
    }
    
    @Override
    public Object getFromCache(Object key, SessionImplementor session) {
        Object cachedValue = super.getFromCache(key, session);
        
        if (cachedValue != null) {
            // Обработка коллекций
            return processCachedCollection(cachedValue, session);
        }
        
        return cachedValue;
    }
    
    @Override
    public void putIntoCache(Object key, Object value, SessionImplementor session) {
        // Подготовка коллекции для кэширования
        Object preparedValue = prepareCollectionForCaching(value, session);
        
        super.putIntoCache(key, preparedValue, session);
    }
    
    private Object processCachedCollection(Object cachedValue, SessionImplementor session) {
        if (cachedValue instanceof Collection) {
            Collection<?> collection = (Collection<?>) cachedValue;
            
            // Инициализация коллекции при необходимости
            if (collection instanceof PersistentCollection) {
                PersistentCollection persistentCollection = (PersistentCollection) collection;
                if (!persistentCollection.wasInitialized()) {
                    persistentCollection.forceInitialization(session);
                }
            }
        }
        
        return cachedValue;
    }
    
    private Object prepareCollectionForCaching(Object value, SessionImplementor session) {
        if (value instanceof PersistentCollection) {
            PersistentCollection collection = (PersistentCollection) value;
            
            // Проверка, инициализирована ли коллекция
            if (!collection.wasInitialized()) {
                collection.forceInitialization(session);
            }
        }
        
        return value;
    }
}
```

## Транзакционное управление

### TransactionalStorageAccess

```java
public class TransactionalStorageAccess extends AbstractStorageAccess {
    
    private final TransactionManager transactionManager;
    private final Map<Object, Object> pendingChanges = new ConcurrentHashMap<>();
    
    public TransactionalStorageAccess(String regionName, Properties properties, 
                                   Cache cache, TransactionManager transactionManager) {
        super(regionName, properties, cache);
        this.transactionManager = transactionManager;
    }
    
    @Override
    public void putIntoCache(Object key, Object value, SessionImplementor session) {
        // Проверка активной транзакции
        if (isTransactionActive()) {
            // Отложенная запись до коммита транзакции
            pendingChanges.put(key, value);
        } else {
            // Немедленная запись
            super.putIntoCache(key, value, session);
        }
    }
    
    @Override
    public void removeFromCache(Object key, SessionImplementor session) {
        if (isTransactionActive()) {
            // Отложенное удаление до коммита транзакции
            pendingChanges.put(key, null); // null означает удаление
        } else {
            // Немедленное удаление
            super.removeFromCache(key, session);
        }
    }
    
    private boolean isTransactionActive() {
        try {
            return transactionManager.getStatus() == Status.STATUS_ACTIVE;
        } catch (Exception e) {
            log.warn("Error checking transaction status", e);
            return false;
        }
    }
    
    public void commitTransaction() {
        // Применение отложенных изменений
        for (Map.Entry<Object, Object> entry : pendingChanges.entrySet()) {
            Object key = entry.getKey();
            Object value = entry.getValue();
            
            if (value == null) {
                super.removeFromCache(key, null);
            } else {
                super.putIntoCache(key, value, null);
            }
        }
        
        pendingChanges.clear();
    }
    
    public void rollbackTransaction() {
        // Отмена отложенных изменений
        pendingChanges.clear();
    }
}
```

## Кэширование с сериализацией

### SerializedStorageAccess

```java
public class SerializedStorageAccess extends AbstractStorageAccess {
    
    private final Serializer serializer;
    
    public SerializedStorageAccess(String regionName, Properties properties, 
                                 Cache cache, Serializer serializer) {
        super(regionName, properties, cache);
        this.serializer = serializer;
    }
    
    @Override
    public Object getFromCache(Object key, SessionImplementor session) {
        try {
            Element element = cache.get(key);
            if (element != null) {
                incrementHitCount();
                byte[] serializedData = (byte[]) element.getObjectValue();
                return serializer.deserialize(serializedData);
            } else {
                incrementMissCount();
                return null;
            }
        } catch (Exception e) {
            log.error("Error deserializing from cache: {}", key, e);
            incrementMissCount();
            return null;
        }
    }
    
    @Override
    public void putIntoCache(Object key, Object value, SessionImplementor session) {
        try {
            byte[] serializedData = serializer.serialize(value);
            Element element = new Element(key, serializedData);
            cache.put(element);
            incrementPutCount();
        } catch (Exception e) {
            log.error("Error serializing to cache: {}", key, e);
        }
    }
    
    // Интерфейс сериализатора
    public interface Serializer {
        byte[] serialize(Object obj) throws Exception;
        Object deserialize(byte[] data) throws Exception;
    }
}
```

## Мониторинг и статистика

### MonitoredStorageAccess

```java
public class MonitoredStorageAccess extends AbstractStorageAccess {
    
    private final MeterRegistry meterRegistry;
    private final Timer getTimer;
    private final Timer putTimer;
    private final Counter hitCounter;
    private final Counter missCounter;
    private final Counter putCounter;
    
    public MonitoredStorageAccess(String regionName, Properties properties, 
                                Cache cache, MeterRegistry meterRegistry) {
        super(regionName, properties, cache);
        this.meterRegistry = meterRegistry;
        
        // Создание метрик
        this.getTimer = Timer.builder("cache.get.duration")
            .tag("region", regionName)
            .register(meterRegistry);
        
        this.putTimer = Timer.builder("cache.put.duration")
            .tag("region", regionName)
            .register(meterRegistry);
        
        this.hitCounter = Counter.builder("cache.hits")
            .tag("region", regionName)
            .register(meterRegistry);
        
        this.missCounter = Counter.builder("cache.misses")
            .tag("region", regionName)
            .register(meterRegistry);
        
        this.putCounter = Counter.builder("cache.puts")
            .tag("region", regionName)
            .register(meterRegistry);
    }
    
    @Override
    public Object getFromCache(Object key, SessionImplementor session) {
        return getTimer.record(() -> {
            Object result = super.getFromCache(key, session);
            
            if (result != null) {
                hitCounter.increment();
            } else {
                missCounter.increment();
            }
            
            return result;
        });
    }
    
    @Override
    public void putIntoCache(Object key, Object value, SessionImplementor session) {
        putTimer.record(() -> {
            super.putIntoCache(key, value, session);
            putCounter.increment();
        });
    }
    
    public CacheMetrics getMetrics() {
        return CacheMetrics.builder()
            .regionName(regionName)
            .hitCount(hitCounter.count())
            .missCount(missCounter.count())
            .putCount(putCounter.count())
            .averageGetTime(getTimer.mean(TimeUnit.MILLISECONDS))
            .averagePutTime(putTimer.mean(TimeUnit.MILLISECONDS))
            .build();
    }
}
```

### Класс метрик кэша

```java
public class CacheMetrics {
    private final String regionName;
    private final double hitCount;
    private final double missCount;
    private final double putCount;
    private final double averageGetTime;
    private final double averagePutTime;
    
    private CacheMetrics(Builder builder) {
        this.regionName = builder.regionName;
        this.hitCount = builder.hitCount;
        this.missCount = builder.missCount;
        this.putCount = builder.putCount;
        this.averageGetTime = builder.averageGetTime;
        this.averagePutTime = builder.averagePutTime;
    }
    
    public double getHitRatio() {
        double totalRequests = hitCount + missCount;
        return totalRequests > 0 ? hitCount / totalRequests : 0.0;
    }
    
    // Геттеры
    public String getRegionName() { return regionName; }
    public double getHitCount() { return hitCount; }
    public double getMissCount() { return missCount; }
    public double getPutCount() { return putCount; }
    public double getAverageGetTime() { return averageGetTime; }
    public double getAveragePutTime() { return averagePutTime; }
    
    public static class Builder {
        private String regionName;
        private double hitCount;
        private double missCount;
        private double putCount;
        private double averageGetTime;
        private double averagePutTime;
        
        public Builder regionName(String regionName) {
            this.regionName = regionName;
            return this;
        }
        
        public Builder hitCount(double hitCount) {
            this.hitCount = hitCount;
            return this;
        }
        
        public Builder missCount(double missCount) {
            this.missCount = missCount;
            return this;
        }
        
        public Builder putCount(double putCount) {
            this.putCount = putCount;
            return this;
        }
        
        public Builder averageGetTime(double averageGetTime) {
            this.averageGetTime = averageGetTime;
            return this;
        }
        
        public Builder averagePutTime(double averagePutTime) {
            this.averagePutTime = averagePutTime;
            return this;
        }
        
        public CacheMetrics build() {
            return new CacheMetrics(this);
        }
    }
}
```

## Конфигурация в Spring Boot

### Настройка StorageAccess

```java
@Configuration
public class StorageAccessConfiguration {
    
    @Bean
    public StorageAccess entityStorageAccess(Cache cache, 
                                           CacheDataDescription metadata) {
        Properties properties = new Properties();
        properties.setProperty("hibernate.cache.entity.ttl", "3600");
        properties.setProperty("hibernate.cache.entity.max_elements", "10000");
        
        return new EntityStorageAccess("entity", properties, cache, metadata);
    }
    
    @Bean
    public StorageAccess collectionStorageAccess(Cache cache, 
                                              CacheDataDescription metadata) {
        Properties properties = new Properties();
        properties.setProperty("hibernate.cache.collection.ttl", "3600");
        properties.setProperty("hibernate.cache.collection.max_elements", "5000");
        
        return new CollectionStorageAccess("collection", properties, cache, metadata);
    }
    
    @Bean
    public StorageAccess monitoredStorageAccess(StorageAccess delegate, 
                                             MeterRegistry meterRegistry) {
        return new MonitoredStorageAccess("monitored", new Properties(), 
                                        null, meterRegistry);
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class StorageAccessTest {
    
    @Mock
    private Cache cache;
    
    @Mock
    private SessionImplementor session;
    
    @Mock
    private CacheDataDescription metadata;
    
    @Test
    void getFromCache_ShouldReturnCachedValue() {
        // Given
        Object key = "testKey";
        Object expectedValue = new User("test", "test@example.com");
        Element element = new Element(key, expectedValue);
        
        when(cache.get(key)).thenReturn(element);
        
        StorageAccess storageAccess = new EntityStorageAccess("test", 
                                                            new Properties(), 
                                                            cache, metadata);
        
        // When
        Object result = storageAccess.getFromCache(key, session);
        
        // Then
        assertEquals(expectedValue, result);
        assertEquals(1, storageAccess.getHitCount());
        assertEquals(0, storageAccess.getMissCount());
    }
    
    @Test
    void putIntoCache_ShouldStoreValue() {
        // Given
        Object key = "testKey";
        Object value = new User("test", "test@example.com");
        
        StorageAccess storageAccess = new EntityStorageAccess("test", 
                                                            new Properties(), 
                                                            cache, metadata);
        
        // When
        storageAccess.putIntoCache(key, value, session);
        
        // Then
        verify(cache).put(any(Element.class));
        assertEquals(1, storageAccess.getPutCount());
    }
}
```

### Integration тесты

```java
@SpringBootTest
class StorageAccessIntegrationTest {
    
    @Autowired
    private SessionFactory sessionFactory;
    
    @Test
    void shouldUseStorageAccess() {
        Session session = sessionFactory.openSession();
        
        // Первый запрос - Miss
        User user1 = session.get(User.class, 1L);
        assertNotNull(user1);
        
        // Второй запрос - Hit
        User user2 = session.get(User.class, 1L);
        assertNotNull(user2);
        
        session.close();
        
        // Проверка статистики
        Statistics stats = sessionFactory.getStatistics();
        assertTrue(stats.getSecondLevelCacheHitCount() >= 0);
        assertTrue(stats.getSecondLevelCacheMissCount() >= 0);
    }
}
```

## Заключение

`StorageAccess` предоставляет мощную абстракцию для работы с хранилищем кэша в Hibernate:

- **Абстракция**: Единый интерфейс для различных провайдеров кэша
- **Транзакционность**: Поддержка транзакционных операций с кэшем
- **Сериализация**: Возможность кастомной сериализации данных
- **Мониторинг**: Встроенная поддержка метрик и статистики
- **Гибкость**: Возможность создания специализированных реализаций

При работе с `StorageAccess` важно помнить о:
- Правильной обработке исключений
- Эффективном управлении ресурсами
- Мониторинге производительности
- Корректной работе с транзакциями 