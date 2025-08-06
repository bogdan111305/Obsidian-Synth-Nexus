# Маппинг наследования в Hibernate: стратегии и практическое применение

## Оглавление
1. [Введение в маппинг наследования](#введение)
2. [@MappedSuperclass](#mapped-superclass)
    1. [Создание класса BaseEntity](#base-entity)
    2. [Аннотация @MappedSuperclass](#mapped-superclass-annotation)
    3. [Замена BaseEntity на интерфейс](#interface-replacement)
    4. [Создание класса AuditableEntity](#auditable-entity)
3. [Стратегия TABLE_PER_CLASS](#table-per-class)
    1. [Реализация стратегии](#table-per-class-implementation)
    2. [Аннотация @Inheritance](#inheritance-annotation)
    3. [Тестирование функциональности](#table-per-class-testing)
    4. [Ограничения IDENTITY](#identity-limitations)
4. [Стратегия SINGLE_TABLE](#single-table)
    1. [Реализация стратегии](#single-table-implementation)
    2. [Тестирование функционала](#single-table-testing)
5. [Стратегия JOINED](#joined)
    1. [Реализация стратегии](#joined-implementation)
    2. [Тестирование функционала](#joined-testing)
6. [Сравнение стратегий](#comparison)
7. [Лучшие практики](#практики)
8. [Вопросы для собеседования](#вопросы)

---

## <a name="введение"></a>Введение в маппинг наследования

Наследование в Hibernate позволяет моделировать иерархии классов Java в реляционных базах данных. Hibernate поддерживает несколько стратегий маппинга наследования: `@MappedSuperclass`, `TABLE_PER_CLASS`, `SINGLE_TABLE` и `JOINED`. Каждая стратегия подходит для разных сценариев, и выбор зависит от требований к производительности, структуре базы данных и сложности модели.

### Стратегии наследования

- **@MappedSuperclass**: Используется для выноса общих полей и маппингов в базовый класс, не создавая отдельной таблицы.
- **TABLE_PER_CLASS**: Каждому классу в иерархии соответствует отдельная таблица.
- **SINGLE_TABLE**: Все классы иерархии хранятся в одной таблице с дискриминатором.
- **JOINED**: Общие поля хранятся в таблице базового класса, а специфические — в таблицах дочерних классов.

Эта статья подробно описывает каждую стратегию, включая реализацию, тестирование и ограничения, с примерами кода и SQL-структур.

---

## <a name="mapped-superclass"></a>@MappedSuperclass

Аннотация `@MappedSuperclass` используется для создания базового класса, содержащего общие поля и маппинги, которые наследуются сущностями, но не мапятся на отдельную таблицу.

### <a name="base-entity"></a>Создание класса BaseEntity

Класс `BaseEntity` обычно содержит общие атрибуты, такие как идентификатор, временные метки или другие метаданные, которые используются всеми сущностями.

```java
import jakarta.persistence.Id;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.MappedSuperclass;

@MappedSuperclass
public class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
}
```

**Работа**:

- Поля `BaseEntity` (например, `id`) наследуются всеми дочерними сущностями.
- Hibernate мапит эти поля на столбцы в таблицах дочерних сущностей.
- Сам `BaseEntity` не создаёт отдельной таблицы в базе данных.

**Использование**:

```java
import jakarta.persistence.Entity;

@Entity
public class User extends BaseEntity {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

**SQL**:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255)
);
```

### <a name="mapped-superclass-annotation"></a>Аннотация @MappedSuperclass

- **Назначение**: Помечает класс как базовый для наследования маппингов, но не как сущность.
- **Особенности**:
    - Не создаёт таблицу в базе данных.
    - Поля и аннотации наследуются дочерними классами.
    - Не поддерживает полиморфные запросы, так как не является сущностью.
- **Применение**: Используется для общих атрибутов, таких как `id`, `createdAt`, `updatedAt`.

### <a name="interface-replacement"></a>Замена BaseEntity на интерфейс

Замена `BaseEntity` на интерфейс может быть предпочтительной в следующих случаях:

- **Гибкость**: Интерфейс позволяет сущностям не зависеть от конкретной реализации базового класса, избегая жёсткой иерархии.
- **Разделение логики**: Интерфейс определяет контракт (например, наличие `getId()`), а реализация остаётся в сущностях.
- **Избежание дублирования маппинга**: Если разные сущности требуют разные стратегии генерации ID, интерфейс позволяет избежать конфликтов.
- **Поддержка независимых иерархий**: Сущности могут реализовать интерфейс, не наследуя общий класс, что полезно для сложных моделей.

**Пример с интерфейсом**:

```java
public interface Identifiable {
    Long getId();
    void setId(Long id);
}

@Entity
public class User implements Identifiable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @Override
    public Long getId() {
        return id;
    }

    @Override
    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

**Проблемы с BaseEntity**:

- Жёсткое наследование ограничивает гибкость (например, нельзя использовать разные стратегии генерации ID).
- Может привести к нежелательному наследованию методов или полей.

### <a name="auditable-entity"></a>Создание класса AuditableEntity

Класс `AuditableEntity` расширяет `BaseEntity`, добавляя поля для аудита (например, временные метки).

```java
import jakarta.persistence.MappedSuperclass;
import jakarta.persistence.Temporal;
import jakarta.persistence.TemporalType;
import java.util.Date;

@MappedSuperclass
public class AuditableEntity extends BaseEntity {
    @Temporal(TemporalType.TIMESTAMP)
    private Date createdAt;

    @Temporal(TemporalType.TIMESTAMP)
    private Date updatedAt;

    public Date getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(Date createdAt) {
        this.createdAt = createdAt;
    }

    public Date getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(Date updatedAt) {
        this.updatedAt = updatedAt;
    }
}
```

**Использование**:

```java
@Entity
public class Employee extends AuditableEntity {
    private String position;

    public String getPosition() {
        return position;
    }

    public void setPosition(String position) {
        this.position = position;
    }
}
```

**SQL**:

```sql
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    position VARCHAR(255)
);
```

**Работа**:

- Поля `id`, `createdAt` и `updatedAt` наследуются в таблицу `employees`.
- Подходит для стандартизации аудита во всех сущностях.

---

## <a name="table-per-class"></a>Стратегия TABLE_PER_CLASS

Стратегия `TABLE_PER_CLASS` мапит каждый класс иерархии на отдельную таблицу, содержащую все поля, включая унаследованные.

### <a name="table-per-class-implementation"></a>Реализация стратегии TABLE_PER_CLASS

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;
import jakarta.persistence.Id;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;

@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    private String brand;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }
}

@Entity
public class Car extends Vehicle {
    private int seats;

    public int getSeats() {
        return seats;
    }

    public void setSeats(int seats) {
        this.seats = seats;
    }
}

@Entity
public class Truck extends Vehicle {
    private double loadCapacity;

    public double getLoadCapacity() {
        return loadCapacity;
    }

    public void setLoadCapacity(double loadCapacity) {
        this.loadCapacity = loadCapacity;
    }
}
```

**SQL**:

```sql
CREATE TABLE cars (
    id BIGINT PRIMARY KEY,
    brand VARCHAR(255),
    seats INTEGER
);

CREATE TABLE trucks (
    id BIGINT PRIMARY KEY,
    brand VARCHAR(255),
    load_capacity DOUBLE PRECISION
);
```

**Работа**:

- Каждая сущность (`Car`, `Truck`) мапится на отдельную таблицу.
- Общие поля (`id`, `brand`) дублируются в каждой таблице.
- Используется стратегия `SEQUENCE` для генерации ID.

### <a name="inheritance-annotation"></a>Аннотация @Inheritance

- **Назначение**: Указывает стратегию наследования для иерархии сущностей.
- **Атрибут strategy**: `InheritanceType.TABLE_PER_CLASS` задаёт маппинг каждой сущности на отдельную таблицу.
- **Особенности**:
    - Не требует дискриминатора, так как таблицы полностью независимы.
    - Поддерживает полиморфные запросы, но с использованием `UNION`.

### <a name="table-per-class-testing"></a>Тестирование функциональности

```java
// Создание и сохранение объектов
Car car = new Car();
car.setBrand("Toyota");
car.setSeats(5);
em.persist(car);

Truck truck = new Truck();
truck.setBrand("Volvo");
truck.setLoadCapacity(10.5);
em.persist(truck);

// Полиморфный запрос
List<Vehicle> vehicles = em.createQuery("SELECT v FROM Vehicle v", Vehicle.class).getResultList();
// Генерирует UNION запрос для всех таблиц в иерархии
```

### <a name="identity-limitations"></a>Ограничения IDENTITY

**Проблема**: `GenerationType.IDENTITY` не работает с `TABLE_PER_CLASS`, так как:

- Каждая таблица имеет свой автоинкремент.
- При полиморфных запросах могут возникнуть конфликты ID.
- Hibernate не может гарантировать уникальность ID между таблицами.

**Решение**: Используйте `GenerationType.SEQUENCE` или `GenerationType.TABLE`.

---

## <a name="single-table"></a>Стратегия SINGLE_TABLE

Стратегия `SINGLE_TABLE` хранит всю иерархию в одной таблице с дискриминатором для различения типов.

### <a name="single-table-implementation"></a>Реализация стратегии SINGLE_TABLE

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;
import jakarta.persistence.DiscriminatorColumn;
import jakarta.persistence.DiscriminatorType;
import jakarta.persistence.DiscriminatorValue;

@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "vehicle_type", discriminatorType = DiscriminatorType.STRING)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String brand;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }
}

@Entity
@DiscriminatorValue("CAR")
public class Car extends Vehicle {
    private int seats;

    public int getSeats() {
        return seats;
    }

    public void setSeats(int seats) {
        this.seats = seats;
    }
}

@Entity
@DiscriminatorValue("TRUCK")
public class Truck extends Vehicle {
    private double loadCapacity;

    public double getLoadCapacity() {
        return loadCapacity;
    }

    public void setLoadCapacity(double loadCapacity) {
        this.loadCapacity = loadCapacity;
    }
}
```

**SQL**:

```sql
CREATE TABLE vehicle (
    id BIGSERIAL PRIMARY KEY,
    vehicle_type VARCHAR(31) NOT NULL,
    brand VARCHAR(255),
    seats INTEGER,
    load_capacity DOUBLE PRECISION
);
```

**Работа**:

- Все сущности хранятся в одной таблице `vehicle`.
- Столбец `vehicle_type` используется как дискриминатор.
- Специфичные поля (`seats`, `load_capacity`) могут быть NULL для других типов.

### <a name="single-table-testing"></a>Тестирование функционала

```java
// Создание и сохранение объектов
Car car = new Car();
car.setBrand("Toyota");
car.setSeats(5);
em.persist(car);

Truck truck = new Truck();
truck.setBrand("Volvo");
truck.setLoadCapacity(10.5);
em.persist(truck);

// Полиморфный запрос
List<Vehicle> vehicles = em.createQuery("SELECT v FROM Vehicle v", Vehicle.class).getResultList();
// Простой SELECT без UNION
```

---

## <a name="joined"></a>Стратегия JOINED

Стратегия `JOINED` использует отдельные таблицы для каждого класса с внешними ключами для связи.

### <a name="joined-implementation"></a>Реализация стратегии JOINED

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;
import jakarta.persistence.PrimaryKeyJoinColumn;

@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String brand;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }
}

@Entity
@PrimaryKeyJoinColumn(name = "vehicle_id")
public class Car extends Vehicle {
    private int seats;

    public int getSeats() {
        return seats;
    }

    public void setSeats(int seats) {
        this.seats = seats;
    }
}

@Entity
@PrimaryKeyJoinColumn(name = "vehicle_id")
public class Truck extends Vehicle {
    private double loadCapacity;

    public double getLoadCapacity() {
        return loadCapacity;
    }

    public void setLoadCapacity(double loadCapacity) {
        this.loadCapacity = loadCapacity;
    }
}
```

**SQL**:

```sql
CREATE TABLE vehicle (
    id BIGSERIAL PRIMARY KEY,
    brand VARCHAR(255)
);

CREATE TABLE car (
    vehicle_id BIGINT PRIMARY KEY,
    seats INTEGER,
    FOREIGN KEY (vehicle_id) REFERENCES vehicle(id)
);

CREATE TABLE truck (
    vehicle_id BIGINT PRIMARY KEY,
    load_capacity DOUBLE PRECISION,
    FOREIGN KEY (vehicle_id) REFERENCES vehicle(id)
);
```

**Работа**:

- Общие поля хранятся в таблице `vehicle`.
- Специфичные поля хранятся в отдельных таблицах (`car`, `truck`).
- Связь осуществляется через внешние ключи.

### <a name="joined-testing"></a>Тестирование функционала

```java
// Создание и сохранение объектов
Car car = new Car();
car.setBrand("Toyota");
car.setSeats(5);
em.persist(car);

Truck truck = new Truck();
truck.setBrand("Volvo");
truck.setLoadCapacity(10.5);
em.persist(truck);

// Полиморфный запрос
List<Vehicle> vehicles = em.createQuery("SELECT v FROM Vehicle v", Vehicle.class).getResultList();
// Использует LEFT JOIN для загрузки связанных данных
```

---

## <a name="comparison"></a>Сравнение стратегий

| Стратегия | Производительность | Простота запросов | Нормализация | Дискриминатор |
|-----------|-------------------|-------------------|--------------|---------------|
| **SINGLE_TABLE** | Высокая | Простые | Нет | Требуется |
| **JOINED** | Средняя | Сложные (JOIN) | Да | Не требуется |
| **TABLE_PER_CLASS** | Низкая | Сложные (UNION) | Нет | Не требуется |
| **@MappedSuperclass** | Высокая | Простые | Нет | Не поддерживает полиморфизм |

### Рекомендации по выбору

- **SINGLE_TABLE**: Для простых иерархий с небольшим количеством полей.
- **JOINED**: Для сложных иерархий с нормализованной структурой.
- **TABLE_PER_CLASS**: Для независимых сущностей с минимальным общим кодом.
- **@MappedSuperclass**: Для общих полей без полиморфных запросов.

---

## <a name="практики"></a>Лучшие практики

### Выбор стратегии

- Анализируйте требования к производительности и сложности запросов.
- Учитывайте размер иерархии и количество полей.
- Тестируйте производительность с реальными данными.

### Производительность

- Избегайте глубоких иерархий с JOINED стратегией.
- Используйте индексы для дискриминаторов в SINGLE_TABLE.
- Оптимизируйте запросы с помощью fetch joins.

### Поддержка кода

- Документируйте выбор стратегии и его обоснование.
- Используйте интерфейсы вместо жёсткого наследования.
- Пишите тесты для всех сценариев использования.

---

## <a name="вопросы"></a>Вопросы для собеседования

1. Какие стратегии наследования поддерживает Hibernate?
2. В чём разница между @MappedSuperclass и @Inheritance?
3. Как работает стратегия SINGLE_TABLE?
4. Какие проблемы возникают с TABLE_PER_CLASS и IDENTITY?
5. Когда стоит использовать JOINED стратегию?
6. Что такое дискриминатор и зачем он нужен?
7. Как влияет выбор стратегии на производительность?
8. Какие ограничения у @MappedSuperclass?
9. Как реализовать полиморфные запросы в разных стратегиях?
10. Как выбрать оптимальную стратегию для конкретного случая?

---

**Рекомендуемые материалы:**
- [Hibernate Inheritance Mapping](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#entity-inheritance)
- [JPA Inheritance](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#a13346)
- [Vlad Mihalcea: Hibernate Inheritance](https://vladmihalcea.com/hibernate-inheritance/)