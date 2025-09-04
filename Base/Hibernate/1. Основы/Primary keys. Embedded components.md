# Встроенные компоненты и первичные ключи в Hibernate

## Оглавление
1. [Введение в встроенные компоненты](#введение)
2. [Встроенные компоненты](#embedded-components)
    1. [Что такое встроенный компонент](#что-такое-компонент)
    2. [Аннотации @Embeddable и @Embedded](#embeddable-embedded)
    3. [Класс EmbeddedComponentType](#embedded-component-type)
    4. [Аннотация @AttributeOverride](#attribute-override)
3. [Практические примеры](#примеры)
    1. [Встроенный компонент с переопределением атрибутов](#пример-переопределения)
4. [Первичные ключи](#primary-keys)
    1. [Зачем нужны первичные ключи](#зачем-нужны-ключи)
    2. [Генерируемый первичный ключ](#generated-key)
    3. [Аннотация @GeneratedValue](#generated-value)
5. [Стратегии генерации ключей](#strategies)
    1. [IDENTITY strategy](#identity-strategy)
    2. [SEQUENCE strategy](#sequence-strategy)
    3. [Аннотация @SequenceGenerator](#sequence-generator)
    4. [TABLE strategy](#table-strategy)
6. [Лучшие практики](#практики)
7. [Вопросы для собеседования](#вопросы)

---

## <a name="введение"></a>Введение в встроенные компоненты

Встроенные компоненты в Hibernate позволяют объединить группу связанных полей в отдельный Java-класс, который можно переиспользовать в нескольких сущностях. Они представляют структурированные данные, не требующие собственной идентичности, и мапятся на столбцы таблицы родительской сущности.

---

## <a name="embedded-components"></a>Встроенные компоненты

### <a name="что-такое-компонент"></a>Что такое встроенный компонент

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

### <a name="embeddable-embedded"></a>Аннотации @Embeddable и @Embedded

- **@Embeddable**: Помечает Java-класс как встроенный компонент, который может быть включён в сущности. Этот класс должен быть сериализуемым (`Serializable`), если используется в составных ключах.
- **@Embedded**: Указывает, что поле сущности является встроенным компонентом. Поля компонента мапятся на столбцы той же таблицы, что и родительская сущность, если не переопределены.

**Работа**:

- Hibernate автоматически мапит поля класса, помеченного `@Embeddable`, на столбцы таблицы сущности, содержащей поле с аннотацией `@Embedded`.
- Имена столбцов по умолчанию берутся из названий полей компонента, но могут быть переопределены.

**Использование**:

- Используется для структурирования данных, например, для адресов, координат, или других составных данных.
- Упрощает поддержку кода, так как изменения в структуре компонента автоматически применяются ко всем сущностям, использующим его.

### <a name="embedded-component-type"></a>Класс EmbeddedComponentType

`EmbeddedComponentType` — это внутренний класс Hibernate, представляющий метаданные встроенного компонента. Он отвечает за:

- Маппинг полей компонента на столбцы таблицы.
- Обработку аннотаций, таких как `@AttributeOverride`, для настройки маппинга.
- Управление сериализацией и десериализацией компонента при взаимодействии с базой данных.

**Работа**:

- Hibernate использует `EmbeddedComponentType` для анализа структуры компонента и генерации SQL-запросов для чтения и записи данных.
- Этот класс прозрачен для разработчика, но понимание его роли помогает при отладке сложных сценариев маппинга, например, при конфликтах имён столбцов.

**Использование**:

- Разработчики редко взаимодействуют с `EmbeddedComponentType` напрямую, но знание о нём полезно при использовании инструментов Hibernate, таких как `Metadata` или `SessionFactory`, для анализа маппинга.

### <a name="attribute-override"></a>Аннотация @AttributeOverride

Аннотация `@AttributeOverride` позволяет переопределять маппинг полей встроенного компонента для конкретной сущности. Это полезно, когда один и тот же компонент используется в разных сущностях, но с разными именами столбцов или другими настройками.

**Работа**:

- Указывает новое имя столбца или другие атрибуты (например, длину или тип) для поля компонента.
- Может применяться многократно через `@AttributeOverrides` для переопределения нескольких полей.

**Использование**:

- Когда нужно адаптировать встроенный компонент под разные таблицы (например, разные имена столбцов для адреса в таблицах `users` и `companies`).
- Для изменения типов данных или ограничений (например, `nullable` или `length`) для конкретной сущности.

---

## <a name="примеры"></a>Практические примеры

### <a name="пример-переопределения"></a>Встроенный компонент с переопределением атрибутов

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

---

## <a name="primary-keys"></a>Первичные ключи

Первичные ключи — это уникальные идентификаторы записей в таблице базы данных, обеспечивающие однозначную идентификацию каждой сущности.

### <a name="зачем-нужны-ключи"></a>Зачем нужны первичные ключи

- **Уникальность**: Гарантируют, что каждая запись в таблице уникальна.
- **Поиск и обновление**: Используются для быстрого поиска, обновления и удаления сущностей через `EntityManager.find()` или JPQL-запросы.
- **Связи**: Служат основой для внешних ключей в связях между таблицами (например, `@ManyToOne`, `@OneToMany`).
- **Целостность данных**: Обеспечивают согласованность данных в реляционной базе.

**Пример**:

- В таблице `users` первичный ключ `id` однозначно идентифицирует каждого пользователя.
- Внешний ключ `company_id` в таблице `users` ссылается на первичный ключ `id` в таблице `companies`.

### <a name="generated-key"></a>Генерируемый первичный ключ (PostgreSQL SERIAL)

В PostgreSQL тип `SERIAL` (или `BIGSERIAL`) используется для автоматической генерации уникальных значений первичного ключа. Это аналог автоинкремента в других СУБД, таких как MySQL.

**Работа**:

- При вставке новой записи PostgreSQL автоматически увеличивает значение `SERIAL` и присваивает его столбцу.
- Hibernate поддерживает это через аннотацию `@GeneratedValue` с различными стратегиями.

**Использование**:

- Подходит для простых сущностей с одним уникальным идентификатором.
- Не требует дополнительных настроек в базе данных, так как `SERIAL` встроен в PostgreSQL.

### <a name="generated-value"></a>Аннотация @GeneratedValue

Аннотация `@GeneratedValue` указывает, что значение первичного ключа генерируется автоматически. Атрибут `strategy` определяет способ генерации:

- `GenerationType.AUTO`: Hibernate выбирает стратегию в зависимости от СУБД (обычно `SEQUENCE` для PostgreSQL).
- `GenerationType.IDENTITY`: Использует автоинкрементный столбец (`SERIAL`).
- `GenerationType.SEQUENCE`: Использует последовательность базы данных.
- `GenerationType.TABLE`: Использует отдельную таблицу для генерации ключей.

**Работа**:

- Hibernate взаимодействует с базой данных для получения следующего значения ключа перед или во время вставки записи.
- Выбор стратегии влияет на производительность и поведение приложения.

---

## <a name="strategies"></a>Стратегии генерации ключей

*IDENTITY strategy*

Стратегия `IDENTITY` использует автоинкрементный столбец базы данных, такой как `SERIAL` в PostgreSQL.

*SEQUENCE strategy*

Стратегия `SEQUENCE` использует последовательности базы данных (например, `CREATE SEQUENCE` в PostgreSQL) для генерации ключей.

*TABLE strategy*

Стратегия `TABLE` использует отдельную таблицу в базе данных для хранения и генерации ключей.

### <a name="identity-strategy"></a>IDENTITY strategy

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

### <a name="sequence-strategy"></a>SEQUENCE strategy

Стратегия `SEQUENCE` использует последовательности базы данных (например, `CREATE SEQUENCE` в PostgreSQL) для генерации ключей.

**Работа**:

- Hibernate запрашивает следующее значение последовательности отдельным запросом перед выполнением `INSERT`.
- Позволяет гибко настраивать генерацию через параметры последовательности (начальное значение, шаг).

**Использование**:

- Подходит для PostgreSQL и Oracle, где последовательности широко используются.
- Позволяет предварительно получить ключ, что полезно для сложных операций.

### <a name="sequence-generator"></a>Аннотация @SequenceGenerator

Аннотация `@SequenceGenerator` определяет параметры последовательности для `SEQUENCE` strategy:

- `name`: Имя генератора, на которое ссылается `@GeneratedValue`.
- `sequenceName`: Имя последовательности в базе данных.
- `allocationSize`: Размер пула значений, запрашиваемых за один раз (по умолчанию 50).
- `initialValue`: Начальное значение последовательности.

**Работа**:

- Hibernate создаёт или использует существующую последовательность в базе данных.
- Запрашивает значения из последовательности для генерации ключей.

**Пример**:

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
    @SequenceGenerator(name = "user_seq", sequenceName = "user_sequence", allocationSize = 1)
    private Long id;
    private String name;

    // Геттеры и сеттеры
}
```

**SQL**:

```sql
CREATE SEQUENCE user_sequence START 1;
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

### <a name="table-strategy"></a>TABLE strategy

Стратегия `TABLE` использует отдельную таблицу в базе данных для хранения и генерации ключей.

**Работа**:

- Hibernate создаёт таблицу (например, `id_gen`) с двумя столбцами: один для имени генератора, другой для текущего значения ключа.
- При необходимости получения нового ключа Hibernate обновляет значение в этой таблице.

**Использование**:

- Подходит для баз данных без поддержки последовательностей или автоинкремента.
- Обеспечивает переносимость между разными СУБД.

**Проблемы**:

- Менее эффективна, чем `SEQUENCE` или `IDENTITY`.
- Требует дополнительной таблицы в базе данных.
- Может стать узким местом при высокой нагрузке.

**Пример**:

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "user_table_gen")
    @TableGenerator(name = "user_table_gen", table = "id_generator", 
                   pkColumnName = "gen_name", valueColumnName = "gen_value",
                   pkColumnValue = "user_id", allocationSize = 1)
    private Long id;
    private String name;

    // Геттеры и сеттеры
}
```

**SQL**:

```sql
CREATE TABLE id_generator (
    gen_name VARCHAR(255) PRIMARY KEY,
    gen_value BIGINT
);
INSERT INTO id_generator (gen_name, gen_value) VALUES ('user_id', 1);

CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

---

## <a name="практики"></a>Лучшие практики

### Выбор стратегии генерации ключей

- **IDENTITY**: Используйте для простых приложений с PostgreSQL или MySQL.
- **SEQUENCE**: Используйте для высоконагруженных приложений с PostgreSQL или Oracle.
- **TABLE**: Используйте для переносимых приложений или баз данных без поддержки последовательностей.

### Встроенные компоненты

- Используйте для логической группировки связанных полей.
- Переопределяйте имена столбцов для избежания конфликтов.
- Не используйте для данных, требующих собственного жизненного цикла.

### Производительность

- Настройте `allocationSize` для `SEQUENCE` в зависимости от нагрузки.
- Избегайте `TABLE` strategy в высоконагруженных системах.
- Используйте `IDENTITY` для простых случаев с небольшим количеством вставок.

---

## <a name="вопросы"></a>Вопросы для собеседования

1. Что такое встроенные компоненты в Hibernate и для чего они используются?
2. Какие аннотации используются для работы с встроенными компонентами?
3. Как работает аннотация @AttributeOverride?
4. Какие ограничения у встроенных компонентов?
5. Какие стратегии генерации первичных ключей поддерживает Hibernate?
6. В чём разница между IDENTITY и SEQUENCE стратегиями?
7. Как работает TABLE strategy для генерации ключей?
8. Когда стоит использовать @SequenceGenerator?
9. Какие проблемы могут возникнуть с встроенными компонентами?
10. Как выбрать оптимальную стратегию генерации ключей?

---

**Рекомендуемые материалы:**
- [Hibernate Embedded Components](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#embeddable)
- [JPA Primary Keys](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#a13346)
- [Vlad Mihalcea: Hibernate Primary Keys](https://vladmihalcea.com/hibernate-primary-keys/)