# Audit зависимых сущностей

## Оглавление
1. [Введение](#введение)
2. [Типы связей между сущностями](#типы-связей-между-сущностями)
3. [Аудит OneToOne связей](#аудит-onetoone-связей)
4. [Аудит OneToMany связей](#аудит-onetomany-связей)
5. [Аудит ManyToOne связей](#аудит-manytoone-связей)
6. [Аудит ManyToMany связей](#аудит-manytomany-связей)
7. [Аудит наследования](#аудит-наследования)
8. [Практические примеры](#практические-примеры)
9. [Лучшие практики](#лучшие-практики)

---

## <a name="введение"></a>Введение

**Audit зависимых сущностей** — это механизм Hibernate Envers для отслеживания изменений в связанных сущностях. Когда одна сущность изменяется, Envers может автоматически создавать audit записи для связанных с ней сущностей, обеспечивая полную картину изменений в системе.

### Основные концепции

- **Зависимые сущности** - сущности, связанные с основной через JPA отношения
- **Каскадный аудит** - автоматическое создание audit записей для связанных сущностей
- **Стратегии аудита** - различные подходы к аудиту связей
- **Производительность** - оптимизация аудита для сложных связей

---

## <a name="типы-связей-между-сущностями"></a>Типы связей между сущностями

### Классификация связей

```
┌─────────────────┐    ┌─────────────────┐
│   OneToOne      │    │   OneToMany     │
│   (1:1)         │    │   (1:N)         │
└─────────────────┘    └─────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│   ManyToOne     │    │   ManyToMany    │
│   (N:1)         │    │   (N:N)         │
└─────────────────┘    └─────────────────┘
```

### Стратегии аудита связей

| Тип связи | Стратегия аудита | Описание |
|-----------|------------------|----------|
| OneToOne | Встроенный аудит | Аудит обеих сущностей |
| OneToMany | Коллекция + элементы | Аудит родителя и коллекции |
| ManyToOne | Ссылка на аудируемую сущность | Аудит зависимой сущности |
| ManyToMany | Промежуточная таблица | Аудит связей через таблицу |

---

## <a name="аудит-onetoone-связей"></a>Аудит OneToOne связей

### Базовая OneToOne связь

```java
@Entity
@Audited
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    private String email;
    
    @OneToOne(cascade = CascadeType.ALL)
    @Audited
    private UserProfile profile;
    
    // getters and setters
}

@Entity
@Audited
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String firstName;
    private String lastName;
    private String phone;
    
    @OneToOne(mappedBy = "profile")
    private User user;
    
    // getters and setters
}
```

### Созданные audit таблицы

```sql
-- Основные таблицы
CREATE TABLE user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(255),
    email VARCHAR(255),
    profile_id BIGINT UNIQUE,
    FOREIGN KEY (profile_id) REFERENCES user_profile(id)
);

CREATE TABLE user_profile (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(255)
);

-- Audit таблицы
CREATE TABLE user_aud (
    id BIGINT NOT NULL,
    username VARCHAR(255),
    email VARCHAR(255),
    profile_id BIGINT,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

CREATE TABLE user_profile_aud (
    id BIGINT NOT NULL,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(255),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

### Пример с кастомной стратегией

```java
@Entity
@Audited
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String employeeNumber;
    private String department;
    
    @OneToOne(cascade = CascadeType.ALL)
    @Audited(auditParents = {Employee.class})
    private EmployeeDetails details;
    
    // getters and setters
}

@Entity
@Audited
public class EmployeeDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String address;
    private String emergencyContact;
    private LocalDate hireDate;
    
    @OneToOne(mappedBy = "details")
    private Employee employee;
    
    // getters and setters
}
```

---

## <a name="аудит-onetomany-связей"></a>Аудит OneToMany связей

### Базовая OneToMany связь

```java
@Entity
@Audited
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String location;
    
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    @Audited
    private List<Employee> employees;
    
    // getters and setters
}

@Entity
@Audited
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String firstName;
    private String lastName;
    private String position;
    
    @ManyToOne
    private Department department;
    
    // getters and setters
}
```

### Созданные audit таблицы

```sql
-- Основные таблицы
CREATE TABLE department (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    location VARCHAR(255)
);

CREATE TABLE employee (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    position VARCHAR(255),
    department_id BIGINT,
    FOREIGN KEY (department_id) REFERENCES department(id)
);

-- Audit таблицы
CREATE TABLE department_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    location VARCHAR(255),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

CREATE TABLE employee_aud (
    id BIGINT NOT NULL,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    position VARCHAR(255),
    department_id BIGINT,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

-- Audit таблица для коллекции
CREATE TABLE department_employees_aud (
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    department_id BIGINT NOT NULL,
    employees_id BIGINT NOT NULL,
    PRIMARY KEY (REV, department_id, employees_id),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

### Пример с кастомной коллекцией

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
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    @Audited(auditChildren = {OrderItem.class})
    private Set<OrderItem> items;
    
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

---

## <a name="аудит-manytoone-связей"></a>Аудит ManyToOne связей

### Базовая ManyToOne связь

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
    
    @ManyToOne
    @Audited
    private Category category;
    
    // getters and setters
}

@Entity
@Audited
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String description;
    
    @OneToMany(mappedBy = "category")
    private List<Product> products;
    
    // getters and setters
}
```

### Созданные audit таблицы

```sql
-- Основные таблицы
CREATE TABLE category (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    description TEXT
);

CREATE TABLE product (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    price DECIMAL(10,2),
    description TEXT,
    category_id BIGINT,
    FOREIGN KEY (category_id) REFERENCES category(id)
);

-- Audit таблицы
CREATE TABLE category_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    description TEXT,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

CREATE TABLE product_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    price DECIMAL(10,2),
    description TEXT,
    category_id BIGINT,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

### Пример с кастомной стратегией

```java
@Entity
@Audited
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private Integer quantity;
    private BigDecimal unitPrice;
    
    @ManyToOne
    @Audited(auditParents = {Order.class})
    private Order order;
    
    @ManyToOne
    @Audited
    private Product product;
    
    // getters and setters
}

@Entity
@Audited
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private LocalDateTime orderDate;
    
    @OneToMany(mappedBy = "order")
    private List<OrderItem> items;
    
    // getters and setters
}
```

---

## <a name="аудит-manytomany-связей"></a>Аудит ManyToMany связей

### Базовая ManyToMany связь

```java
@Entity
@Audited
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    @Audited
    private Set<Course> courses;
    
    // getters and setters
}

@Entity
@Audited
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String description;
    private Integer credits;
    
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students;
    
    // getters and setters
}
```

### Созданные audit таблицы

```sql
-- Основные таблицы
CREATE TABLE student (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    email VARCHAR(255)
);

CREATE TABLE course (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    description TEXT,
    credits INTEGER
);

CREATE TABLE student_course (
    student_id BIGINT,
    course_id BIGINT,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES student(id),
    FOREIGN KEY (course_id) REFERENCES course(id)
);

-- Audit таблицы
CREATE TABLE student_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    email VARCHAR(255),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

CREATE TABLE course_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    description TEXT,
    credits INTEGER,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

-- Audit таблица для связи ManyToMany
CREATE TABLE student_course_aud (
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    student_id BIGINT NOT NULL,
    course_id BIGINT NOT NULL,
    PRIMARY KEY (REV, student_id, course_id),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

### Пример с кастомной промежуточной сущностью

```java
@Entity
@Audited
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @OneToMany(mappedBy = "student")
    @Audited
    private List<Enrollment> enrollments;
    
    // getters and setters
}

@Entity
@Audited
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String description;
    
    @OneToMany(mappedBy = "course")
    private List<Enrollment> enrollments;
    
    // getters and setters
}

@Entity
@Audited
public class Enrollment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @Audited
    private Student student;
    
    @ManyToOne
    @Audited
    private Course course;
    
    private LocalDate enrollmentDate;
    private String grade;
    
    // getters and setters
}
```

---

## <a name="аудит-наследования"></a>Аудит наследования

### Single Table Inheritance

```java
@Entity
@Audited
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    // getters and setters
}

@Entity
@DiscriminatorValue("EMPLOYEE")
public class Employee extends Person {
    private String employeeNumber;
    private String department;
    private BigDecimal salary;
    
    // getters and setters
}

@Entity
@DiscriminatorValue("CUSTOMER")
public class Customer extends Person {
    private String customerNumber;
    private String membershipLevel;
    private LocalDate registrationDate;
    
    // getters and setters
}
```

### Созданные audit таблицы

```sql
-- Основная таблица
CREATE TABLE person (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    email VARCHAR(255),
    type VARCHAR(31),
    employee_number VARCHAR(255),
    department VARCHAR(255),
    salary DECIMAL(10,2),
    customer_number VARCHAR(255),
    membership_level VARCHAR(255),
    registration_date DATE
);

-- Audit таблица
CREATE TABLE person_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    email VARCHAR(255),
    type VARCHAR(31),
    employee_number VARCHAR(255),
    department VARCHAR(255),
    salary DECIMAL(10,2),
    customer_number VARCHAR(255),
    membership_level VARCHAR(255),
    registration_date DATE,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

### Joined Table Inheritance

```java
@Entity
@Audited
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String make;
    private String model;
    private Integer year;
    
    // getters and setters
}

@Entity
public class Car extends Vehicle {
    private Integer numberOfDoors;
    private String fuelType;
    
    // getters and setters
}

@Entity
public class Motorcycle extends Vehicle {
    private Integer engineCapacity;
    private String type;
    
    // getters and setters
}
```

### Созданные audit таблицы

```sql
-- Основные таблицы
CREATE TABLE vehicle (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    make VARCHAR(255),
    model VARCHAR(255),
    year INTEGER
);

CREATE TABLE car (
    id BIGINT PRIMARY KEY,
    number_of_doors INTEGER,
    fuel_type VARCHAR(255),
    FOREIGN KEY (id) REFERENCES vehicle(id)
);

CREATE TABLE motorcycle (
    id BIGINT PRIMARY KEY,
    engine_capacity INTEGER,
    type VARCHAR(255),
    FOREIGN KEY (id) REFERENCES vehicle(id)
);

-- Audit таблицы
CREATE TABLE vehicle_aud (
    id BIGINT NOT NULL,
    make VARCHAR(255),
    model VARCHAR(255),
    year INTEGER,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

CREATE TABLE car_aud (
    id BIGINT NOT NULL,
    number_of_doors INTEGER,
    fuel_type VARCHAR(255),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

CREATE TABLE motorcycle_aud (
    id BIGINT NOT NULL,
    engine_capacity INTEGER,
    type VARCHAR(255),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

---

## <a name="практические-примеры"></a>Практические примеры

### Пример 1: Система управления библиотекой

```java
@Entity
@Audited
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    private String isbn;
    private String author;
    
    @ManyToOne
    @Audited
    private Category category;
    
    @OneToMany(mappedBy = "book", cascade = CascadeType.ALL)
    @Audited
    private List<BookCopy> copies;
    
    // getters and setters
}

