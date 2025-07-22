# Встроенные компоненты

Встроенные компоненты в Hibernate позволяют объединить группу связанных полей в отдельный Java-класс, который можно переиспользовать в нескольких сущностях. Они представляют структурированные данные, не требующие собственной идентичности, и мапятся на столбцы таблицы родительской сущности.

## Что такое встроенный компонент?

Встроенный компонент — это Java-класс, который:

- Содержит набор полей, мапящихся на столбцы таблицы родительской сущности.
- Не имеет собственного первичного ключа и жизненного цикла, так как является частью сущности.
- Используется для логической группировки данных, таких как адрес, контактная информация или параметры конфигурации, чтобы избежать дублирования кода и повысить читаемость.

**Пример использования**:

- Если у нескольких сущностей есть поле "адрес" (улица, город, почтовый индекс), вместо дублирования этих полей в каждой сущности можно создать класс `Address` и использовать его как встроенный компонент.
- Это улучшает поддержку кода, упрощает рефакторинг и делает модель данных более организованной.

**Ограничения**:

- Встроенные компоненты не могут быть лениво загружаемыми, так как они являются частью сущности.
- Они не поддерживают каскадирование операций, так как не имеют собственного жизненного цикла.

## Аннотации @Embeddable и @Embedded

- **@Embeddable**: Помечает Java-класс как встроенный компонент, который может быть включён в сущности. Этот класс должен быть сериализуемым (`Serializable`), если используется в составных ключах.
- **@Embedded**: Указывает, что поле сущности является встроенным компонентом. Поля компонента мапятся на столбцы той же таблицы, что и родительская сущность, если не переопределены.

**Работа**:

- Hibernate автоматически мапит поля класса, помеченного `@Embeddable`, на столбцы таблицы сущности, содержащей поле с аннотацией `@Embedded`.
- Имена столбцов по умолчанию берутся из названий полей компонента, но могут быть переопределены.

**Использование**:

- Используется для структурирования данных, например, для адресов, координат, или других составных данных.
- Упрощает поддержку кода, так как изменения в структуре компонента автоматически применяются ко всем сущностям, использующим его.

## Класс EmbeddedComponentType

`EmbeddedComponentType` — это внутренний класс Hibernate, представляющий метаданные встроенного компонента. Он отвечает за:

- Маппинг полей компонента на столбцы таблицы.
- Обработку аннотаций, таких как `@AttributeOverride`, для настройки маппинга.
- Управление сериализацией и десериализацией компонента при взаимодействии с базой данных.

**Работа**:

- Hibernate использует `EmbeddedComponentType` для анализа структуры компонента и генерации SQL-запросов для чтения и записи данных.
- Этот класс прозрачен для разработчика, но понимание его роли помогает при отладке сложных сценариев маппинга, например, при конфликтах имён столбцов.

**Использование**:

- Разработчики редко взаимодействуют с `EmbeddedComponentType` напрямую, но знание о нём полезно при использовании инструментов Hibernate, таких как `Metadata` или `SessionFactory`, для анализа маппинга.

## Аннотация @AttributeOverride

Аннотация `@AttributeOverride` позволяет переопределять маппинг полей встроенного компонента для конкретной сущности. Это полезно, когда один и тот же компонент используется в разных сущностях, но с разными именами столбцов или другими настройками.

**Работа**:

- Указывает новое имя столбца или другие атрибуты (например, длину или тип) для поля компонента.
- Может применяться многократно через `@AttributeOverrides` для переопределения нескольких полей.

**Использование**:

- Когда нужно адаптировать встроенный компонент под разные таблицы (например, разные имена столбцов для адреса в таблицах `users` и `companies`).
- Для изменения типов данных или ограничений (например, `nullable` или `length`) для конкретной сущности.

## Пример: Встроенный компонент с переопределением атрибутов

Создадим встроенный компонент `Address` и используем его в сущности `User` с переопределением имён столбцов.

