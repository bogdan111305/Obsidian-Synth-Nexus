# Подключение зависимостей ehcache

EhCache является одним из самых популярных провайдеров кэширования для Hibernate. Правильное подключение и настройка зависимостей EhCache критически важны для корректной работы Second Level Cache.

## Зависимости Maven

### Основные зависимости

```xml
<dependencies>
    <!-- Hibernate Core -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>5.6.15.Final</version>
    </dependency>
    
    <!-- EhCache для Hibernate -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-ehcache</artifactId>
        <version>5.6.15.Final</version>
    </dependency>
    
    <!-- EhCache Core -->
    <dependency>
        <groupId>net.sf.ehcache</groupId>
        <artifactId>ehcache</artifactId>
        <version>2.10.9.2</version>
    </dependency>
    
    <!-- EhCache Web (для веб-приложений) -->
    <dependency>
        <groupId>net.sf.ehcache</groupId>
        <artifactId>ehcache-web</artifactId>
        <version>2.0.4</version>
    </dependency>
</dependencies>
```

### Spring Boot зависимости

```xml
<dependencies>
    <!-- Spring Boot Starter Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- EhCache для Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    
    <!-- EhCache -->
    <dependency>
        <groupId>net.sf.ehcache</groupId>
        <artifactId>ehcache</artifactId>
    </dependency>
</dependencies>
```

### Gradle зависимости

```gradle
dependencies {
    // Hibernate
    implementation 'org.hibernate:hibernate-core:5.6.15.Final'
    implementation 'org.hibernate:hibernate-ehcache:5.6.15.Final'
    
    // EhCache
    implementation 'net.sf.ehcache:ehcache:2.10.9.2'
    implementation 'net.sf.ehcache:ehcache-web:2.0.4'
    
    // Spring Boot
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-cache'
}
```

## Конфигурация EhCache

### Базовая конфигурация

```java
@Configuration
public class EhCacheConfiguration {
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        
        Properties properties = new Properties();
        
        // Включение Second Level Cache
        properties.setProperty("hibernate.cache.use_second_level_cache", "true");
        properties.setProperty("hibernate.cache.use_query_cache", "true");
        
        // Настройка EhCache как провайдера
        properties.setProperty("hibernate.cache.region.factory_class", 
                             "org.hibernate.cache.ehcache.EhCacheRegionFactory");
        
        // Путь к конфигурационному файлу EhCache
        properties.setProperty("hibernate.cache.ehcache.config", "ehcache.xml");
        
        // Настройка статистики
        properties.setProperty("hibernate.generate_statistics", "true");
        
        em.setJpaProperties(properties);
        return em;
    }
}
```

### Программная конфигурация EhCache

```java
@Configuration
public class ProgrammaticEhCacheConfiguration {
    
    @Bean
    public CacheManager ehCacheManager() {
        CacheManager cacheManager = CacheManager.create();
        
        // Создание кэша для сущностей
        Cache entityCache = new Cache(
            "entityCache",           // имя кэша
            10000,                   // максимальное количество элементов в памяти
            false,                   // eternal (вечный)
            false,                   // overflowToDisk
            3600,                    // timeToLiveSeconds
            1800,                    // timeToIdleSeconds
            false,                   // diskPersistent
            0,                       // diskExpiryThreadIntervalSeconds
            null,                    // registeredEventListeners
            null                     // registeredCacheExtensions
        );
        
        cacheManager.addCache(entityCache);
        
        // Создание кэша для коллекций
        Cache collectionCache = new Cache(
            "collectionCache",
            5000,
            false,
            false,
            3600,
            1800,
            false,
            0,
            null,
            null
        );
        
        cacheManager.addCache(collectionCache);
        
        // Создание кэша для запросов
        Cache queryCache = new Cache(
            "queryCache",
            1000,
            false,
            false,
            1800,
            900,
            false,
            0,
            null,
            null
        );
        
        cacheManager.addCache(queryCache);
        
        return cacheManager;
    }
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        
        Properties properties = new Properties();
        
        // Включение Second Level Cache
        properties.setProperty("hibernate.cache.use_second_level_cache", "true");
        properties.setProperty("hibernate.cache.use_query_cache", "true");
        
        // Использование программной конфигурации
        properties.setProperty("hibernate.cache.region.factory_class", 
                             "org.hibernate.cache.ehcache.EhCacheRegionFactory");
        
        // Отключение автоматической загрузки конфигурации
        properties.setProperty("hibernate.cache.ehcache.config", "");
        
        em.setJpaProperties(properties);
        return em;
    }
}
```

