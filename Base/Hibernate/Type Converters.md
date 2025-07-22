**Type Converters в Hibernate**

Type Converters в Hibernate — это механизмы для преобразования данных между Java-объектами и типами базы данных. Они позволяют адаптировать нестандартные типы данных или специфические форматы для хранения в базе и обратно, упрощая работу с данными. Hibernate предоставляет встроенную систему типов для стандартных Java-типов (например, `String`, `Integer`, `Date`) и поддерживает расширение через **Custom Attribute Converters** и **Custom User Types**.

**Зачем нужны Type Converters?**

- **Преобразование данных**: Обеспечивают маппинг между Java-объектами и SQL-типами, поддерживая нестандартные типы (например, перечисления, JSON, сложные объекты).
- **Гибкость**: Позволяют настраивать форматы данных и сложные маппинги, такие как сериализация в JSON или маппинг объекта на несколько столбцов.
- **Прозрачность**: Скрывают детали преобразования, упрощая взаимодействие между Java-кодом и базой данных.

**Basic Types**

Hibernate предоставляет встроенную систему типов, которая автоматически сопоставляет стандартные Java-типы (например, `String`, `Integer`, `Date`) с соответствующими SQL-типами (например, `VARCHAR`, `INTEGER`, `TIMESTAMP`). Эти типы называются **Basic Types**.

**Type Descriptors**:

- Внутренние компоненты Hibernate, описывающие, как Java-тип мапится на SQL-тип.
- Управляют преобразованием данных и их сериализацией.
- Прозрачны для разработчика, но могут быть переопределены через кастомные типы.

**Enum User Type**:

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

**Custom Attribute Converter**

**Custom Attribute Converter** — механизм JPA 2.1, поддерживаемый Hibernate, для преобразования значений атрибутов сущности между Java и базой данных. Это простой способ обработки нестандартных типов.

**Интерфейс AttributeConverter**:

- Интерфейс `javax.persistence.AttributeConverter<X, Y>` определяет два метода:
    - `convertToDatabaseColumn(X)`: Преобразует Java-объект в SQL-значение.
    - `convertToEntityAttribute(Y)`: Преобразует SQL-значение в Java-объект.
- `X` — тип Java, `Y` — тип SQL.

**Аннотация @Convert**:

- Указывает, что поле сущности использует определённый конвертер.
- Применяется к конкретному полю, если автоматическое применение не используется.

**Auto Apply AttributeConverter**:

- Аннотация `@Converter(autoApply = true)` позволяет автоматически применять конвертер ко всем полям заданного типа в приложении.
- Упрощает использование, но требует осторожности, чтобы избежать нежелательных преобразований.

**Пример: Конвертер для Enum**

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

**Пример: Конвертер для JSON**

```java
import javax.persistence.AttributeConverter;
import javax.persistence.Converter;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

@Converter
public class JsonConverter implements AttributeConverter<Map<String, Object>, String> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @ dispatch: groovy
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

**Custom User Type**

**Custom User Type** — мощный механизм Hibernate для маппинга сложных объектов, особенно при работе с несколькими столбцами базы данных или нестандартной логикой.

**JSON Type**:

- Позволяет мапить Java-объекты на столбцы типа `JSON` или `JSONB` (например, в PostgreSQL).
- Часто используется с библиотекой `hibernate-types` для упрощения работы с JSON.

**Библиотека hibernate-types**:

- Расширяет Hibernate, предоставляя готовые типы для JSON, массивов, интервалов дат и других сложных структур.
- Упрощает интеграцию, заменяя необходимость писать кастомные `UserType` для распространённых случаев.

**Аннотация @Type**:

- Указывает, что поле сущности использует кастомный тип Hibernate.
- Требует указания класса `UserType` или имени зарегистрированного типа.

**Аннотация @TypeDef**:

- Регистрирует кастомный тип на уровне пакета или приложения.
- Позволяет использовать тип многократно без явного указания в каждой сущности.

**Пример: UserType для объекта Money**

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

**Пример: JSON Type с hibernate-types**

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

**Итог**

- **Basic Types**: Встроенные типы Hibernate для стандартных Java-типов, с поддержкой enum через `@Enumerated`.
- **Type Descriptors**: Внутренние метаданные для маппинга типов.
- **Custom Attribute Converter** (`@Converter`, `@Convert`, `autoApply`): Простой способ преобразования одного атрибута в один столбец, идеально для enum и JSON.
- **Custom User Type** (`@Type`, `@TypeDef`): Мощный механизм для сложных маппингов, включая мультистолбцовые типы и JSON с библиотекой `hibernate-types`.

Эти механизмы позволяют гибко настраивать маппинг данных в Hibernate, упрощая работу с нестандартными типами и сложными структурами.