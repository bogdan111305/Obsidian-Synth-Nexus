# Cache Concurrency Strategy и Regions

## Оглавление
1. [Что такое Region](#что-такое-region)
2. [Как задать регион](#задать-регион)
3. [Конфигурация регионов в Ehcache](#ehcache-config)
4. [Конфигурация через Caffeine](#caffeine)
5. [Стратегия выбора: какую стратегию для каких сущностей](#выбор-стратегии)
6. [Опасность нативного SQL](#native-sql-danger)
7. [Мониторинг через Statistics](#statistics)

---

## 1. Что такое Region <a name="что-такое-region"></a>

**Region** — именованное пространство в L2 Cache с независимой конфигурацией (TTL, max entries, eviction policy).

Каждая entity-класс, каждая коллекция и Query Cache имеют **свой регион**:

```
По умолчанию:
com.example.entity.Product                    → entity region
com.example.entity.Department.employees       → collection region
org.hibernate.cache.spi.QueryResultsRegion   → query cache region
org.hibernate.cache.spi.TimestampsRegion     → timestamps region (для query cache)
```

Регионы позволяют настраивать кэш **по-разному** для разных типов данных:
- Справочники (Country, Currency) → TTL = никогда, max = 500
- Продукты → TTL = 30 минут, max = 10000
- Заказы → не кэшировать

---

## 2. Как задать регион <a name="задать-регион"></a>

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "products")
public class Product { ... }

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "reference-data")
public class Country { ... }

// Коллекция — свой регион
@OneToMany(mappedBy = "department")
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "department-employees")
private List<Employee> employees;
```

> [!INFO]
> Если `region` не указан — используется полное имя класса (`com.example.Product`). Это работает, но конфигурировать такие регионы по имени сложнее.

---

## 3. Конфигурация регионов в Ehcache <a name="ehcache-config"></a>

### Подключение Ehcache 3 (JCache / JSR-107)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
            uri: classpath:ehcache.xml
```

### ehcache.xml

```xml
<config xmlns="http://www.ehcache.org/v3">

    <!-- Справочные данные: не истекают -->
    <cache alias="reference-data">
        <expiry>
            <none/>
        </expiry>
        <resources>
            <heap unit="entries">500</heap>
        </resources>
    </cache>

    <!-- Продукты: 30 минут TTL, offheap -->
    <cache alias="products">
        <expiry>
            <ttl unit="minutes">30</ttl>
        </expiry>
        <resources>
            <heap unit="entries">5000</heap>
            <offheap unit="MB">100</offheap>
        </resources>
    </cache>

    <!-- Коллекции -->
    <cache alias="department-employees">
        <expiry>
            <ttl unit="minutes">5</ttl>
        </expiry>
        <resources>
            <heap unit="entries">2000</heap>
        </resources>
    </cache>

    <!-- Query Cache -->
    <cache alias="org.hibernate.cache.spi.QueryResultsRegion">
        <expiry>
            <ttl unit="minutes">5</ttl>
        </expiry>
        <resources>
            <heap unit="entries">200</heap>
        </resources>
    </cache>

    <cache alias="org.hibernate.cache.spi.TimestampsRegion">
        <expiry>
            <none/>
        </expiry>
        <resources>
            <heap unit="entries">100</heap>
        </resources>
    </cache>

</config>
```

---

## 4. Конфигурация через Caffeine <a name="caffeine"></a>

Caffeine — in-process кэш, более простой чем Ehcache (но без offheap и disk):

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>jcache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: com.github.benmanes.caffeine.jcache.spi.CaffeineJCachingProvider
            missing_cache_strategy: create-warn
```

```java
// Программная конфигурация Caffeine регионов через @Bean
@Configuration
public class CacheConfig {

    @Bean
    public javax.cache.CacheManager caffeineCacheManager() {
        CachingProvider provider = Caching.getCachingProvider(
            "com.github.benmanes.caffeine.jcache.spi.CaffeineJCachingProvider");
        javax.cache.CacheManager cacheManager = provider.getCacheManager();

        // Регион для справочников
        cacheManager.createCache("reference-data",
            new MutableConfiguration<>()
                .setExpiryPolicyFactory(EternalExpiryPolicy.factoryOf())
                .setStatisticsEnabled(true));

        // Регион для продуктов
        cacheManager.createCache("products",
            new MutableConfiguration<>()
                .setExpiryPolicyFactory(
                    CreatedExpiryPolicy.factoryOf(new Duration(TimeUnit.MINUTES, 30)))
                .setStatisticsEnabled(true));

        return cacheManager;
    }
}
```

---

## 5. Стратегия выбора: какую стратегию для каких сущностей <a name="выбор-стратегии"></a>

### Матрица выбора

```
Данные никогда не меняются (или только через DDL)?
  → READ_ONLY + регион без TTL

Данные меняются редко (раз в день/неделю)?
  → NONSTRICT_READ_WRITE + TTL совпадает с частотой изменений

Данные меняются регулярно, но несогласованность недопустима?
  → READ_WRITE + разумный TTL

JTA-транзакции, финансовые данные?
  → TRANSACTIONAL (только JEE)
```

### Практические примеры

| Entity | Стратегия | TTL | Регион |
|--------|-----------|-----|--------|
| Country, Currency | READ_ONLY | Без TTL | reference-data |
| Category, Tag | NONSTRICT_READ_WRITE | 1 час | categories |
| Product, User | READ_WRITE | 30 мин | products |
| Order, Transaction | Не кэшировать | — | — |
| Session/Token | Не кэшировать | — | — |

> [!WARNING]
> Не кэшируй entity, которые часто меняются через bulk HQL/SQL. Инвалидация будет постоянной — кэш только добавляет overhead.

---

## 6. Опасность нативного SQL <a name="native-sql-danger"></a>

> [!WARNING]
> **UPDATE/DELETE через нативный SQL полностью обходит инвалидацию L2 Cache!**

```java
// Нативный SQL — Hibernate не знает, что изменилось
entityManager.createNativeQuery(
    "UPDATE products SET price = price * 1.1 WHERE category_id = 5")
    .executeUpdate();

// L2 Cache содержит СТАРЫЕ цены для всех продуктов из category=5!
// Все последующие find(Product.class, id) вернут устаревшие данные
// до истечения TTL или ручной инвалидации
```

**Решение: ручная инвалидация после нативного SQL**

```java
@Transactional
public void updatePrices(Long categoryId, BigDecimal multiplier) {
    entityManager.createNativeQuery(
        "UPDATE products SET price = price * :mult WHERE category_id = :catId")
        .setParameter("mult", multiplier)
        .setParameter("catId", categoryId)
        .executeUpdate();

    // Инвалидируем весь регион продуктов
    sessionFactory.getCache().evictEntityData(Product.class);
}
```

**HQL bulk update — тоже не инвалидирует автоматически!**

```java
// HQL bulk update
entityManager.createQuery(
    "UPDATE Product p SET p.price = p.price * :mult WHERE p.category.id = :catId")
    .setParameter("mult", multiplier)
    .setParameter("catId", categoryId)
    .executeUpdate();

// L2 Cache НЕ инвалидирован для затронутых Product
// Требуется ручная инвалидация аналогично нативному SQL
```

---

## 7. Мониторинг через Statistics <a name="statistics"></a>

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

```java
@Autowired
private SessionFactory sessionFactory;

public void printCacheStats() {
    Statistics stats = sessionFactory.getStatistics();

    long hits = stats.getSecondLevelCacheHitCount();
    long misses = stats.getSecondLevelCacheMissCount();
    double hitRatio = hits * 100.0 / (hits + misses);

    System.out.printf("L2 Cache Hit Ratio: %.1f%%%n", hitRatio);

    // По регионам
    for (String region : stats.getSecondLevelCacheRegionNames()) {
        CacheRegionStatistics regionStats = stats.getDomainDataRegionStatistics(region);
        System.out.printf("Region '%s': hits=%d, misses=%d, puts=%d%n",
            region,
            regionStats.getHitCount(),
            regionStats.getMissCount(),
            regionStats.getPutCount());
    }
}
```

**Интерпретация**:
- **Hit Ratio > 80%** — кэш работает хорошо
- **Hit Ratio < 50%** — TTL слишком короткий или entity обновляются слишком часто
- **Put >> Hit** — данные кладутся в кэш, но не читаются оттуда (нет смысла кэшировать)

---

**Связанные файлы:**
- [[Аннотация @Cache]] — стратегии кэширования на entity
- [[First Level Cache vs Second Level Cache]] — архитектура кэширования
- [[Hibernate statistics]] — мониторинг статистики
- [[Подключение зависимостей ehcache]] — подключение провайдера
