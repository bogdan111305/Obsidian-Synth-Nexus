# Аннотации @Audited и @NotAudited

## Оглавление
1. [Введение](#введение)
2. [Аннотация @Audited](#аннотация-audited)
3. [Аннотация @NotAudited](#аннотация-notaudited)
4. [Уровни применения аннотаций](#уровни-применения-аннотаций)
5. [Параметры аннотаций](#параметры-аннотаций)
6. [Практические примеры](#практические-примеры)
7. [Лучшие практики](#лучшие-практики)
8. [Частые ошибки](#частые-ошибки)

---

## <a name="введение"></a>Введение

Аннотации `@Audited` и `@NotAudited` являются основными инструментами для настройки аудита в Hibernate Envers. Они позволяют гибко управлять тем, какие сущности и поля должны отслеживаться в системе аудита.

### Основные концепции

- **@Audited** - включает аудит для сущности или поля
- **@NotAudited** - исключает поле из аудита
- **Наследование аудита** - аудит может наследоваться от родительских классов
- **Гранулярность** - аудит можно настроить на уровне класса, поля или коллекции

---

## <a name="аннотация-audited"></a>Аннотация @Audited

### Базовое использование

```java
import org.hibernate.envers.Audited;
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;

@Entity
@Audited
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    // getters and setters
}
```

### Параметры аннотации @Audited

```java
@Audited(
    auditParents = {ParentEntity.class},  // Родительские сущности для аудита
    auditChildren = {ChildEntity.class},  // Дочерние сущности для аудита
    auditTableName = "custom_audit_table", // Кастомное имя таблицы аудита
    auditTableSuffix = "_HISTORY"         // Кастомный суффикс таблицы
)
```

### Примеры использования

#### 1. Аудит на уровне класса

```java
@Entity
@Audited
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private BigDecimal price;
    private String description;
    
    // getters and setters
}
```

#### 2. Аудит с кастомным именем таблицы

```java
@Entity
@Audited(auditTableName = "product_history")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private BigDecimal price;
    
    // getters and setters
}
```

#### 3. Аудит с кастомным суффиксом

```java
@Entity
@Audited(auditTableSuffix = "_VERSIONS")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private LocalDateTime orderDate;
    private BigDecimal totalAmount;
    
    // getters and setters
}
```

---

## <a name="аннотация-notaudited"></a>Аннотация @NotAudited

### Базовое использование

```java
import org.hibernate.envers.NotAudited;
import javax.persistence.Entity;
import javax.persistence.Id;

@Entity
@Audited
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @NotAudited
    private String password; // Пароль не будет отслеживаться
    
    @NotAudited
    private String temporaryToken; // Временные данные не аудируются
    
    // getters and setters
}
```

### Примеры исключения полей

#### 1. Исключение чувствительных данных

```java
@Entity
@Audited
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String firstName;
    private String lastName;
    private String email;
    
    @NotAudited
    private String socialSecurityNumber; // Чувствительные данные
    
    @NotAudited
    private String bankAccountNumber; // Финансовая информация
    
    // getters and setters
}
```

#### 2. Исключение временных полей

```java
@Entity
@Audited
public class Document {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    private String content;
    private LocalDateTime createdAt;
    
    @NotAudited
    private LocalDateTime lastAccessed; // Временное поле
    
    @NotAudited
    private String sessionId; // Сессионные данные
    
    // getters and setters
}
```

#### 3. Исключение вычисляемых полей

```java
@Entity
@Audited
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private BigDecimal unitPrice;
    private Integer quantity;
    
    @NotAudited
    public BigDecimal getTotalPrice() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
    
    // getters and setters
}
```

---

## <a name="уровни-применения-аннотаций"></a>Уровни применения аннотаций

### 1. Уровень класса

```java
@Entity
@Audited // Аудит всего класса
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    private String phone;
    
    // Все поля будут аудироваться
}
```

### 2. Уровень поля

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Audited // Только это поле будет аудироваться
    private String name;
    
    private String description; // Не будет аудироваться
    
    @Audited
    private BigDecimal price; // Будет аудироваться
    
    // getters and setters
}
```

### 3. Уровень коллекции

```java
@Entity
@Audited
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "department")
    @Audited // Аудит коллекции сотрудников
    private List<Employee> employees;
    
    @OneToMany(mappedBy = "department")
    @NotAudited // Исключение из аудита
    private List<Document> documents;
    
    // getters and setters
}
```

### 4. Уровень наследования

```java
@MappedSuperclass
@Audited
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // getters and setters
}

@Entity
public class User extends BaseEntity {
    // Наследует аудит от BaseEntity
    private String name;
    private String email;
    
    // getters and setters
}
```

---

## <a name="параметры-аннотаций"></a>Параметры аннотаций

### Параметры @Audited

```java
@Audited(
    // Родительские сущности для аудита
    auditParents = {ParentEntity.class, AnotherParent.class},
    
    // Дочерние сущности для аудита
    auditChildren = {ChildEntity.class},
    
    // Кастомное имя таблицы аудита
    auditTableName = "custom_audit_table",
    
    // Кастомный суффикс таблицы
    auditTableSuffix = "_HISTORY"
)
```

### Примеры с параметрами

#### 1. Аудит с родительскими сущностями

```java
@Entity
@Audited(auditParents = {BaseEntity.class})
public class Employee extends BaseEntity {
    private String firstName;
    private String lastName;
    private String position;
    
    // getters and setters
}
```

#### 2. Аудит с дочерними сущностями

```java
@Entity
@Audited(auditChildren = {OrderItem.class})
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private LocalDateTime orderDate;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items;
    
    // getters and setters
}
```

#### 3. Кастомная таблица аудита

```java
@Entity
@Audited(
    auditTableName = "user_audit_history",
    auditTableSuffix = "_VERSIONS"
)
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    private String email;
    
    // getters and setters
}
```

---

## <a name="практические-примеры"></a>Практические примеры

### Пример 1: Система управления пользователями

```java
@Entity
@Audited
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    private String email;
    private String firstName;
    private String lastName;
    
    @NotAudited
    private String passwordHash; // Безопасность
    
    @NotAudited
    private String lastLoginIp; // Временные данные
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status;
    
    // getters and setters
}
```

### Пример 2: Система управления заказами

```java
@Entity
@Audited
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private LocalDateTime orderDate;
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @ManyToOne
    @Audited
    private Customer customer;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    @Audited
    private List<OrderItem> items;
    
    @NotAudited
    private String internalNotes; // Внутренние заметки
    
    // getters and setters
}