```java
import javax.persistence.Embeddable;

@Embeddable
public class Address {
    private String street;
    private String city;
    private String postalCode;

    // Геттеры и сеттеры
    public String getStreet() { return street; }
    public void setStreet(String street) { this.street = street; }
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
    public String getPostalCode() { return postalCode; }
    public void setPostalCode(String postalCode) { this.postalCode = postalCode; }
}
```

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Embedded;
import javax.persistence.AttributeOverride;
import javax.persistence.AttributeOverrides;
import javax.persistence.Column;

@Entity
public class User {
    @Id
    private Long id;
    private String name;
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "home_street")),
        @AttributeOverride(name = "city", column = @Column(name = "home_city")),
        @AttributeOverride(name = "postalCode", column = @Column(name = "home_postal_code"))
    })
    private Address address;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Address getAddress() { return address; }
    public void setAddress(Address address) { this.address = address; }
}
```

**SQL-таблица**:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    home_street VARCHAR(255),
    home_city VARCHAR(255),
    home_postal_code VARCHAR(10)
);
```

**Работа**:

- Класс `Address` помечен как `@Embeddable`, что позволяет использовать его как компонент.
- В сущности `User` поле `address` помечено как `@Embedded`, а аннотация `@AttributeOverrides` переопределяет имена столбцов (`street` → `home_street`, `city` → `home_city`, `postalCode` → `home_postal_code`).
- Hibernate мапит поля `Address` на столбцы таблицы `users` с указанными именами.

**Использование**:

- Создание пользователя с адресом:

```java
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();

User user = new User();
user.setId(1L);
user.setName("John");
Address address = new Address();
address.setStreet("123 Main St");
address.setCity("New York");
address.setPostalCode("10001");
user.setAddress(address);
em.persist(user);

em.getTransaction().commit();
em.close();
```

**Проблемы**:

- Если не переопределить имена столбцов, может возникнуть конфликт, если другой встроенный компонент использует те же имена полей (например, `street` в разных таблицах).
- Встроенные компоненты не поддерживают ленивую загрузку, так как являются частью сущности.

# Первичные ключи

Первичные ключи — это уникальные идентификаторы записей в таблице базы данных, обеспечивающие однозначную идентификацию каждой сущности.

## Зачем нужны?

- **Уникальность**: Гарантируют, что каждая запись в таблице уникальна.
- **Поиск и обновление**: Используются для быстрого поиска, обновления и удаления сущностей через `EntityManager.find()` или JPQL-запросы.
- **Связи**: Служат основой для внешних ключей в связях между таблицами (например, `@ManyToOne`, `@OneToMany`).
- **Целостность данных**: Обеспечивают согласованность данных в реляционной базе.

**Пример**:

- В таблице `users` первичный ключ `id` однозначно идентифицирует каждого пользователя.
- Внешний ключ `company_id` в таблице `users` ссылается на первичный ключ `id` в таблице `companies`.

## Генерируемый первичный ключ (PostgreSQL SERIAL)

В PostgreSQL тип `SERIAL` (или `BIGSERIAL`) используется для автоматической генерации уникальных значений первичного ключа. Это аналог автоинкремента в других СУБД, таких как MySQL.

**Работа**:

- При вставке новой записи PostgreSQL автоматически увеличивает значение `SERIAL` и присваивает его столбцу.
- Hibernate поддерживает это через аннотацию `@GeneratedValue` с различными стратегиями.

**Использование**:

- Подходит для простых сущностей с одним уникальным идентификатором.
- Не требует дополнительных настроек в базе данных, так как `SERIAL` встроен в PostgreSQL.

## Аннотация @GeneratedValue

Аннотация `@GeneratedValue` указывает, что значение первичного ключа генерируется автоматически. Атрибут `strategy` определяет способ генерации:

- `GenerationType.AUTO`: Hibernate выбирает стратегию в зависимости от СУБД (обычно `SEQUENCE` для PostgreSQL).
- `GenerationType.IDENTITY`: Использует автоинкрементный столбец (`SERIAL`).
- `GenerationType.SEQUENCE`: Использует последовательность базы данных.
- `GenerationType.TABLE`: Использует отдельную таблицу для генерации ключей.

**Работа**:

- Hibernate взаимодействует с базой данных для получения следующего значения ключа перед или во время вставки записи.
- Выбор стратегии влияет на производительность и поведение приложения.