## Конфигурационный файл ehcache.xml

### Базовая конфигурация

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false">

    <!-- Глобальные настройки -->
    <defaultCache
        maxElementsInMemory="10000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="120"
        overflowToDisk="true"
        maxElementsOnDisk="10000000"
        diskPersistent="false"
        diskExpiryThreadIntervalSeconds="120"
        memoryStoreEvictionPolicy="LRU" />

    <!-- Кэш для сущностей -->
    <cache name="com.example.entity.User"
           maxElementsInMemory="1000"
           eternal="false"
           timeToIdleSeconds="3600"
           timeToLiveSeconds="3600"
           overflowToDisk="true"
           maxElementsOnDisk="10000"
           diskPersistent="false"
           diskExpiryThreadIntervalSeconds="120"
           memoryStoreEvictionPolicy="LRU" />

    <!-- Кэш для коллекций -->
    <cache name="com.example.entity.User.orders"
           maxElementsInMemory="500"
           eternal="false"
           timeToIdleSeconds="3600"
           timeToLiveSeconds="3600"
           overflowToDisk="true"
           maxElementsOnDisk="5000"
           diskPersistent="false"
           diskExpiryThreadIntervalSeconds="120"
           memoryStoreEvictionPolicy="LRU" />

    <!-- Кэш для запросов -->
    <cache name="org.hibernate.cache.internal.StandardQueryCache"
           maxElementsInMemory="1000"
           eternal="false"
           timeToIdleSeconds="1800"
           timeToLiveSeconds="1800"
           overflowToDisk="true"
           maxElementsOnDisk="10000"
           diskPersistent="false"
           diskExpiryThreadIntervalSeconds="120"
           memoryStoreEvictionPolicy="LRU" />

    <!-- Кэш для временных меток -->
    <cache name="org.hibernate.cache.spi.UpdateTimestampsCache"
           maxElementsInMemory="10000"
           eternal="true"
           overflowToDisk="false"
           memoryStoreEvictionPolicy="LRU" />

</ehcache>
```

### Расширенная конфигурация

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false"
         monitoring="autodetect"
         dynamicConfig="true">

    <!-- Настройки диска -->
    <diskStore path="java.io.tmpdir/ehcache" />

    <!-- Глобальные настройки -->
    <defaultCache
        maxElementsInMemory="10000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="120"
        overflowToDisk="true"
        maxElementsOnDisk="10000000"
        diskPersistent="false"
        diskExpiryThreadIntervalSeconds="120"
        memoryStoreEvictionPolicy="LRU"
        diskSpoolBufferSizeMB="30"
        maxMemoryPolicy="LRU" />

    <!-- Кэш для часто читаемых сущностей -->
    <cache name="com.example.entity.Category"
           maxElementsInMemory="500"
           eternal="true"
           overflowToDisk="false"
           memoryStoreEvictionPolicy="LRU" />

    <!-- Кэш для редко изменяемых сущностей -->
    <cache name="com.example.entity.Country"
           maxElementsInMemory="200"
           eternal="true"
           overflowToDisk="false"
           memoryStoreEvictionPolicy="LRU" />

    <!-- Кэш для часто изменяемых сущностей -->
    <cache name="com.example.entity.User"
           maxElementsInMemory="1000"
           eternal="false"
           timeToIdleSeconds="1800"
           timeToLiveSeconds="3600"
           overflowToDisk="true"
           maxElementsOnDisk="10000"
           diskPersistent="false"
           diskExpiryThreadIntervalSeconds="120"
           memoryStoreEvictionPolicy="LRU"
           diskSpoolBufferSizeMB="30" />

    <!-- Кэш для больших коллекций -->
    <cache name="com.example.entity.User.orders"
           maxElementsInMemory="200"
           eternal="false"
           timeToIdleSeconds="1800"
           timeToLiveSeconds="3600"
           overflowToDisk="true"
           maxElementsOnDisk="5000"
           diskPersistent="false"
           diskExpiryThreadIntervalSeconds="120"
           memoryStoreEvictionPolicy="LRU"
           diskSpoolBufferSizeMB="30" />

    <!-- Кэш для результатов запросов -->
    <cache name="org.hibernate.cache.internal.StandardQueryCache"
           maxElementsInMemory="500"
           eternal="false"
           timeToIdleSeconds="900"
           timeToLiveSeconds="1800"
           overflowToDisk="true"
           maxElementsOnDisk="5000"
           diskPersistent="false"
           diskExpiryThreadIntervalSeconds="120"
           memoryStoreEvictionPolicy="LRU"
           diskSpoolBufferSizeMB="30" />

    <!-- Кэш для временных меток -->
    <cache name="org.hibernate.cache.spi.UpdateTimestampsCache"
           maxElementsInMemory="10000"
           eternal="true"
           overflowToDisk="false"
           memoryStoreEvictionPolicy="LRU" />

    <!-- Кэш для статистики -->
    <cache name="statisticsCache"
           maxElementsInMemory="1000"
           eternal="false"
           timeToIdleSeconds="300"
           timeToLiveSeconds="600"
           overflowToDisk="false"
           memoryStoreEvictionPolicy="LRU" />

</ehcache>
```

