# OneToOne - Связь "один к одному"

## Оглавление
1. [Введение в OneToOne](#введение)
2. [Создание DDL и сущностей](#ddl-сущности)
3. [Аннотация @OneToOne](#аннотация)
4. [Двунаправленные связи](#двунаправленные)
5. [Тестирование связи](#тестирование)
6. [Практические примеры](#примеры)
7. [Лучшие практики](#практики)

---

## <a name="введение"></a>Введение в OneToOne

Связь "один к одному" (`@OneToOne`) используется, когда каждая сущность связана ровно с одной другой сущностью. Например, один пользователь (`User`) может иметь один профиль (`UserProfile`). Эта связь может быть реализована через внешний ключ в одной из таблиц или через промежуточную таблицу.

### Применение OneToOne

- **Пользователь и профиль**: Один пользователь имеет один профиль
- **Сотрудник и рабочее место**: Один сотрудник занимает одно рабочее место
- **Заказ и доставка**: Один заказ имеет одну доставку
- **Студент и диплом**: Один студент получает один диплом
- **Автомобиль и владелец**: Один автомобиль принадлежит одному владельцу

### Принцип работы

1. **Внешний ключ**: В одной из таблиц создается столбец, ссылающийся на первичный ключ другой таблицы
2. **Уникальность**: Внешний ключ должен быть уникальным
3. **Двунаправленность**: Часто используется как двунаправленная связь

---

## <a name="ddl-сущности"></a>Создание DDL и сущностей

### Создание таблиц

```sql
-- Таблица пользователей (основная сущность)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица профилей (связанная сущность)
CREATE TABLE user_profiles (
    id BIGSERIAL PRIMARY KEY,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    birth_date DATE,
    user_id BIGINT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Индекс для улучшения производительности
CREATE INDEX idx_user_profiles_user_id ON user_profiles(user_id);
```

### Сущность User

```java
import javax.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "username", nullable = false, unique = true)
    private String username;
    
    @Column(name = "email", nullable = false, unique = true)
    private String email;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private UserProfile profile;
    
    // Конструкторы
    public User() {}
    
    public User(String username, String email) {
        this.username = username;
        this.email = email;
        this.createdAt = LocalDateTime.now();
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public UserProfile getProfile() { return profile; }
    public void setProfile(UserProfile profile) { this.profile = profile; }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", username='" + username + "', email='" + email + "'}";
    }
}
```

### Сущность UserProfile

```java
import javax.persistence.*;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "user_profiles")
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "first_name")
    private String firstName;
    
    @Column(name = "last_name")
    private String lastName;
    
    @Column(name = "phone")
    private String phone;
    
    @Column(name = "address", columnDefinition = "TEXT")
    private String address;
    
    @Column(name = "birth_date")
    private LocalDate birthDate;
    
    @OneToOne
    @JoinColumn(name = "user_id", nullable = false, unique = true)
    private User user;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // Конструкторы
    public UserProfile() {}
    
    public UserProfile(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.createdAt = LocalDateTime.now();
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    
    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
    
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    
    public String getAddress() { return address; }
    public void setAddress(String address) { this.address = address; }
    
    public LocalDate getBirthDate() { return birthDate; }
    public void setBirthDate(LocalDate birthDate) { this.birthDate = birthDate; }
    
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    @Override
    public String toString() {
        return "UserProfile{id=" + id + ", firstName='" + firstName + 
               "', lastName='" + lastName + "', user=" + (user != null ? user.getUsername() : "null") + "}";
    }
}
```

---

## <a name="аннотация"></a>Аннотация @OneToOne

Аннотация `@OneToOne` указывает, что поле сущности связано с одной другой сущностью. Она может использоваться как с `@JoinColumn`, так и с `mappedBy`.

### Базовая реализация

```java
// В дочерней сущности (владелец связи)
@OneToOne
@JoinColumn(name = "user_id", nullable = false, unique = true)
private User user;

// В родительской сущности
@OneToOne(mappedBy = "user")
private UserProfile profile;
```

### Расширенная настройка

```java
@OneToOne(
    mappedBy = "user",                    // Поле в дочерней сущности
    cascade = CascadeType.ALL,            // Каскадирование всех операций
    fetch = FetchType.LAZY,               // Ленивая загрузка
    optional = true                       // Связь может быть null
)
private UserProfile profile;
```

### Параметры аннотации

#### mappedBy
Указывает поле в дочерней сущности, которое управляет связью:
```java
@OneToOne(mappedBy = "user")
private UserProfile profile;
```

#### cascade
Определяет типы каскадирования:
```java
@OneToOne(mappedBy = "user", cascade = CascadeType.PERSIST)
private UserProfile profile;
```

#### fetch
Определяет стратегию загрузки:
```java
@OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
private UserProfile profile;
```

#### optional
Указывает, может ли связь быть null:
```java
@OneToOne(mappedBy = "user", optional = false)
private UserProfile profile;
```

---

## <a name="двунаправленные"></a>Двунаправленные связи

### Принцип работы

В двунаправленной связи обе сущности имеют ссылки друг на друга:
- **Родительская сущность**: Содержит ссылку на дочернюю сущность
- **Дочерняя сущность**: Содержит ссылку на родительскую сущность и является владельцем связи

### Правила управления связью

1. **Владелец связи**: Дочерняя сущность является владельцем связи
2. **Синхронизация**: Изменения должны быть синхронизированы с обеих сторон
3. **Методы управления**: Используйте специальные методы для управления связью

### Пример двунаправленной связи

```java
@Entity
public class User {
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL)
    private UserProfile profile;
    
    // Методы для управления связью
    public void setProfile(UserProfile profile) {
        this.profile = profile;
        if (profile != null) {
            profile.setUser(this);
        }
    }
    
    public void removeProfile() {
        if (this.profile != null) {
            this.profile.setUser(null);
            this.profile = null;
        }
    }
}

@Entity
public class UserProfile {
    @OneToOne
    @JoinColumn(name = "user_id", nullable = false, unique = true)
    private User user;
    
    // Геттеры и сеттеры
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}
```

### Использование

```java
User user = new User("john_doe", "john@example.com");
UserProfile profile = new UserProfile("John", "Doe");
profile.setPhone("+1234567890");

// Правильное управление связью
user.setProfile(profile);

em.persist(user); // Сохраняются обе сущности
```

---

## <a name="тестирование"></a>Тестирование связи

### Базовое тестирование

```java
import javax.persistence.*;
import java.time.LocalDate;
import java.util.List;

public class OneToOneTest {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("pu");
        EntityManager em = emf.createEntityManager();
        
        try {
            em.getTransaction().begin();
            
            // Создание пользователя
            User user = new User("john_doe", "john@example.com");
            em.persist(user);
            
            // Создание профиля
            UserProfile profile = new UserProfile("John", "Doe");
            profile.setPhone("+1234567890");
            profile.setAddress("123 Main St, City");
            profile.setBirthDate(LocalDate.of(1990, 5, 15));
            profile.setUser(user);
            em.persist(profile);
            
            em.getTransaction().commit();
            
            // Проверка связи
            User foundUser = em.find(User.class, user.getId());
            System.out.println("Пользователь: " + foundUser);
            System.out.println("Профиль: " + foundUser.getProfile());
            
            UserProfile foundProfile = em.find(UserProfile.class, profile.getId());
            System.out.println("Профиль: " + foundProfile);
            System.out.println("Пользователь профиля: " + foundProfile.getUser());
            
            // Поиск пользователей с профилями
            List<User> usersWithProfiles = em.createQuery(
                "SELECT u FROM User u JOIN FETCH u.profile", User.class)
                .getResultList();
            
            System.out.println("Пользователи с профилями:");
            usersWithProfiles.forEach(u -> {
                System.out.println("User: " + u.getUsername() + 
                                 ", Profile: " + u.getProfile().getFirstName() + 
                                 " " + u.getProfile().getLastName());
            });
            
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
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL)
    private UserProfile profile;
    
    // ... остальной код ...
}

// Тестирование
public class CascadeTest {
    public static void main(String[] args) {
        EntityManager em = // получение EntityManager
        
        em.getTransaction().begin();
        
        User user = new User("jane_doe", "jane@example.com");
        UserProfile profile = new UserProfile("Jane", "Doe");
        user.setProfile(profile);
        
        // Сохраняется только user, profile сохраняется автоматически
        em.persist(user);
        
        // Удаление профиля
        user.removeProfile();
        // Profile будет удален из БД благодаря cascade = CascadeType.ALL
        
        em.getTransaction().commit();
    }
}
```

---

## <a name="примеры"></a>Практические примеры

### Пример 1: Сотрудник и рабочее место

```java
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
    
    @OneToOne(mappedBy = "employee", cascade = CascadeType.ALL)
    private Workstation workstation;
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "workstations")
public class Workstation {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "computer_id", nullable = false, unique = true)
    private String computerId;
    
    @Column(name = "location")
    private String location;
    
    @OneToOne
    @JoinColumn(name = "employee_id", unique = true)
    private Employee employee;
    
    // Геттеры и сеттеры
}
```

### Пример 2: Заказ и доставка

```java
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
    
    @OneToOne(mappedBy = "order", cascade = CascadeType.ALL)
    private Delivery delivery;
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "deliveries")
public class Delivery {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "tracking_number", nullable = false, unique = true)
    private String trackingNumber;
    
    @Column(name = "delivery_address", nullable = false)
    private String deliveryAddress;
    
    @Column(name = "estimated_delivery_date")
    private LocalDate estimatedDeliveryDate;
    
    @OneToOne
    @JoinColumn(name = "order_id", nullable = false, unique = true)
    private Order order;
    
    // Геттеры и сеттеры
}
```

### Пример 3: Студент и диплом

```java
@Entity
@Table(name = "students")
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "student_number", nullable = false, unique = true)
    private String studentNumber;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @OneToOne(mappedBy = "student", cascade = CascadeType.ALL)
    private Diploma diploma;
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "diplomas")
public class Diploma {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "diploma_number", nullable = false, unique = true)
    private String diplomaNumber;
    
    @Column(name = "specialization", nullable = false)
    private String specialization;
    
    @Column(name = "graduation_date")
    private LocalDate graduationDate;
    
    @Column(name = "grade")
    private String grade;
    
    @OneToOne
    @JoinColumn(name = "student_id", nullable = false, unique = true)
    private Student student;
    
    // Геттеры и сеттеры
}
```

### Пример 4: Автомобиль и владелец

```java
@Entity
@Table(name = "cars")
public class Car {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "license_plate", nullable = false, unique = true)
    private String licensePlate;
    
    @Column(name = "model", nullable = false)
    private String model;
    
    @Column(name = "year")
    private Integer year;
    
    @OneToOne(mappedBy = "car", cascade = CascadeType.ALL)
    private Owner owner;
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "owners")
public class Owner {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "phone", nullable = false)
    private String phone;
    
    @Column(name = "address")
    private String address;
    
    @OneToOne
    @JoinColumn(name = "car_id", nullable = false, unique = true)
    private Car car;
    
    // Геттеры и сеттеры
}
```

---

## <a name="практики"></a>Лучшие практики

### 1. Правильное управление двунаправленными связями

```java
@Entity
public class User {
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL)
    private UserProfile profile;
    
    public void setProfile(UserProfile profile) {
        this.profile = profile;
        if (profile != null) {
            profile.setUser(this);
        }
    }
    
    public void removeProfile() {
        if (this.profile != null) {
            this.profile.setUser(null);
            this.profile = null;
        }
    }
}
```

### 2. Использование уникальных ограничений

```java
@OneToOne
@JoinColumn(name = "user_id", nullable = false, unique = true)
private User user;
```

### 3. Оптимизация загрузки

```java
// Использование JOIN FETCH для избежания N+1 проблем
List<User> users = em.createQuery(
    "SELECT u FROM User u JOIN FETCH u.profile", User.class)
    .getResultList();
```

### 4. Валидация данных

```java
@OneToOne(optional = false)
@JoinColumn(name = "user_id", nullable = false, unique = true)
private User user;

// В коде
public void setUser(User user) {
    if (user == null) {
        throw new IllegalArgumentException("User cannot be null");
    }
    this.user = user;
}
```

### 5. Обработка исключений

```java
try {
    User user = new User("john_doe", "john@example.com");
    UserProfile profile = new UserProfile("John", "Doe");
    user.setProfile(profile);
    em.persist(user);
} catch (Exception e) {
    // Обработка ошибок
    em.getTransaction().rollback();
    e.printStackTrace();
}
```

### 6. Использование индексов

```sql
-- Создание индекса для внешнего ключа
CREATE INDEX idx_user_profiles_user_id ON user_profiles(user_id);
```

### 7. Выбор стратегии загрузки

```java
// Для часто используемых связей
@OneToOne(mappedBy = "user", fetch = FetchType.EAGER)
private UserProfile profile;

// Для редко используемых связей
@OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
private UserProfile profile;
```

---

**Связанные разделы:**
- [01_Основы_ассоциаций.md](01_Основы_ассоциаций.md) - Основы ассоциаций
- [02_ManyToOne_связи.md](02_ManyToOne_связи.md) - Связи "многие к одному"
- [03_OneToMany_связи.md](03_OneToMany_связи.md) - Связи "один ко многим"
- [06_Управление_загрузкой.md](06_Управление_загрузкой.md) - Управление загрузкой данных
- [07_Каскадирование.md](07_Каскадирование.md) - Каскадирование операций 