## IDENTITY strategy

Стратегия `IDENTITY` использует автоинкрементный столбец базы данных, такой как `SERIAL` в PostgreSQL.

**Работа**:

- Hibernate полагается на базу данных для генерации значения при выполнении `INSERT`.
- После вставки Hibernate извлекает сгенерированный ключ (например, через `RETURNING id` в PostgreSQL).

**Использование**:

- Проста в настройке и подходит для баз данных с поддержкой автоинкремента.
- Не требует дополнительных объектов в базе (в отличие от `SEQUENCE` или `TABLE`).

**Проблемы**:

- Может быть менее эффективной в пакетных вставках, так как требует немедленного выполнения `INSERT` для получения ключа.
- Не поддерживается в некоторых СУБД, где автоинкремент отсутствует.

**Пример**:

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    // Геттеры и сеттеры
}
```

**SQL**:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

## SEQUENCE strategy

Стратегия `SEQUENCE` использует последовательности базы данных (например, `CREATE SEQUENCE` в PostgreSQL) для генерации ключей.

**Работа**:

- Hibernate запрашивает следующее значение последовательности отдельным запросом перед выполнением `INSERT`.
- Позволяет гибко настраивать генерацию через параметры последовательности (начальное значение, шаг).

**Использование**:

- Подходит для PostgreSQL и Oracle, где последовательности широко используются.
- Позволяет предварительно получить ключ, что полезно для сложных операций.

## Аннотация @SequenceGenerator

Аннотация `@SequenceGenerator` определяет параметры последовательности для `SEQUENCE` strategy:

- `name`: Имя генератора, на которое ссылается `@GeneratedValue`.
- `sequenceName`: Имя последовательности в базе данных.
- `allocationSize`: Размер пула значений, запрашиваемых за один раз (по умолчанию 50).
- `initialValue`: Начальное значение последовательности.

**Работа**:

- Hibernate создаёт или использует существующую последовательность в базе данных.
- Запрашивает значения из последовательности для генерации ключей.

## Не происходит сразу INSERT в случае SEQUENCE strategy

В отличие от `IDENTITY`, где `INSERT` выполняется немедленно для получения ключа, при использовании `SEQUENCE` Hibernate сначала запрашивает значение из последовательности, а затем использует его в `INSERT`. Это:

- Снижает количество немедленных вставок.
- Улучшает производительность при пакетной обработке.

**Проблемы**:

- Требует создания последовательности в базе данных.
- Дополнительный запрос для получения значения может увеличить накладные расходы.

## TABLE strategy

Стратегия `TABLE` использует отдельную таблицу в базе данных для хранения и генерации ключей.

**Работа**:

- Hibernate создаёт таблицу (например, `id_gen`) с двумя столбцами: один для имени генератора, другой для текущего значения ключа.
- При необходимости нового ключа Hibernate обновляет таблицу и возвращает значение.

**Использование**:

- Переносима между СУБД, так как не зависит от встроенных механизмов автоинкремента или последовательностей.
- Подходит для устаревших систем или СУБД без поддержки `SERIAL` или `SEQUENCE`.

**Проблемы**:

- Менее эффективна из-за необходимости дополнительных запросов для обновления таблицы.
- Может вызывать проблемы с конкурентностью при высоких нагрузках.

## Аннотация @TableGenerator

Аннотация `@TableGenerator` определяет параметры таблицы для генерации ключей:

- `name`: Имя генератора.
- `table`: Имя таблицы для хранения ключей.
- `pkColumnName`: Столбец, содержащий имя генератора.
- `valueColumnName`: Столбец, содержащий значение ключа.
- `allocationSize`: Размер пула значений.

## Пример: Генерируемый первичный ключ

### SEQUENCE strategy

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.SequenceGenerator;

@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "emp_seq")
    @SequenceGenerator(name = "emp_seq", sequenceName = "employee_sequence", allocationSize = 1)
    private Long id;
    private String name;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

**SQL**:

```sql
CREATE SEQUENCE employee_sequence START WITH 1 INCREMENT BY 1;
CREATE TABLE employees (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

**Работа**:

- Hibernate запрашивает следующее значение из последовательности `employee_sequence` перед вставкой.
- `allocationSize = 1` гарантирует, что каждое значение запрашивается отдельно, минимизируя риск пропуска значений.

**Использование**:

```java
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();

Employee employee = new Employee();
employee.setName("Alice");
em.persist(employee); // Hibernate запрашивает значение из последовательности

em.getTransaction().commit();
em.close();
```

### TABLE strategy

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.TableGenerator;

@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "dept_gen")
    @TableGenerator(name = "dept_gen", table = "id_gen", pkColumnName = "gen_name", valueColumnName = "gen_value", allocationSize = 1)
    private Long id;
    private String name;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