## Spring Boot конфигурация

### application.properties

```properties
# Hibernate Second Level Cache
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory
hibernate.cache.ehcache.config=ehcache.xml

# Статистика
hibernate.generate_statistics=true

# Настройки EhCache
spring.cache.type=ehcache
spring.cache.ehcache.config=ehcache.xml

# Настройки JPA
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.EhCacheRegionFactory
spring.jpa.properties.hibernate.cache.ehcache.config=ehcache.xml
spring.jpa.properties.hibernate.generate_statistics=true
```

### application.yml

```yaml
spring:
  cache:
    type: ehcache
    ehcache:
      config: ehcache.xml
  
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.ehcache.EhCacheRegionFactory
          ehcache:
            config: ehcache.xml
        generate_statistics: true
```

## Настройка кэширования сущностей

### Аннотации для кэширования

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    private String email;
    
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
    
    // Геттеры и сеттеры...
}

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Category {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String description;
    
    // Геттеры и сеттеры...
}

@Entity
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private BigDecimal price;
    
    @ManyToOne
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    private Category category;
    
    // Геттеры и сеттеры...
}
```

### Стратегии кэширования

```java
public enum CacheConcurrencyStrategy {
    
    // Только чтение - для неизменяемых данных
    READ_ONLY,
    
    // Чтение-запись - для часто изменяемых данных
    READ_WRITE,
    
    // Нестрогое чтение-запись - для редко изменяемых данных
    NONSTRICT_READ_WRITE,
    
    // Транзакционное - для критически важных данных
    TRANSACTIONAL
}

// Примеры использования
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY) // Для справочников
public class Country { }

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // Для пользователей
public class User { }

@Entity
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE) // Для продуктов
public class Product { }

@Entity
@Cache(usage = CacheConcurrencyStrategy.TRANSACTIONAL) // Для финансовых данных
public class Transaction { }
```

## Мониторинг и управление

### Мониторинг EhCache

```java
@Service
public class EhCacheMonitor {
    
    @Autowired
    private CacheManager cacheManager;
    
    public void monitorCache() {
        String[] cacheNames = cacheManager.getCacheNames();
        
        for (String cacheName : cacheNames) {
            Cache cache = cacheManager.getCache(cacheName);
            
            System.out.println("=== Cache: " + cacheName + " ===");
            System.out.println("Size: " + cache.getSize());
            System.out.println("Memory Store Size: " + cache.getMemoryStoreSize());
            System.out.println("Disk Store Size: " + cache.getDiskStoreSize());
            System.out.println("Hit Count: " + cache.getStatistics().getCacheHits());
            System.out.println("Miss Count: " + cache.getStatistics().getCacheMisses());
            System.out.println("Hit Ratio: " + cache.getStatistics().getCacheHitRatio());
            System.out.println();
        }
    }
    
