# Аннотация @Cache

## Оглавление
1. [Что такое @Cache](#что-такое)
2. [CacheConcurrencyStrategy: все стратегии](#стратегии)
3. [READ_ONLY — для справочников](#read-only)
4. [NONSTRICT_READ_WRITE — редкие обновления](#nonstrict)
5. [READ_WRITE — гарантированная согласованность](#read-write)
6. [TRANSACTIONAL — JTA-транзакции](#transactional)
7. [Кэширование коллекций](#коллекции)
8. [Примеры конфигурации](#примеры)

---

## 1. Что такое @Cache <a name="что-такое"></a>

`@Cache` — Hibernate-аннотация (не JPA!) для включения Second Level Cache на уровне entity или коллекции.

```java
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // включает L2 Cache
public class Product {
    @Id
    private Long id;
    private String name;
}
```

> [!INFO]
> JPA предоставляет аналог через `@jakarta.persistence.Cacheable`, но без контроля стратегии. Используй Hibernate `@Cache` для полного контроля.

Без `@Cache` entity **не попадёт** в L2 Cache даже при включённом `use_second_level_cache=true`.

---

## 2. CacheConcurrencyStrategy: все стратегии <a name="стратегии"></a>

Стратегия определяет, **как** кэш обеспечивает согласованность при конкурентных операциях чтения/записи.

| Стратегия | Запись | Чтение | Согласованность | Overhead |
|-----------|--------|--------|-----------------|---------|
| READ_ONLY | Нет | Да | Полная | Минимальный |
| NONSTRICT_READ_WRITE | Редко | Да | Eventual | Низкий |
| READ_WRITE | Да | Да | Строгая | Средний |
| TRANSACTIONAL | Да (JTA) | Да | Полная (ACID) | Высокий |

---

## 3. READ_ONLY — для справочников <a name="read-only"></a>

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Country {
    @Id
    private Long id;
    private String code;
    private String name;
}
```

**Характеристики**:
- Entity **никогда не изменяется** после создания
- Нет механизма блокировки — самый быстрый
- Попытка `update` → `UnsupportedOperationException`
- Hibernate оптимизирует: кэширует данные напрямую без копирования

**Когда использовать**: справочники стран, валют, кодов, типов, ролей — всё что меняется только через миграцию данных, не через приложение.

> [!WARNING]
> При `READ_ONLY` нельзя изменять entity через Hibernate. Если нужны редкие обновления (раз в месяц, только через admin) — используй `NONSTRICT_READ_WRITE`.

---

## 4. NONSTRICT_READ_WRITE — редкие обновления <a name="nonstrict"></a>

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
public class Category {
    @Id
    private Long id;
    private String name;
    private String description;
}
```

**Характеристики**:
- Entity меняется **редко**
- При обновлении: запись **сразу удаляется** из кэша (invalidation)
- Кратковременная несогласованность возможна: другие потоки могут прочитать из кэша **между** удалением и следующей загрузкой из БД
- Нет механизма блокировки (soft lock)

**Что происходит при update**:
```
Thread 1: читает Category из кэша (HIT) → кэш = { name: "Old" }
Thread 2: обновляет Category → инвалидирует кэш
Thread 1: (уже прочитал старые данные — inconsistency в памяти)
Thread 3: читает Category → MISS → SELECT из БД → { name: "New" } → в кэш
```

**Когда использовать**: категории товаров, теги, настройки — меняются редко, и brief inconsistency допустима.

---

## 5. READ_WRITE — гарантированная согласованность <a name="read-write"></a>

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    private String name;
    private BigDecimal price;

    @Version
    private Long version;
}
```

**Характеристики**:
- Использует механизм **soft lock**: при обновлении запись в кэше временно заменяется маркером блокировки
- Пока идёт обновление — другие запросы идут в БД (cache miss)
- После commit — кэш обновляется новыми данными
- Гарантирует строгую согласованность

**Что происходит при update**:
```
Thread 1: начинает UPDATE Product#1 → устанавливает soft lock в кэше
Thread 2: запрашивает Product#1 → видит soft lock → идёт в БД (MISS)
Thread 1: коммитит → удаляет soft lock, кладёт новые данные в кэш
Thread 3: запрашивает Product#1 → HIT, получает актуальные данные
```

**Overhead**: дополнительные операции с кэшем (lock/unlock) на каждый UPDATE.

**Когда использовать**: entity, которые читаются часто и обновляются умеренно, при этом несогласованность недопустима.

> [!WARNING]
> `READ_WRITE` работает только с **optimistic locking** и **локальным** кэшем. В кластерном окружении без distributed cache soft lock не защищает от гонки между нодами.

---

## 6. TRANSACTIONAL — JTA-транзакции <a name="transactional"></a>

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.TRANSACTIONAL)
public class Account {
    @Id
    private Long id;
    private BigDecimal balance;
}
```

**Характеристики**:
- Только для **JTA-окружения** (JEE/Jakarta EE, не Spring по умолчанию)
- Кэш участвует в JTA-транзакции: при rollback транзакции — rollback изменений в кэше
- Полная ACID-семантика кэша
- Требует кэш-провайдер с поддержкой XA (Infinispan, JBoss Cache)

> [!INFO]
> В Spring Boot приложениях `TRANSACTIONAL` практически не используется. Выбирай между `READ_WRITE` (согласованность) и `NONSTRICT_READ_WRITE` (производительность).

---

## 7. Кэширование коллекций <a name="коллекции"></a>

Коллекции кэшируются **отдельно** от entity. Нужно добавить `@Cache` и на entity, и на коллекцию:

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Department {
    @Id
    private Long id;

    @OneToMany(mappedBy = "department")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // отдельный кэш для коллекции!
    private List<Employee> employees = new ArrayList<>();
}
```

> [!WARNING]
> Если пометить только entity, но не коллекцию — коллекция будет каждый раз загружаться из БД. Кэш хранит только IDs элементов коллекции, а сами entity берутся из кэша entity.

**Что хранит кэш коллекции**:
```
Department#1.employees → [Employee#1, Employee#3, Employee#7]  (только IDs)
```
При чтении: IDs из кэша коллекции + Employee из кэша entity по ID (или SELECT если нет в кэше).

---

## 8. Примеры конфигурации <a name="примеры"></a>

### Полный пример entity с кэшем

```java
@Entity
@Table(name = "products")
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(precision = 19, scale = 2)
    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @OneToMany(mappedBy = "product")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "product-tags")
    private Set<Tag> tags = new HashSet<>();

    @Version
    private Long version;
}
```

### Справочник только для чтения

```java
@Entity
@Table(name = "currencies")
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "reference-data")
@Immutable // Hibernate: гарантирует что entity не изменяется (дополнительная оптимизация)
public class Currency {
    @Id
    private String code; // "USD", "EUR"
    private String name;
    private Integer numericCode;
}
```

### Конфигурация региона в ehcache.xml

```xml
<config xmlns="http://www.ehcache.org/v3">
    <cache alias="products">
        <expiry>
            <ttl unit="minutes">30</ttl>
        </expiry>
        <resources>
            <heap unit="entries">5000</heap>
            <offheap unit="MB">100</offheap>
        </resources>
    </cache>

    <cache alias="reference-data">
        <expiry>
            <none/>
        </expiry>
        <resources>
            <heap unit="entries">500</heap>
        </resources>
    </cache>
</config>
```

---

**Связанные файлы:**
- [[First Level Cache vs Second Level Cache]] — обзор архитектуры кэширования
- [[Cache Concurrency Strategy]] — регионы и их конфигурация
- [[Кэширование коллекций]] — детали кэширования коллекций
- [[Подключение зависимостей ehcache]] — подключение провайдера