**SQL**:

```sql
CREATE TABLE id_gen (
    gen_name VARCHAR(255) PRIMARY KEY,
    gen_value BIGINT NOT NULL
);
CREATE TABLE departments (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

**Работа**:

- Hibernate создаёт запись в таблице `id_gen` с именем генератора `dept_gen` и обновляет `gen_value` для генерации ключей.
- При вставке новой записи в `departments` используется следующее значение из `id_gen`.

**Использование**:

```java
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();

Department department = new Department();
department.setName("IT");
em.persist(department); // Hibernate обновляет id_gen и использует значение

em.getTransaction().commit();
em.close();
```

**Проблемы**:

- `TABLE` strategy требует дополнительной таблицы, что увеличивает сложность схемы.
- Может быть медленнее из-за необходимости блокировки таблицы `id_gen` при конкурентных операциях.

# EmbeddedId (Составной первичный ключ)

Составной первичный ключ используется, когда идентификация сущности требует нескольких полей. В Hibernate это реализуется через аннотацию `@EmbeddedId`.

## Зачем нужен?

- **Сложная идентификация**: Когда одного поля недостаточно для уникальной идентификации (например, заказ идентифицируется комбинацией ID клиента и номера заказа).
- **Инкапсуляция**: Объединяет поля ключа в отдельный класс для удобства и переиспользования.
- **Поддержка связей**: Используется в сложных связях, где ключ включает внешние ключи.

**Пример сценария**:

- Таблица заказов, где каждая запись уникальна по комбинации `customerId` и `orderNumber`.

## Аннотация @EmbeddedId

`@EmbeddedId` указывает, что поле сущности является составным первичным ключом, представленным классом, помеченным как `@Embeddable`. Этот класс должен:

- Реализовать интерфейс `Serializable`.
- Иметь корректные реализации `equals()` и `hashCode()`.

## Создание и получение сущности по EmbeddedId

- **Создание**: Создайте экземпляр класса составного ключа, установите его значения и присвойте сущности перед сохранением.
- **Получение**: Используйте `EntityManager.find()`, передавая экземпляр класса составного ключа.

## Пример: Составной первичный ключ с EmbeddedId

```java
import javax.persistence.Embeddable;
import java.io.Serializable;
import java.util.Objects;

@Embeddable
public class OrderId implements Serializable {
    private Long customerId;
    private Long orderNumber;

    // Геттеры, сеттеры, equals и hashCode
    public Long getCustomerId() { return customerId; }
    public void setCustomerId(Long customerId) { this.customerId = customerId; }
    public Long getOrderNumber() { return orderNumber; }
    public void setOrderNumber(Long orderNumber) { this.orderNumber = orderNumber; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        OrderId orderId = (OrderId) o;
        return Objects.equals(customerId, orderId.customerId) &&
               Objects.equals(orderNumber, orderId.orderNumber);
    }

    @Override
    public int hashCode() {
        return Objects.hash(customerId, orderNumber);
    }
}
```

```java
import javax.persistence.Entity;
import javax.persistence.EmbeddedId;

@Entity
public class Order {
    @EmbeddedId
    private OrderId id;
    private String product;

