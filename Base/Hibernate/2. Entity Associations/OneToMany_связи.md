# OneToMany - Связь "один ко многим"

## Оглавление
1. [Введение в OneToMany](#введение)
2. [Создание DDL и сущностей](#ddl-сущности)
3. [Аннотация @OneToMany](#аннотация)
4. [Двунаправленные связи](#двунаправленные)
5. [Тестирование связи](#тестирование)
6. [Практические примеры](#примеры)
7. [Лучшие практики](#практики)

---

## <a name="введение"></a>Введение в OneToMany

Связь "один ко многим" (`@OneToMany`) используется, когда одна сущность связана с множеством других сущностей. Например, один отдел (`Department`) может содержать множество сотрудников (`Employee`). Эта связь реализуется через внешний ключ в таблице дочерней сущности.

### Применение OneToMany

- **Отдел и сотрудники**: Один отдел содержит множество сотрудников
- **Клиент и заказы**: Один клиент может иметь множество заказов
- **Пользователь и посты**: Один пользователь может создать множество постов
- **Категория и продукты**: Одна категория может содержать множество продуктов
- **Компания и филиалы**: Одна компания может иметь множество филиалов

### Принцип работы

1. **Внешний ключ**: В таблице дочерней сущности создается столбец, ссылающийся на первичный ключ родительской сущности
2. **Коллекция**: В родительской сущности используется коллекция для хранения связанных объектов
3. **mappedBy**: Указывает поле в дочерней сущности, которое управляет связью

---

## <a name="ddl-сущности"></a>Создание DDL и сущностей

### Создание таблиц

```sql
-- Таблица отделов (родительская сущность)
CREATE TABLE departments (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица сотрудников (дочерняя сущность)
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    salary DECIMAL(10,2),
    department_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);

-- Индекс для улучшения производительности
CREATE INDEX idx_employees_department_id ON employees(department_id);
```

### Сущность Department

```java
import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "departments")
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "location")
    private String location;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Employee> employees = new ArrayList<>();
    
    // Конструкторы
    public Department() {}
    
    public Department(String name, String location) {
        this.name = name;
        this.location = location;
        this.createdAt = LocalDateTime.now();
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getLocation() { return location; }
    public void setLocation(String location) { this.location = location; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public List<Employee> getEmployees() { return employees; }
    public void setEmployees(List<Employee> employees) { this.employees = employees; }
    
    // Методы для управления связью
    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this);
    }
    
    public void removeEmployee(Employee employee) {
        employees.remove(employee);
        employee.setDepartment(null);
    }
    
    @Override
    public String toString() {
        return "Department{id=" + id + ", name='" + name + "', employees=" + employees.size() + "}";
    }
}
```

### Сущность Employee

```java
import javax.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "employees")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "email", nullable = false, unique = true)
    private String email;
    
    @Column(name = "salary")
    private BigDecimal salary;
    
    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // Конструкторы
    public Employee() {}
    
    public Employee(String name, String email, BigDecimal salary) {
        this.name = name;
        this.email = email;
        this.salary = salary;
        this.createdAt = LocalDateTime.now();
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public BigDecimal getSalary() { return salary; }
    public void setSalary(BigDecimal salary) { this.salary = salary; }
    
    public Department getDepartment() { return department; }
    public void setDepartment(Department department) { this.department = department; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    @Override
    public String toString() {
        return "Employee{id=" + id + ", name='" + name + "', department=" + 
               (department != null ? department.getName() : "null") + "}";
    }
}
```

---

## <a name="аннотация"></a>Аннотация @OneToMany

Аннотация `@OneToMany` указывает, что поле сущности содержит коллекцию связанных сущностей. Она используется с параметром `mappedBy`, который указывает поле в дочерней сущности.

### Базовая реализация

```java
@OneToMany(mappedBy = "department")
private List<Employee> employees = new ArrayList<>();
```

### Расширенная настройка

```java
@OneToMany(
    mappedBy = "department",                    // Поле в дочерней сущности
    cascade = CascadeType.ALL,                 // Каскадирование всех операций
    fetch = FetchType.LAZY,                    // Ленивая загрузка
    orphanRemoval = true                       // Удаление "сирот"
)
@OrderBy("name ASC")                          // Сортировка по имени
private List<Employee> employees = new ArrayList<>();
```

### Параметры аннотации

#### mappedBy
Указывает поле в дочерней сущности, которое управляет связью:
```java
@OneToMany(mappedBy = "department")
private List<Employee> employees;
```

#### cascade
Определяет типы каскадирования:
```java
@OneToMany(mappedBy = "department", cascade = CascadeType.PERSIST)
private List<Employee> employees;
```

#### fetch
Определяет стратегию загрузки:
```java
@OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
private List<Employee> employees;
```

#### orphanRemoval
Удаляет "сиротские" записи при удалении из коллекции:
```java
@OneToMany(mappedBy = "department", orphanRemoval = true)
private List<Employee> employees;
```

---

## <a name="двунаправленные"></a>Двунаправленные связи

### Принцип работы

В двунаправленной связи обе сущности имеют ссылки друг на друга:
- **Родительская сущность**: Содержит коллекцию дочерних сущностей
- **Дочерняя сущность**: Содержит ссылку на родительскую сущность

### Правила управления связью

1. **Владелец связи**: Дочерняя сущность является владельцем связи
2. **Синхронизация**: Изменения должны быть синхронизированы с обеих сторон
3. **Методы управления**: Используйте специальные методы для управления связью

### Пример двунаправленной связи

```java
@Entity
public class Department {
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees = new ArrayList<>();
    
    // Методы для управления связью
    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this);
    }
    
    public void removeEmployee(Employee employee) {
        employees.remove(employee);
        employee.setDepartment(null);
    }
}

@Entity
public class Employee {
    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
    
    // Геттеры и сеттеры
    public Department getDepartment() { return department; }
    public void setDepartment(Department department) { this.department = department; }
}
```

### Использование

```java
Department dept = new Department("IT", "Floor 3");
Employee emp1 = new Employee("John", "john@company.com", new BigDecimal("50000"));
Employee emp2 = new Employee("Jane", "jane@company.com", new BigDecimal("55000"));

// Правильное управление связью
dept.addEmployee(emp1);
dept.addEmployee(emp2);

em.persist(dept); // Сохраняются все сущности
```

---

## <a name="тестирование"></a>Тестирование связи

### Базовое тестирование

```java
import javax.persistence.*;
import java.math.BigDecimal;
import java.util.List;

public class OneToManyTest {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("pu");
        EntityManager em = emf.createEntityManager();
        
        try {
            em.getTransaction().begin();
            
            // Создание отдела
            Department dept = new Department("IT", "Floor 3");
            em.persist(dept);
            
            // Создание сотрудников
            Employee emp1 = new Employee("John Doe", "john@company.com", new BigDecimal("50000"));
            Employee emp2 = new Employee("Jane Smith", "jane@company.com", new BigDecimal("55000"));
            Employee emp3 = new Employee("Bob Wilson", "bob@company.com", new BigDecimal("48000"));
            
            // Добавление сотрудников в отдел
            dept.addEmployee(emp1);
            dept.addEmployee(emp2);
            dept.addEmployee(emp3);
            
            em.getTransaction().commit();
            
            // Проверка связи
            Department foundDept = em.find(Department.class, dept.getId());
            System.out.println("Отдел: " + foundDept.getName());
            System.out.println("Сотрудники:");
            foundDept.getEmployees().forEach(System.out::println);
            
            // Поиск сотрудников по отделу
            List<Employee> employeesInDept = em.createQuery(
                "SELECT e FROM Employee e WHERE e.department = :dept", Employee.class)
                .setParameter("dept", foundDept)
                .getResultList();
            
            System.out.println("Найдено сотрудников: " + employeesInDept.size());
            
        } catch (Exception e) {
            em.getTransaction().rollback();
            e.printStackTrace();
        } finally {
            em.close();
            emf.close();
        }
    }
}
```

### Тестирование с каскадированием

```java
@Entity
public class Department {
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();
    
    // ... остальной код ...
}

// Тестирование
public class CascadeTest {
    public static void main(String[] args) {
        EntityManager em = // получение EntityManager
        
        em.getTransaction().begin();
        
        Department dept = new Department("HR", "Floor 2");
        Employee emp = new Employee("Alice", "alice@company.com", new BigDecimal("45000"));
        dept.addEmployee(emp);
        
        // Сохраняется только department, employee сохраняется автоматически
        em.persist(dept);
        
        // Удаление сотрудника из коллекции
        dept.removeEmployee(emp);
        // Employee будет удален из БД благодаря orphanRemoval = true
        
        em.getTransaction().commit();
    }
}
```

---

## <a name="примеры"></a>Практические примеры

### Пример 1: Клиент и заказы

```java
@Entity
@Table(name = "customers")
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "email", nullable = false, unique = true)
    private String email;
    
    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
    
    // Методы управления связью
    public void addOrder(Order order) {
        orders.add(order);
        order.setCustomer(this);
    }
    
    public void removeOrder(Order order) {
        orders.remove(order);
        order.setCustomer(null);
    }
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", nullable = false, unique = true)
    private String orderNumber;
    
    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount;
    
    @ManyToOne
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;
    
    // Геттеры и сеттеры
}
```

### Пример 2: Пользователь и посты

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "username", nullable = false, unique = true)
    private String username;
    
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
    @OrderBy("createdAt DESC")
    private List<Post> posts = new ArrayList<>();
    
    // Методы управления связью
    public void addPost(Post post) {
        posts.add(post);
        post.setAuthor(this);
    }
    
    public void removePost(Post post) {
        posts.remove(post);
        post.setAuthor(null);
    }
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "posts")
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "title", nullable = false)
    private String title;
    
    @Column(name = "content", columnDefinition = "TEXT")
    private String content;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @ManyToOne
    @JoinColumn(name = "author_id", nullable = false)
    private User author;
    
    // Геттеры и сеттеры
}
```

### Пример 3: Категория и продукты

```java
@Entity
@Table(name = "categories")
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "description")
    private String description;
    
    @OneToMany(mappedBy = "category", cascade = CascadeType.ALL)
    @OrderBy("name ASC")
    private List<Product> products = new ArrayList<>();
    
    // Методы управления связью
    public void addProduct(Product product) {
        products.add(product);
        product.setCategory(this);
    }
    
    public void removeProduct(Product product) {
        products.remove(product);
        product.setCategory(null);
    }
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "price", nullable = false)
    private BigDecimal price;
    
    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;
    
    // Геттеры и сеттеры
}
```

---

## <a name="практики"></a>Лучшие практики

### 1. Правильное управление двунаправленными связями

```java
@Entity
public class Department {
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees = new ArrayList<>();
    
    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this);
    }
    
    public void removeEmployee(Employee employee) {
        employees.remove(employee);
        employee.setDepartment(null);
    }
}
```

### 2. Использование orphanRemoval

```java
@OneToMany(mappedBy = "department", orphanRemoval = true)
private List<Employee> employees = new ArrayList<>();
```

### 3. Оптимизация загрузки

```java
// Использование JOIN FETCH для избежания N+1 проблем
List<Department> depts = em.createQuery(
    "SELECT d FROM Department d JOIN FETCH d.employees", Department.class)
    .getResultList();
```

### 4. Сортировка коллекций

```java
@OneToMany(mappedBy = "department")
@OrderBy("name ASC")
private List<Employee> employees;
```

### 5. Обработка исключений

```java
try {
    Department dept = new Department("IT", "Floor 3");
    Employee emp = new Employee("John", "john@company.com", new BigDecimal("50000"));
    dept.addEmployee(emp);
    em.persist(dept);
} catch (Exception e) {
    // Обработка ошибок
    em.getTransaction().rollback();
    e.printStackTrace();
}
```

### 6. Использование индексов

```sql
-- Создание индекса для внешнего ключа
CREATE INDEX idx_employees_department_id ON employees(department_id);
```

---

**Связанные разделы:**
- [01_Основы_ассоциаций.md](01_Основы_ассоциаций.md) - Основы ассоциаций
- [02_ManyToOne_связи.md](02_ManyToOne_связи.md) - Связи "многие к одному"
- [06_Управление_загрузкой.md](06_Управление_загрузкой.md) - Управление загрузкой данных
- [07_Каскадирование.md](07_Каскадирование.md) - Каскадирование операций 