@Entity
@Audited
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    private Order order;
    
    @ManyToOne
    @Audited
    private Product product;
    
    private Integer quantity;
    private BigDecimal unitPrice;
    
    // getters and setters
}
```

### Пример 3: Система управления документами

```java
@Entity
@Audited
public class Document {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    private String content;
    private String filePath;
    
    @Enumerated(EnumType.STRING)
    private DocumentStatus status;
    
    @ManyToOne
    @Audited
    private User createdBy;
    
    @ManyToOne
    @Audited
    private User lastModifiedBy;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    @NotAudited
    private String temporaryFilePath; // Временные файлы
    
    @NotAudited
    private String sessionId; // Сессионные данные
    
    // getters and setters
}
```

---

## <a name="лучшие-практики"></a>Лучшие практики

### 1. Безопасность данных

```java
@Entity
@Audited
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    private String email;
    
    @NotAudited
    private String passwordHash; // Никогда не аудируйте пароли
    
    @NotAudited
    private String creditCardNumber; // Не аудируйте финансовые данные
    
    // getters and setters
}
```

### 2. Производительность

```java
@Entity
@Audited
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String description;
    private BigDecimal price;
    
    @NotAudited
    private byte[] largeImage; // Избегайте аудита больших объектов
    
    @NotAudited
    private String temporaryCache; // Не аудируйте кэш
    
    // getters and setters
}
```

### 3. Структурированность

```java
@MappedSuperclass
@Audited
public abstract class AuditableEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    @ManyToOne
    @Audited
    private User createdBy;
    
    @ManyToOne
    @Audited
    private User updatedBy;
    
    // getters and setters
}

@Entity
public class Product extends AuditableEntity {
    // Наследует аудит от родительского класса
    private String name;
    private String description;
    private BigDecimal price;
    
    // getters and setters
}
```

### 4. Именование таблиц

```java
@Entity
@Audited(auditTableSuffix = "_AUDIT")
public class Customer {
    // Таблица аудита будет называться: customer_audit
    // getters and setters
}

@Entity
@Audited(auditTableName = "customer_history")
public class Customer {
    // Таблица аудита будет называться: customer_history
    // getters and setters
}
```

---

## <a name="частые-ошибки"></a>Частые ошибки

### 1. Аудит чувствительных данных

```java
// ❌ НЕПРАВИЛЬНО
@Entity
@Audited
public class User {
    private String password; // Пароль будет в аудите!
    private String creditCardNumber; // Номер карты в аудите!
}

// ✅ ПРАВИЛЬНО
@Entity
@Audited
public class User {
    @NotAudited
    private String password;
    
    @NotAudited
    private String creditCardNumber;
}
```

### 2. Аудит больших объектов

```java
// ❌ НЕПРАВИЛЬНО
@Entity
@Audited
public class Document {
    private byte[] largeFile; // Большой файл в аудите
}

// ✅ ПРАВИЛЬНО
@Entity
@Audited
public class Document {
    @NotAudited
    private byte[] largeFile;
    
    private String fileName; // Аудируем только метаданные
    private Long fileSize;
}
```

### 3. Аудит временных данных

```java
// ❌ НЕПРАВИЛЬНО
@Entity
@Audited
public class Session {
    private String sessionId; // Временные данные в аудите
    private LocalDateTime lastAccess;
}

// ✅ ПРАВИЛЬНО
@Entity
@Audited
public class Session {
    @NotAudited
    private String sessionId;
    
    @NotAudited
    private LocalDateTime lastAccess;
    
    private String userId; // Аудируем только важные данные
}
```

### 4. Неправильное наследование

```java
// ❌ НЕПРАВИЛЬНО
@MappedSuperclass
public abstract class BaseEntity {
    // Нет @Audited
}

@Entity
@Audited
public class User extends BaseEntity {
    // Аудит не наследуется
}

// ✅ ПРАВИЛЬНО
@MappedSuperclass
@Audited
public abstract class BaseEntity {
    // Аудит включен
}

@Entity
public class User extends BaseEntity {
    // Аудит наследуется
}
```

---

## Заключение

Аннотации `@Audited` и `@NotAudited` предоставляют гибкий механизм для настройки аудита в Hibernate Envers:

1. **@Audited** - включает аудит для сущности или поля
2. **@NotAudited** - исключает поле из аудита
3. **Уровни применения** - класс, поле, коллекция, наследование
4. **Параметры** - кастомизация имен таблиц и связей
5. **Безопасность** - исключение чувствительных данных
6. **Производительность** - избегание аудита больших объектов

Правильное использование этих аннотаций обеспечивает эффективный и безопасный аудит данных в приложении. 