    public void evictCache(String cacheName) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.removeAll();
            System.out.println("Evicted cache: " + cacheName);
        }
    }
    
    public void evictAllCaches() {
        cacheManager.clearAll();
        System.out.println("Evicted all caches");
    }
}
```

### Управление кэшем через JMX

```java
@Configuration
public class EhCacheJMXConfiguration {
    
    @Bean
    public MBeanServer mBeanServer() {
        return ManagementFactory.getPlatformMBeanServer();
    }
    
    @Bean
    public EhCacheMBeanServer ehCacheMBeanServer(CacheManager cacheManager, 
                                                MBeanServer mBeanServer) {
        EhCacheMBeanServer ehCacheMBeanServer = new EhCacheMBeanServer();
        ehCacheMBeanServer.setCacheManager(cacheManager);
        ehCacheMBeanServer.setMBeanServer(mBeanServer);
        return ehCacheMBeanServer;
    }
}
```

## Тестирование

### Unit тесты

```java
@ExtendWith(MockitoExtension.class)
class EhCacheConfigurationTest {
    
    @Test
    void shouldConfigureEhCache() {
        // Given
        Properties properties = new Properties();
        properties.setProperty("hibernate.cache.use_second_level_cache", "true");
        properties.setProperty("hibernate.cache.region.factory_class", 
                             "org.hibernate.cache.ehcache.EhCacheRegionFactory");
        
        // When
        boolean secondLevelCacheEnabled = Boolean.parseBoolean(
            properties.getProperty("hibernate.cache.use_second_level_cache"));
        String regionFactoryClass = properties.getProperty("hibernate.cache.region.factory_class");
        
        // Then
        assertTrue(secondLevelCacheEnabled);
        assertEquals("org.hibernate.cache.ehcache.EhCacheRegionFactory", regionFactoryClass);
    }
}
```

### Integration тесты

```java
@SpringBootTest
class EhCacheIntegrationTest {
    
    @Autowired
    private SessionFactory sessionFactory;
    
    @Test
    void shouldUseEhCache() {
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
    
    @Test
    void shouldCacheQueryResults() {
        Session session = sessionFactory.openSession();
        
        // Первый запрос
        List<User> users1 = session.createQuery("FROM User WHERE active = true")
            .setCacheable(true)
            .list();
        
        // Второй запрос - должен использовать кэш
        List<User> users2 = session.createQuery("FROM User WHERE active = true")
            .setCacheable(true)
            .list();
        
        session.close();
        
        assertEquals(users1.size(), users2.size());
    }
}
```

## Производительность и оптимизация

### Настройка производительности

```java
@Configuration
public class EhCachePerformanceConfiguration {
    
    @Bean
    public CacheManager highPerformanceCacheManager() {
        CacheManager cacheManager = CacheManager.create();
        
        // Кэш с высокой производительностью
        Cache highPerfCache = new Cache(
            "highPerfCache",
            50000,                   // больше элементов в памяти
            false,
            false,
            7200,                    // больше TTL
            3600,
            true,                    // включить диск
            60,
            null,
            null
        );
        
        cacheManager.addCache(highPerfCache);
        
        return cacheManager;
    }
    
    @Bean
    public CacheManager memoryOptimizedCacheManager() {
        CacheManager cacheManager = CacheManager.create();
        
        // Кэш с оптимизацией памяти
        Cache memoryOptCache = new Cache(
            "memoryOptCache",
            1000,                    // меньше элементов
            false,
            false,
            1800,                    // меньше TTL
            900,
            false,                   // отключить диск
            0,
            null,
            null
        );
        
        cacheManager.addCache(memoryOptCache);
        
        return cacheManager;
    }
}
```

## Заключение

Правильное подключение и настройка EhCache для Hibernate включает:

- **Зависимости**: Корректное подключение всех необходимых библиотек
- **Конфигурация**: Настройка EhCache через XML или программно
- **Стратегии**: Выбор подходящих стратегий кэширования для разных типов данных
- **Мониторинг**: Отслеживание производительности и управление кэшем
- **Оптимизация**: Настройка параметров для достижения оптимальной производительности

При работе с EhCache важно помнить о:
- Правильном выборе стратегий кэширования
- Мониторинге производительности
- Управлении памятью и дисковым пространством
- Тестировании в различных сценариях нагрузки 