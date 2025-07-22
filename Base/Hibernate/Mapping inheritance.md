Inheritance. MappedSuperclass(Создание класса BaseEntity, Аннотация @MappedSuperclass, Почему нужно заменить BaseEntity на интерфейс? Создание класса AuditableEntity), Inheritance. TABLE_PER_CLASS (Реализация стратегии TABLE_PER_CLASS, Аннотация @Inheritance, Тестирование функциональности, Почему нельзя использовать IDENTITY в TABLE_PER_CLASS), Inheritance. SINGLE_TABLE(Реализация стратегии SINGLE_TABLE, Тестирование функционала), Inheritance. JOINED(Реализация стратегии JOINED, Тестирование функционала, Резюме), Резюме по всем стратегиям наследования
# Введение в маппинг наследования в Hibernate

Наследование в Hibernate позволяет моделировать иерархии классов Java в реляционных базах данных. Hibernate поддерживает несколько стратегий маппинга наследования: `@MappedSuperclass`, `TABLE_PER_CLASS`, `SINGLE_TABLE` и `JOINED`. Каждая стратегия подходит для разных сценариев, и выбор зависит от требований к производительности, структуре базы данных и сложности модели.

- **@MappedSuperclass**: Используется для выноса общих полей и маппингов в базовый класс, не создавая отдельной таблицы.
- **TABLE_PER_CLASS**: Каждому классу в иерархии соответствует отдельная таблица.
- **SINGLE_TABLE**: Все классы иерархии хранятся в одной таблице с дискриминатором.
- **JOINED**: Общие поля хранятся в таблице базового класса, а специфические — в таблицах дочерних классов.

Эта статья подробно описывает каждую стратегию, включая реализацию, тестирование и ограничения, с примерами кода и SQL-структур.

# MappedSuperclass

Аннотация `@MappedSuperclass` используется для создания базового класса, содержащего общие поля и маппинги, которые наследуются сущностями, но не мапятся на отдельную таблицу.

## Создание класса BaseEntity

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

## Аннотация @MappedSuperclass

- **Назначение**: Помечает класс как базовый для наследования маппингов, но не как сущность.
- **Особенности**:
    - Не создаёт таблицу в базе данных.
    - Поля и аннотации наследуются дочерними классами.
    - Не поддерживает полиморфные запросы, так как не является сущностью.
- **Применение**: Используется для общих атрибутов, таких как `id`, `createdAt`, `updatedAt`.

## Почему нужно заменить BaseEntity на интерфейс?

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

## Создание класса AuditableEntity

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

# Стратегия TABLE_PER_CLASS

Стратегия `TABLE_PER_CLASS` мапит каждый класс иерархии на отдельную таблицу, содержащую все поля, включая унаследованные.

## Реализация стратегии TABLE_PER_CLASS

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

## Аннотация @Inheritance

- **Назначение**: Указывает стратегию наследования для иерархии сущностей.
- **Атрибут strategy**: `InheritanceType.TABLE_PER_CLASS` задаёт маппинг каждой сущности на отдельную таблицу.
- **Особенности**:
    - Не требует дискриминатора, так как таблицы полностью независимы.
    - Поддерживает полиморфные запросы, но с использованием `UNION`.

## Тестирование функциональности

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import jakarta.persistence.Persistence;

public class TablePerClassTest {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("testPU");
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();

        Car car = new Car();
        car.setBrand("Toyota");
        car.setSeats(5);
        em.persist(car);

        Truck truck = new Truck();
        truck.setBrand("Volvo");
        truck.setLoadCapacity(10.5);
        em.persist(truck);

        em.getTransaction().commit();

        // Полиморфный запрос
        var vehicles = em.createQuery("SELECT v FROM Vehicle v", Vehicle.class).getResultList();
        vehicles.forEach(v -> System.out.println(v.getBrand()));
        
        em.close();
        emf.close();
    }
}
```

**Результат**:

- Создаются записи в таблицах `cars` и `trucks`.
- Полиморфный запрос возвращает список всех `Vehicle` (`Car` и `Truck`), используя `UNION` для объединения данных.

**SQL-запрос**:

```sql
SELECT id, brand, seats, NULL as load_capacity FROM cars
UNION
SELECT id, brand, NULL as seats, load_capacity FROM trucks;
```

## Почему нельзя использовать IDENTITY в TABLE_PER_CLASS

- **Проблема**: Стратегия `IDENTITY` (автоинкремент в базе данных) создаёт уникальные идентификаторы в каждой таблице независимо. Это приводит к дублированию ID между таблицами (например, `Car` и `Truck` могут иметь одинаковый `id`), что нарушает уникальность в полиморфных запросах.
- **Решение**: Используйте `SEQUENCE` или `TABLE` для генерации глобально уникальных ID.
- **Причина**: Hibernate выполняет полиморфные запросы через `UNION`, требуя уникальности ID между всеми таблицами иерархии.
- **Пример ошибки**:

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Ошибка!
    private Long id;
}
```

- Это приведёт к дублированию ID в таблицах `cars` и `trucks`, что вызовет некорректные результаты при полиморфных запросах.

# Стратегия SINGLE_TABLE

Стратегия `SINGLE_TABLE` хранит все классы иерархии в одной таблице, используя столбец-дискриминатор для различения типов.

## Реализация стратегии SINGLE_TABLE

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;
import jakarta.persistence.Id;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.DiscriminatorColumn;
import jakarta.persistence.DiscriminatorType;

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
CREATE TABLE vehicles (
    id BIGSERIAL PRIMARY KEY,
    vehicle_type VARCHAR(255),
    brand VARCHAR(255),
    seats INTEGER,
    load_capacity DOUBLE PRECISION
);
```

**Работа**:

- Все сущности (`Car`, `Truck`) хранятся в таблице `vehicles`.
- Столбец `vehicle_type` указывает тип сущности (`CAR` или `TRUCK`).
- Неиспользуемые поля для конкретного типа остаются `NULL` (например, `load_capacity` для `Car`).

## Тестирование функционала

```java
public class SingleTableTest {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("testPU");
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();