@Entity
@Audited
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String description;
    
    @OneToMany(mappedBy = "category")
    private List<Book> books;
    
    // getters and setters
}

@Entity
@Audited
public class BookCopy {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    private Book book;
    
    private String copyNumber;
    private BookStatus status;
    
    @OneToMany(mappedBy = "bookCopy")
    @Audited
    private List<Loan> loans;
    
    // getters and setters
}

@Entity
@Audited
public class Loan {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @Audited
    private Member member;
    
    @ManyToOne
    private BookCopy bookCopy;
    
    private LocalDate loanDate;
    private LocalDate dueDate;
    private LocalDate returnDate;
    
    // getters and setters
}
```

### Пример 2: Система управления заказами

```java
@Entity
@Audited
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String firstName;
    private String lastName;
    private String email;
    
    @OneToMany(mappedBy = "customer")
    @Audited
    private List<Order> orders;
    
    @OneToOne(cascade = CascadeType.ALL)
    @Audited
    private CustomerProfile profile;
    
    // getters and setters
}

@Entity
@Audited
public class CustomerProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String phone;
    private String address;
    private LocalDate birthDate;
    
    @OneToOne(mappedBy = "profile")
    private Customer customer;
    
    // getters and setters
}

@Entity
@Audited
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private LocalDateTime orderDate;
    private BigDecimal totalAmount;
    
    @ManyToOne
    @Audited
    private Customer customer;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    @Audited
    private List<OrderItem> items;
    
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