    // Геттеры и сеттеры
    public OrderId getId() { return id; }
    public void setId(OrderId id) { this.id = id; }
    public String getProduct() { return product; }
    public void setProduct(String product) { this.product = product; }
}
```

**SQL**:

```sql
CREATE TABLE orders (
    customer_id BIGINT NOT NULL,
    order_number BIGINT NOT NULL,
    product VARCHAR(255),
    PRIMARY KEY (customer_id, order_number)
);
```

**Работа**:

- Класс `OrderId` инкапсулирует составной ключ (`customerId`, `orderNumber`).
- Аннотация `@EmbeddedId` указывает, что `id` является первичным ключом сущности `Order`.
- Hibernate мапит поля `customerId` и `orderNumber` на столбцы `customer_id` и `order_number` в таблице `orders`.

**Использование**:

```java
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();

OrderId orderId = new OrderId();
orderId.setCustomerId(1L);
orderId.setOrderNumber(1001L);

Order order = new Order();
order.setId(orderId);
order.setProduct("Laptop");
em.persist(order);

em.getTransaction().commit();

// Получение сущности
Order foundOrder = em.find(Order.class, orderId);
System.out.println("Order: " + foundOrder.getProduct());
em.close();
```

**Проблемы**:

- Составные ключи сложнее в использовании, так как требуют создания отдельного класса и корректной реализации `equals()` и `hashCode()`.
- Могут усложнить JPQL-запросы, так как нужно обращаться к полям ключа через точку (например, `o.id.customerId`).

# Другие основные аннотации

Hibernate и JPA предоставляют дополнительные аннотации для тонкой настройки маппинга полей и свойств.

## Аннотация @Access

`@Access` определяет, как Hibernate мапит поля или свойства сущности:

- `AccessType.FIELD`: Маппинг на основе полей (аннотации ставятся на поля).
- `AccessType.PROPERTY`: Маппинг на основе геттеров (аннотации ставятся на геттеры).

**Работа**:

- По умолчанию Hibernate определяет тип доступа по расположению аннотаций (`@Id`, `@Column` и т.д.).
- `@Access` позволяет явно задать тип доступа, если требуется смешивание полей и свойств.

**Использование**:

- Используется, когда нужно мапить некоторые поля через геттеры, а другие — напрямую.
- Полезно для устаревших классов, где геттеры содержат логику.

**Пример**:

```java
@Entity
@Access(AccessType.PROPERTY)
public class Employee {
    private Long id;
    private String name;

    @Id
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    @Column(name = "full_name")
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

## Аннотация @Transient

`@Transient` помечает поле или свойство как неподлежащее маппингу на базу данных.

**Работа**:

- Поле, помеченное `@Transient`, игнорируется Hibernate при генерации SQL-запросов.
- Используется для временных или вычисляемых данных, которые не нужно хранить в базе.

**Использование**:

- Для хранения промежуточных результатов или кэшированных данных.
- Для полей, используемых только в бизнес-логике.

## Аннотация @Temporal

`@Temporal` указывает тип данных для полей `java.util.Date` или `java.util.Calendar`:

- `TemporalType.DATE`: Хранит только дату (например, `2025-07-21`).
- `TemporalType.TIME`: Хранит только время (например, `14:30:00`).
- `TemporalType.TIMESTAMP`: Хранит дату и время (например, `2025-07-21 14:30:00`).

**Работа**:

- Hibernate использует соответствующий SQL-тип (`DATE`, `TIME`, `TIMESTAMP`) для маппинга.

**Использование**:

- Необходима для старых API (`Date`, `Calendar`). Для современных `LocalDate`, `LocalTime`, `LocalDateTime` используется `@Column` с соответствующими типами.

## Аннотация @ColumnTransformer

`@ColumnTransformer` задаёт SQL-выражения для преобразования данных при записи или чтении из базы данных.

**Работа**:

- `read`: SQL-выражение для преобразования данных при чтении (например, `UPPER(name)`).
- `write`: SQL-выражение для преобразования данных при записи (например, `LOWER(?)`).

**Использование**:

- Для шифрования/дешифрования данных.
- Для форматирования данных, таких как преобразование регистра или вычисление значений.

## Аннотация @Formula

`@Formula` определяет SQL-выражение для вычисления значения поля вместо маппинга на столбец.

**Работа**:

- Hibernate выполняет указанное SQL-выражение при загрузке сущности.
- Значение не сохраняется в базе, а вычисляется динамически.

**Использование**:

- Для вычисляемых полей, таких как количество связанных записей или агрегированные данные.

## Пример: Использование основных аннотаций

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;
import javax.persistence.Column;
import javax.persistence.Transient;
import javax.persistence.Temporal;
import javax.persistence.TemporalType;
import javax.persistence.ColumnTransformer;
import javax.persistence.Formula;
import java.util.Date;

@Entity
@Table(name = "employees")
public class Employee {
    @Id
    private Long id;

