# ManyToOne - Связь "многие к одному"

## Оглавление
1. [Введение в ManyToOne](#введение)
2. [Создание DDL и сущности](#ddl-сущности)
3. [Аннотация @ManyToOne](#аннотация)
4. [Тестирование связи](#тестирование)
5. [Внутренний класс ManyToOneType](#manytoonetype)
6. [Практические примеры](#примеры)
7. [Лучшие практики](#практики)

---

## <a name="введение"></a>Введение в ManyToOne

Связь "многие к одному" (`@ManyToOne`) используется, когда несколько сущностей ссылаются на одну общую сущность. Например, множество пользователей (`User`) могут быть связаны с одной компанией (`Company`). Эта связь реализуется через внешний ключ в таблице дочерней сущности.

### Применение ManyToOne

- **Пользователи принадлежат к компании**: Множество пользователей работают в одной компании
- **Заказы принадлежат клиенту**: Множество заказов может быть у одного клиента
- **Сообщения принадлежат пользователю**: Множество сообщений может быть у одного пользователя
- **Документы принадлежат отделу**: Множество документов может быть в одном отделе
- **Комментарии принадлежат посту**: Множество комментариев может быть у одного поста

### Принцип работы

1. **Внешний ключ**: В таблице дочерней сущности создается столбец, ссылающийся на первичный ключ родительской сущности
2. **Однонаправленная связь**: По умолчанию связь доступна только с одной стороны
3. **Ленивая загрузка**: По умолчанию связанная сущность загружается при первом обращении

---

## <a name="ddl-сущности"></a>Создание DDL и сущности

### Создание таблиц

Для работы с `@ManyToOne` создадим таблицу `companies` для хранения компаний и таблицу `users` для пользователей с внешним ключом `company_id`.

```sql
-- Таблица компаний (родительская сущность)
CREATE TABLE companies (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица пользователей (дочерняя сущность)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    company_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(id)
);

-- Индекс для улучшения производительности запросов
CREATE INDEX idx_users_company_id ON users(company_id);
```

### Сущность Company

```java
import javax.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "companies")
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // Конструкторы
    public Company() {}
    
    public Company(String name) {
        this.name = name;
        this.createdAt = LocalDateTime.now();
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    @Override
    public String toString() {
        return "Company{id=" + id + ", name='" + name + "'}";
    }
}
```

**Ключевые особенности:**
- **`id`**: Первичный ключ с автоинкрементом (`GenerationType.IDENTITY` соответствует PostgreSQL `SERIAL`)
- **`name`**: Название компании, хранимое в столбце `name`
- **`createdAt`**: Временная метка создания записи
- **`@Table`**: Явное указание имени таблицы
- **`@Column`**: Настройка столбцов с ограничениями

---

## <a name="аннотация"></a>Аннотация @ManyToOne

Аннотация `@ManyToOne` указывает, что поле сущности ссылается на другую сущность через внешний ключ. Она часто используется с аннотацией `@JoinColumn`, которая задаёт имя столбца внешнего ключа в таблице.

### Базовая реализация

```java
import javax.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "email", nullable = false, unique = true)
    private String email;
    
    @ManyToOne
    @JoinColumn(name = "company_id")
    private Company company;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // Конструкторы
    public User() {}
    
    public User(String name, String email) {
        this.name = name;
        this.email = email;
        this.createdAt = LocalDateTime.now();
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public Company getCompany() { return company; }
    public void setCompany(Company company) { this.company = company; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "', company=" + company + "}";
    }
}
```

### Расширенная настройка @JoinColumn

```java
@ManyToOne(
    fetch = FetchType.LAZY,           // Ленивая загрузка
    optional = true,                  // Связь может быть null
    cascade = CascadeType.PERSIST     // Каскадирование сохранения
)
@JoinColumn(
    name = "company_id",              // Имя столбца внешнего ключа
    referencedColumnName = "id",      // Имя столбца в целевой таблице
    nullable = true,                  // Может ли быть null
    unique = false,                   // Должен ли быть уникальным
    insertable = true,                // Участвует ли в INSERT
    updatable = true                  // Участвует ли в UPDATE
)
private Company company;
```

### Обязательная связь

```java
@ManyToOne(optional = false)
@JoinColumn(name = "company_id", nullable = false)
private Company company;
```

Если `optional = false`, попытка сохранить `User` с `company = null` вызовет исключение `ConstraintViolationException`.

---

## <a name="тестирование"></a>Тестирование связи

### Базовое тестирование

```java
import javax.persistence.*;
import java.util.List;

public class ManyToOneTest {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("pu");
        EntityManager em = emf.createEntityManager();
        
        try {
            em.getTransaction().begin();
            
            // Создание компании
            Company company = new Company("TechCorp");
            em.persist(company);
            System.out.println("Создана компания: " + company);
            
            // Создание пользователей
            User user1 = new User("John Doe", "john@techcorp.com");
            user1.setCompany(company);
            em.persist(user1);
            
            User user2 = new User("Jane Smith", "jane@techcorp.com");
            user2.setCompany(company);
            em.persist(user2);
            
            em.getTransaction().commit();
            
            // Проверка связи
            User foundUser = em.find(User.class, user1.getId());
            System.out.println("Найден пользователь: " + foundUser);
            System.out.println("Компания пользователя: " + foundUser.getCompany());
            
            // Поиск пользователей по компании
            List<User> usersInCompany = em.createQuery(
                "SELECT u FROM User u WHERE u.company = :company", User.class)
                .setParameter("company", company)
                .getResultList();
            
            System.out.println("Пользователи в компании " + company.getName() + ":");
            usersInCompany.forEach(System.out::println);
            
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
public class User {
    // ... другие поля ...
    
    @ManyToOne(cascade = CascadeType.PERSIST)
    @JoinColumn(name = "company_id")
    private Company company;
    
    // ... геттеры и сеттеры ...
}

// Тестирование
public class CascadeTest {
    public static void main(String[] args) {
        EntityManager em = // получение EntityManager
        
        em.getTransaction().begin();
        
        // Создание пользователя с новой компанией
        Company newCompany = new Company("Startup Inc");
        User user = new User("Alice", "alice@startup.com");
        user.setCompany(newCompany);
        
        // Сохраняется только user, company сохраняется автоматически
        em.persist(user);
        
        em.getTransaction().commit();
        
        // Проверка
        Company savedCompany = em.find(Company.class, newCompany.getId());
        System.out.println("Автоматически сохраненная компания: " + savedCompany);
    }
}
```

---

## <a name="manytoonetype"></a>Внутренний класс ManyToOneType

`ManyToOneType` — это внутренний класс Hibernate, который управляет маппингом связи `@ManyToOne`. Он отвечает за:

### Основные функции

- **Определение внешнего ключа**: Создание и управление связью через внешний ключ
- **Обработка стратегий выборки**: Управление ленивой и немедленной загрузкой
- **Каскадирование операций**: Применение операций к связанным сущностям
- **Валидация данных**: Проверка целостности связей

### Принцип работы

```java
// Внутренняя реализация Hibernate (упрощенно)
public class ManyToOneType {
    
    public Object get(ResultSet rs, String name, SessionImplementor session) {
        // Загрузка связанной сущности по внешнему ключу
        Object id = rs.getObject(name);
        if (id == null) return null;
        
        return session.get(associatedClass, id);
    }
    
    public void set(PreparedStatement st, Object value, int index, SessionImplementor session) {
        // Сохранение внешнего ключа
        if (value == null) {
            st.setNull(index, Types.BIGINT);
        } else {
            st.setLong(index, ((Company) value).getId());
        }
    }
}
```

### Отладка проблем

Разработчикам редко приходится взаимодействовать с `ManyToOneType` напрямую, но понимание его роли полезно при отладке:

- **Проблемы с производительностью**: Анализ SQL-запросов
- **Проблемы с каскадированием**: Отслеживание операций
- **Проблемы с ленивой загрузкой**: Диагностика прокси-объектов

---

## <a name="примеры"></a>Практические примеры

### Пример 1: Пользователи и роли

```java
@Entity
@Table(name = "roles")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, unique = true)
    private String name;
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "username", nullable = false, unique = true)
    private String username;
    
    @ManyToOne
    @JoinColumn(name = "role_id")
    private Role role;
    
    // Геттеры и сеттеры
}
```

### Пример 2: Заказы и клиенты

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

### Пример 3: Сообщения и пользователи

```java
@Entity
@Table(name = "messages")
public class Message {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "content", nullable = false)
    private String content;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @ManyToOne
    @JoinColumn(name = "sender_id", nullable = false)
    private User sender;
    
    @ManyToOne
    @JoinColumn(name = "recipient_id", nullable = false)
    private User recipient;
    
    // Геттеры и сеттеры
}
```

---

## <a name="практики"></a>Лучшие практики

### 1. Выбор стратегии загрузки

```java
// Для часто используемых связей
@ManyToOne(fetch = FetchType.EAGER)
@JoinColumn(name = "company_id")
private Company company;

// Для редко используемых связей
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "company_id")
private Company company;
```

### 2. Использование индексов

```sql
-- Создание индекса для внешнего ключа
CREATE INDEX idx_users_company_id ON users(company_id);
```

### 3. Валидация данных

```java
@ManyToOne(optional = false)
@JoinColumn(name = "company_id", nullable = false)
private Company company;

// В коде
public void setCompany(Company company) {
    if (company == null) {
        throw new IllegalArgumentException("Company cannot be null");
    }
    this.company = company;
}
```

### 4. Оптимизация запросов

```java
// Использование JOIN FETCH для избежания N+1 проблем
List<User> users = em.createQuery(
    "SELECT u FROM User u JOIN FETCH u.company", User.class)
    .getResultList();
```

### 5. Обработка исключений

```java
try {
    User user = new User("John", "john@example.com");
    user.setCompany(null); // Вызовет исключение если optional = false
    em.persist(user);
} catch (ConstraintViolationException e) {
    // Обработка ошибки валидации
    System.err.println("Company is required for user");
}
```

---

**Связанные разделы:**
- [01_Основы_ассоциаций.md](01_Основы_ассоциаций.md) - Основы ассоциаций
- [06_Управление_загрузкой.md](06_Управление_загрузкой.md) - Управление загрузкой данных
- [07_Каскадирование.md](07_Каскадирование.md) - Каскадирование операций 