---

## <a name="лучшие-практики"></a>Лучшие практики

### 1. Оптимизация производительности

```java
// ✅ ПРАВИЛЬНО - Аудит только необходимых связей
@Entity
@Audited
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @Audited // Аудируем только важные связи
    private Customer customer;
    
    @ManyToOne
    @NotAudited // Не аудируем редко изменяемые связи
    private ShippingMethod shippingMethod;
    
    // getters and setters
}
```

### 2. Избегание циклических зависимостей

```java
// ❌ НЕПРАВИЛЬНО - Циклическая зависимость
@Entity
@Audited
public class Department {
    @OneToMany(mappedBy = "department")
    @Audited
    private List<Employee> employees;
}

@Entity
@Audited
public class Employee {
    @ManyToOne
    @Audited
    private Department department;
}

// ✅ ПРАВИЛЬНО - Односторонняя связь
@Entity
@Audited
public class Department {
    @OneToMany(mappedBy = "department")
    @NotAudited // Не аудируем коллекцию
    private List<Employee> employees;
}

@Entity
@Audited
public class Employee {
    @ManyToOne
    @Audited // Аудируем только ссылку
    private Department department;
}
```

### 3. Кастомизация стратегий аудита

```java
@Entity
@Audited
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private BigDecimal price;
    
    @ManyToOne
    @Audited(auditParents = {Category.class})
    private Category category;
    
    // getters and setters
}
```

### 4. Управление каскадными операциями

```java
@Entity
@Audited
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    @Audited
    private List<OrderItem> items;
    
    // getters and setters
}

// При удалении заказа, элементы заказа также удаляются
// и создаются соответствующие audit записи
```

### 5. Мониторинг размера audit таблиц

```sql
-- Проверка размера audit таблиц для связанных сущностей
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)',
    table_rows
FROM information_schema.tables 
WHERE table_schema = 'your_database' 
AND table_name LIKE '%_aud'
ORDER BY (data_length + index_length) DESC;
```

---

## Заключение

Audit зависимых сущностей в Hibernate Envers предоставляет мощный механизм для отслеживания изменений в сложных системах:

1. **Гибкость** - поддержка всех типов JPA связей
2. **Производительность** - оптимизированные стратегии аудита
3. **Масштабируемость** - эффективная работа с большими объемами данных
4. **Надежность** - сохранение целостности данных при изменениях
5. **Мониторинг** - возможность отслеживания размера и производительности

Правильная настройка аудита зависимых сущностей обеспечивает полную картину изменений в системе и эффективное управление данными. 