        Car car = new Car();
        car.setBrand("Toyota");
        car.setSeats(5);
        em.persist(car);

        Truck truck = new Truck();
        truck.setBrand("Volvo");
        truck.setLoadCapacity(10.5);
        em.persist(truck);

        em.getTransaction().commit();

        // Полиморфный запрос
        var vehicles = em.createQuery("SELECT v FROM Vehicle v", Vehicle.class).getResultList();
        vehicles.forEach(v -> System.out.println(v.getBrand() + ", Type: " + v.getClass().getSimpleName()));
        
        em.close();
        emf.close();
    }
}
```

**Результат**:

- Записи в таблице `vehicles`:

```sql
INSERT INTO vehicles (vehicle_type, brand, seats) VALUES ('CAR', 'Toyota', 5);
INSERT INTO vehicles (vehicle_type, brand, load_capacity) VALUES ('TRUCK', 'Volvo', 10.5);
```

- Полиморфный запрос возвращает все `Vehicle`, автоматически определяя тип по `vehicle_type`.

# Стратегия JOINED

Стратегия `JOINED` хранит общие поля в таблице базового класса, а специфические — в таблицах дочерних классов.

## Реализация стратегии JOINED

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;
import jakarta.persistence.Id;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;

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
CREATE TABLE vehicles (
    id BIGSERIAL PRIMARY KEY,
    brand VARCHAR(255)
);

CREATE TABLE cars (
    id BIGINT PRIMARY KEY,
    seats INTEGER,
    FOREIGN KEY (id) REFERENCES vehicles(id)
);

CREATE TABLE trucks (
    id BIGINT PRIMARY KEY,
    load_capacity DOUBLE PRECISION,
    FOREIGN KEY (id) REFERENCES vehicles(id)
);
```

**Работа**:

- Таблица `vehicles` хранит общие поля (`id`, `brand`).
- Таблицы `cars` и `trucks` содержат специфические поля и ссылаются на `vehicles` через внешний ключ.
- Полиморфные запросы используют `JOIN` для объединения данных.

## Тестирование функционала

```java
public class JoinedTest {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("testPU");
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();

        Car car = new Car();
        car.setBrand("Toyota");
        car.setSeats(5);
        em.persist(car);

        Truck truck = new Truck();
        truck.setBrand("Volvo");
        truck.setLoadCapacity(10.5);
        em.persist(truck);

        em.getTransaction().commit();

        // Полиморфный запрос
        var vehicles = em.createQuery("SELECT v FROM Vehicle v", Vehicle.class).getResultList();
        vehicles.forEach(v -> System.out.println(v.getBrand() + ", Type: " + v.getClass().getSimpleName()));
        
        em.close();
        emf.close();
    }
}
```

**Результат**:

- Записи:

```sql
INSERT INTO vehicles (id, brand) VALUES (1, 'Toyota');
INSERT INTO cars (id, seats) VALUES (1, 5);
INSERT INTO vehicles (id, brand) VALUES (2, 'Volvo');
INSERT INTO trucks (id, load_capacity) VALUES (2, 10.5);
```

- Полиморфный запрос использует `JOIN` для получения данных из всех таблиц.

## Резюме

- **Преимущества JOINED**:
    - Нормализованная структура базы данных.
    - Экономия пространства, так как общие поля хранятся в одной таблице.
    - Подходит для сложных иерархий с большим количеством общих полей.
- **Недостатки**:
    - Полиморфные запросы требуют `JOIN`, что снижает производительность.
    - Сложнее в управлении из-за нескольких таблиц.
- **Применение**: Используется, когда важна нормализация и структура данных.

# Резюме по всем стратегиям наследования

|Стратегия|Описание|Преимущества|Недостатки|Применение|
|---|---|---|---|---|
|**MappedSuperclass**|Общие поля и маппинги наследуются, но не создают таблицу.|Простота, переиспользование кода, нет таблицы для базового класса.|Не поддерживает полиморфные запросы, ограничена наследованием маппингов.|Общие атрибуты (ID, аудит) для несвязанных сущностей.|
|**TABLE_PER_CLASS**|Каждая сущность мапится на отдельную таблицу со всеми полями.|Простая структура, независимые таблицы, быстрые запросы к одной сущности.|Дублирование общих полей, проблемы с `IDENTITY`, сложные полиморфные запросы (`UNION`).|Иерархии с независимыми сущностями.|
|**SINGLE_TABLE**|Все сущности в одной таблице с дискриминатором.|Быстрые запросы, простота управления, поддержка полиморфизма.|Много `NULL` для неиспользуемых полей, проблемы с масштабированием.|Простые иерархии с небольшим числом полей.|
|**JOINED**|Общие поля в таблице базового класса, специфические — в дочерних.|Нормализованная структура, экономия пространства, поддержка полиморфизма.|Сложные `JOIN`-запросы, снижение производительности.|Сложные иерархии с нормализацией данных.|

**Рекомендации**:

- Используйте `@MappedSuperclass` для общих атрибутов, если полиморфизм не нужен.
- Выбирайте `SINGLE_TABLE` для простых иерархий с высокой производительностью запросов.
- Используйте `TABLE_PER_CLASS` для независимых сущностей, но избегайте `IDENTITY`.
- Применяйте `JOINED` для нормализованных структур и сложных иерархий.
- Рассмотрите интерфейсы вместо базовых классов для большей гибкости.