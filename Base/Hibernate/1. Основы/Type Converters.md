# Type Converters в Hibernate: преобразование типов данных

## Оглавление
1. [Введение в Type Converters](#введение)
2. [Basic Types](#basic-types)
    1. [Type Descriptors](#type-descriptors)
    2. [Enum User Type](#enum-type)
3. [Custom Attribute Converter](#attribute-converter)
    1. [Интерфейс AttributeConverter](#attribute-converter-interface)
    2. [Аннотация @Convert](#convert-annotation)
    3. [Auto Apply AttributeConverter](#auto-apply)
4. [Custom User Type](#user-type)
    1. [JSON Type](#json-type)
    2. [Библиотека hibernate-types](#hibernate-types)
    3. [Аннотация @Type](#type-annotation)
    4. [Аннотация @TypeDef](#typedef-annotation)
5. [Практические примеры](#примеры)
    1. [Конвертер для Enum](#пример-enum)
    2. [Конвертер для JSON](#пример-json)
    3. [UserType для сложных объектов](#пример-usertype)
6. [Лучшие практики](#практики)
7. [Вопросы для собеседования](#вопросы)

---

## <a name="введение"></a>Введение в Type Converters

**Type Converters в Hibernate** — это механизмы для преобразования данных между Java-объектами и типами базы данных. Они позволяют адаптировать нестандартные типы данных или специфические форматы для хранения в базе и обратно, упрощая работу с данными. Hibernate предоставляет встроенную систему типов для стандартных Java-типов (например, `String`, `Integer`, `Date`) и поддерживает расширение через **Custom Attribute Converters** и **Custom User Types**.

### Зачем нужны Type Converters?

- **Преобразование данных**: Обеспечивают маппинг между Java-объектами и SQL-типами, поддерживая нестандартные типы (например, перечисления, JSON, сложные объекты).
- **Гибкость**: Позволяют настраивать форматы данных и сложные маппинги, такие как сериализация в JSON или маппинг объекта на несколько столбцов.
- **Прозрачность**: Скрывают детали преобразования, упрощая взаимодействие между Java-кодом и базой данных.

---

## <a name="basic-types"></a>Basic Types

Hibernate предоставляет встроенную систему типов, которая автоматически сопоставляет стандартные Java-типы (например, `String`, `Integer`, `Date`) с соответствующими SQL-типами (например, `VARCHAR`, `INTEGER`, `TIMESTAMP`). Эти типы называются **Basic Types**.

### <a name="type-descriptors"></a>Type Descriptors

- Внутренние компоненты Hibernate, описывающие, как Java-тип мапится на SQL-тип.
- Управляют преобразованием данных и их сериализацией.
- Прозрачны для разработчика, но могут быть переопределены через кастомные типы.

### <a name="enum-type"></a>Enum User Type

- Перечисления (enum) в Java по умолчанию мапятся на строку (`VARCHAR`) или число (`INTEGER`) в базе данных.
- Hibernate автоматически поддерживает маппинг enum через `@Enumerated(EnumType.STRING)` или `@Enumerated(EnumType.ORDINAL)`.
- Для более сложных случаев (например, кастомное представление enum) используется `AttributeConverter` или `UserType`.

**Пример: Маппинг Enum с помощью @Enumerated**

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Enumerated;
import javax.persistence.EnumType;

@Entity
public class User {
    @Id
    private Long id;

    @Enumerated(EnumType.STRING)
    private UserStatus status;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public UserStatus getStatus() { return status; }
    public void setStatus(UserStatus status) { this.status = status; }
}

public enum UserStatus {
    ACTIVE, INACTIVE
}
```

В этом примере `UserStatus` мапится на строку (`ACTIVE` или `INACTIVE`) в базе данных благодаря `@Enumerated(EnumType.STRING)`.

---

## <a name="attribute-converter"></a>Custom Attribute Converter

**Custom Attribute Converter** — механизм JPA 2.1, поддерживаемый Hibernate, для преобразования значений атрибутов сущности между Java и базой данных. Это простой способ обработки нестандартных типов.

### <a name="attribute-converter-interface"></a>Интерфейс AttributeConverter

- Интерфейс `javax.persistence.AttributeConverter<X, Y>` определяет два метода:
    - `convertToDatabaseColumn(X)`: Преобразует Java-объект в SQL-значение.
    - `convertToEntityAttribute(Y)`: Преобразует SQL-значение в Java-объект.
- `X` — тип Java, `Y` — тип SQL.

### <a name="convert-annotation"></a>Аннотация @Convert

- Указывает, что поле сущности использует определённый конвертер.
- Применяется к конкретному полю, если автоматическое применение не используется.

### <a name="auto-apply"></a>Auto Apply AttributeConverter

- Аннотация `@Converter(autoApply = true)` позволяет автоматически применять конвертер ко всем полям заданного типа в приложении.
- Упрощает использование, но требует осторожности, чтобы избежать нежелательных преобразований.

---

## <a name="user-type"></a>Custom User Type

**Custom User Type** — мощный механизм Hibernate для маппинга сложных объектов, особенно при работе с несколькими столбцами базы данных или нестандартной логикой.

### <a name="json-type"></a>JSON Type

- Позволяет мапить Java-объекты на столбцы типа `JSON` или `JSONB` (например, в PostgreSQL).
- Часто используется с библиотекой `hibernate-types` для упрощения работы с JSON.

### <a name="hibernate-types"></a>Библиотека hibernate-types

- Расширяет Hibernate, предоставляя готовые типы для JSON, массивов, интервалов дат и других сложных структур.
- Упрощает интеграцию, заменяя необходимость писать кастомные `UserType` для распространённых случаев.

### <a name="type-annotation"></a>Аннотация @Type

- Указывает, что поле сущности использует кастомный тип Hibernate.
- Требует указания класса `UserType` или имени зарегистрированного типа.

### <a name="typedef-annotation"></a>Аннотация @TypeDef

- Регистрирует кастомный тип на уровне пакета или приложения.
- Позволяет использовать тип многократно без явного указания в каждой сущности.

---

## <a name="примеры"></a>Практические примеры

### <a name="пример-enum"></a>Конвертер для Enum

```java
import javax.persistence.AttributeConverter;
import javax.persistence.Converter;

@Converter(autoApply = true)
public class UserStatusConverter implements AttributeConverter<UserStatus, String> {
    @Override
    public String convertToDatabaseColumn(UserStatus status) {
        return status != null ? status.getCode() : null;
    }

    @Override
    public UserStatus convertToEntityAttribute(String code) {
        if (code == null) return null;
        return UserStatus.fromCode(code);
    }
}

public enum UserStatus {
    ACTIVE("A"), INACTIVE("I");
    private final String code;

    UserStatus(String code) { this.code = code; }
    public String getCode() { return code; }
    public static UserStatus fromCode(String code) {
        for (UserStatus status : values()) {
            if (status.code.equals(code)) return status;
        }
        throw new IllegalArgumentException("Unknown code: " + code);
    }
}
```

```java
import javax.persistence.Entity;
import javax.persistence.Id;

@Entity
public class User {
    @Id
    private Long id;
    private UserStatus status;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public UserStatus getStatus() { return status; }
    public void setStatus(UserStatus status) { this.status = status; }
}
```

### <a name="пример-json"></a>Конвертер для JSON

```java
import javax.persistence.AttributeConverter;
import javax.persistence.Converter;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

@Converter
public class JsonConverter implements AttributeConverter<Map<String, Object>, String> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(Map<String, Object> attribute) {
        try {
            return objectMapper.writeValueAsString(attribute);
        } catch (Exception e) {
            throw new RuntimeException("Failed to convert to JSON", e);
        }
    }

    @Override
    public Map<String, Object> convertToEntityAttribute(String dbData) {
        try {
            return objectMapper.readValue(dbData, Map.class);
        } catch (Exception e) {
            throw new RuntimeException("Failed to parse JSON", e);
        }
    }
}
```

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Convert;
import java.util.Map;

@Entity
public class User {
    @Id
    private Long id;
    @Convert(converter = JsonConverter.class)
    private Map<String, Object> attributes;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Map<String, Object> getAttributes() { return attributes; }
    public void setAttributes(Map<String, Object> attributes) { this.attributes = attributes; }
}
```

### <a name="пример-usertype"></a>UserType для объекта Money

```java
import org.hibernate.engine.spi.SharedSessionContractImplementor;
import org.hibernate.usertype.UserType;
import java.io.Serializable;
import java.math.BigDecimal;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Types;

public class MoneyType implements UserType<Money> {
    @Override
    public int getSqlType() {
        return Types.OTHER;
    }

    @Override
    public Class<Money> returnedClass() {
        return Money.class;
    }

    @Override
    public boolean equals(Money x, Money y) {
        return x == y || (x != null && x.equals(y));
    }

    @Override
    public int hashCode(Money x) {
        return x != null ? x.hashCode() : 0;
    }

    @Override
    public Money nullSafeGet(ResultSet rs, int position, SharedSessionContractImplementor session, Object owner) throws SQLException {
        BigDecimal amount = rs.getBigDecimal(position);
        String currency = rs.getString(position + 1);
        return amount != null && currency != null ? new Money(amount, currency) : null;
    }

    @Override
    public void nullSafeSet(PreparedStatement st, Money value, int index, SharedSessionContractImplementor session) throws SQLException {
        if (value == null) {
            st.setNull(index, Types.NUMERIC);
            st.setNull(index + 1, Types.VARCHAR);
        } else {
            st.setBigDecimal(index, value.getAmount());
            st.setString(index + 1, value.getCurrency());
        }
    }

    @Override
    public Money deepCopy(Money value) {
        return value != null ? new Money(value.getAmount(), value.getCurrency()) : null;
    }

    @Override
    public boolean isMutable() {
        return true;
    }

    @Override
    public Serializable disassemble(Money value) {
        return (Serializable) value;
    }

    @Override
    public Money assemble(Serializable cached, Object owner) {
        return (Money) cached;
    }

    @Override
    public Money replace(Money detached, Money managed, Object owner) {
        return detached;
    }
}

public class Money implements Serializable {
    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }
}
```

```java
import org.hibernate.annotations.Type;
import org.hibernate.annotations.TypeDef;
import org.hibernate.annotations.Columns;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;

@TypeDef(name = "money", typeClass = MoneyType.class)
@Entity
public class Account {
    @Id
    private Long id;
    @Type(type = "money")
    @Columns(columns = {
        @Column(name = "amount"),
        @Column(name = "currency")
    })
    private Money balance;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Money getBalance() { return balance; }
    public void setBalance(Money balance) { this.balance = balance; }
}
```

### JSON Type с hibernate-types

```java
import com.vladmihalcea.hibernate.type.json.JsonType;
import org.hibernate.annotations.Type;
import org.hibernate.annotations.TypeDef;
import javax.persistence.Entity;
import javax.persistence.Id;
import java.util.Map;

@TypeDef(name = "json", typeClass = JsonType.class)
@Entity
public class User {
    @Id
    private Long id;
    @Type(type = "json")
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> attributes;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Map<String, Object> getAttributes() { return attributes; }
    public void setAttributes(Map<String, Object> attributes) { this.attributes = attributes; }
}
```

Для использования `hibernate-types` добавьте зависимость в проект (например, через Maven):

```xml
<dependency>
    <groupId>com.vladmihalcea</groupId>
    <artifactId>hibernate-types-52</artifactId>
    <version>2.21.1</version>
</dependency>
```

---

## <a name="практики"></a>Лучшие практики

### Выбор типа конвертера

- **@Enumerated**: Используйте для простых enum без кастомной логики.
- **AttributeConverter**: Используйте для простых преобразований одного поля в один столбец.
- **UserType**: Используйте для сложных объектов или мультистолбцовых маппингов.

### Производительность

- Избегайте создания новых объектов в методах конвертации.
- Кэшируйте результаты преобразований, если это возможно.
- Используйте пулы объектов для часто используемых типов.

### Обработка ошибок

- Всегда обрабатывайте исключения в конвертерах.
- Предоставляйте понятные сообщения об ошибках.
- Логируйте проблемы для отладки.

### Тестирование

- Пишите unit-тесты для всех конвертеров.
- Тестируйте граничные случаи (null, пустые значения).
- Проверяйте производительность при больших объемах данных.

---

## <a name="вопросы"></a>Вопросы для собеседования

1. Что такое Type Converters в Hibernate и для чего они используются?
2. Какие типы конвертеров вы знаете? В чём их различия?
3. Как работает маппинг enum в Hibernate?
4. Что такое AttributeConverter и как его реализовать?
5. Когда стоит использовать UserType вместо AttributeConverter?
6. Как работает аннотация @Convert?
7. Что такое autoApply в AttributeConverter?
8. Как реализовать конвертер для JSON?
9. Как работает библиотека hibernate-types?
10. Какие проблемы могут возникнуть при использовании кастомных конвертеров?

---

**Рекомендуемые материалы:**
- [Hibernate Type System Documentation](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#basic)
- [JPA Attribute Converters](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#a13346)
- [Vlad Mihalcea: Hibernate Types](https://vladmihalcea.com/hibernate-types/)
- [Hibernate Types Library](https://github.com/vladmihalcea/hibernate-types)