    @Column(name = "full_name")
    @ColumnTransformer(read = "UPPER(full_name)", write = "LOWER(?)")
    private String name;

    @Temporal(TemporalType.DATE)
    private Date birthDate;

    @Transient
    private String temporaryData;

    @Formula("(SELECT COUNT(*) FROM orders o WHERE o.employee_id = id)")
    private int orderCount;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Date getBirthDate() { return birthDate; }
    public void setBirthDate(Date birthDate) { this.birthDate = birthDate; }
    public String getTemporaryData() { return temporaryData; }
    public void setTemporaryData(String temporaryData) { this.temporaryData = temporaryData; }
    public int getOrderCount() { return orderCount; }
    public void setOrderCount(int orderCount) { this.orderCount = orderCount; }
}
```

**SQL**:

```sql
CREATE TABLE employees (
    id BIGINT PRIMARY KEY,
    full_name VARCHAR(255),
    birth_date DATE
);
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    employee_id BIGINT,
    FOREIGN KEY (employee_id) REFERENCES employees(id)
);
```

**Работа**:

- `@ColumnTransformer`: Преобразует `full_name` в верхний регистр при чтении и нижний при записи.
- `@Temporal(TemporalType.DATE)`: Мапит `birthDate` как SQL `DATE`.
- `@Transient`: Исключает `temporaryData` из маппинга.
- `@Formula`: Вычисляет `orderCount` как количество заказов, связанных с сотрудником.

**Использование**:

```java
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();

Employee employee = new Employee();
employee.setId(1L);
employee.setName("Alice");
employee.setBirthDate(new Date());
employee.setTemporaryData("temp");
em.persist(employee);

em.getTransaction().commit();

Employee found = em.find(Employee.class, 1L);
System.out.println("Name: " + found.getName()); // ALICE (в верхнем регистре)
System.out.println("Order count: " + found.getOrderCount());
em.close();
```

**Проблемы**:

- `@ColumnTransformer` зависит от возможностей СУБД, что может снизить переносимость.
- `@Formula` увеличивает сложность запросов, так как требует выполнения подзапросов.

# Итог

- **Встроенные компоненты**:
    - Позволяют группировать связанные поля в переиспользуемые классы с помощью `@Embeddable` и `@Embedded`.
    - Поддерживают настройку маппинга через `@AttributeOverride` для адаптации к разным таблицам.
    - Упрощают поддержку кода, но не поддерживают ленивую загрузку или каскадирование.
- **Первичные ключи**:
    - Обеспечивают уникальную идентификацию с помощью `@Id` и `@GeneratedValue`.
    - Поддерживают стратегии `IDENTITY`, `SEQUENCE` и `TABLE`, каждая с плюсами и минусами.
    - `SEQUENCE` и `TABLE` обеспечивают гибкость, но требуют дополнительных настроек.
- **EmbeddedId**:
    - Используется для составных ключей, инкапсулированных в отдельный класс.
    - Требует реализации `Serializable`, `equals()` и `hashCode()`.
    - Усложняет запросы, но подходит для сложных сценариев идентификации.
- **Основные аннотации**:
    - `@Access`: Управляет типом доступа (поля или свойства).
    - `@Transient`: Исключает поля из маппинга.
    - `@Temporal`: Настраивает маппинг дат.
    - `@ColumnTransformer`: Добавляет кастомные SQL-преобразования.
    - `@Formula`: Вычисляет значения динамически через SQL.

Эти механизмы обеспечивают гибкое моделирование идентичности сущностей и структурированных данных в Hibernate, упрощая взаимодействие между Java-объектами и реляционной базой данных. Правильный выбор стратегий и аннотаций позволяет оптимизировать производительность и поддерживаемость приложения.