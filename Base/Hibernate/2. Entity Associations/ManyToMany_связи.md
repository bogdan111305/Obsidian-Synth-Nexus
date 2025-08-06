# ManyToMany - Связь "многие ко многим"

## Оглавление
1. [Введение в ManyToMany](#введение)
2. [Создание DDL и сущностей](#ddl-сущности)
3. [Аннотация @ManyToMany](#аннотация)
4. [Промежуточная таблица](#промежуточная-таблица)
5. [Двунаправленные связи](#двунаправленные)
6. [Тестирование связи](#тестирование)
7. [Практические примеры](#примеры)
8. [Лучшие практики](#практики)

---

## <a name="введение"></a>Введение в ManyToMany

Связь "многие ко многим" (`@ManyToMany`) используется, когда множество сущностей связаны с множеством других сущностей. Например, множество пользователей (`User`) могут иметь множество ролей (`Role`). Эта связь реализуется через промежуточную таблицу.

### Применение ManyToMany

- **Пользователи и роли**: Множество пользователей могут иметь множество ролей
- **Студенты и курсы**: Множество студентов могут посещать множество курсов
- **Продукты и категории**: Множество продуктов могут принадлежать множеству категорий
- **Авторы и книги**: Множество авторов могут написать множество книг
- **Теги и посты**: Множество тегов могут быть связаны с множеством постов

### Принцип работы

1. **Промежуточная таблица**: Создается отдельная таблица для хранения связей
2. **Внешние ключи**: В промежуточной таблице создаются два внешних ключа
3. **Коллекции**: В обеих сущностях используются коллекции для хранения связанных объектов

---

## <a name="ddl-сущности"></a>Создание DDL и сущностей

### Создание таблиц

```sql
-- Таблица пользователей
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица ролей
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Промежуточная таблица для связи многие-ко-многим
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
);

-- Индексы для улучшения производительности
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_user_roles_role_id ON user_roles(role_id);
```

### Сущность User

```java
import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

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
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
    
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
    
    public Set<Role> getRoles() { return roles; }
    public void setRoles(Set<Role> roles) { this.roles = roles; }
    
    // Методы для управления связью
    public void addRole(Role role) {
        roles.add(role);
        role.getUsers().add(this);
    }
    
    public void removeRole(Role role) {
        roles.remove(role);
        role.getUsers().remove(this);
    }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", username='" + username + "', roles=" + roles.size() + "}";
    }
}
```

### Сущность Role

```java
import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "roles")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, unique = true)
    private String name;
    
    @Column(name = "description", columnDefinition = "TEXT")
    private String description;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @ManyToMany(mappedBy = "roles", fetch = FetchType.LAZY)
    private Set<User> users = new HashSet<>();
    
    // Конструкторы
    public Role() {}
    
    public Role(String name, String description) {
        this.name = name;
        this.description = description;
        this.createdAt = LocalDateTime.now();
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public Set<User> getUsers() { return users; }
    public void setUsers(Set<User> users) { this.users = users; }
    
    @Override
    public String toString() {
        return "Role{id=" + id + ", name='" + name + "', users=" + users.size() + "}";
    }
}
```

---

## <a name="аннотация"></a>Аннотация @ManyToMany

Аннотация `@ManyToMany` указывает, что поле сущности содержит коллекцию связанных сущностей. Она используется с `@JoinTable` для настройки промежуточной таблицы.

### Базовая реализация

```java
@ManyToMany
@JoinTable(
    name = "user_roles",
    joinColumns = @JoinColumn(name = "user_id"),
    inverseJoinColumns = @JoinColumn(name = "role_id")
)
private Set<Role> roles = new HashSet<>();
```

### Расширенная настройка

```java
@ManyToMany(
    fetch = FetchType.LAZY,                    // Ленивая загрузка
    cascade = CascadeType.PERSIST              // Каскадирование сохранения
)
@JoinTable(
    name = "user_roles",                       // Имя промежуточной таблицы
    joinColumns = @JoinColumn(name = "user_id"),      // Столбец для текущей сущности
    inverseJoinColumns = @JoinColumn(name = "role_id"), // Столбец для связанной сущности
    uniqueConstraints = @UniqueConstraint(     // Уникальные ограничения
        columnNames = {"user_id", "role_id"}
    )
)
private Set<Role> roles = new HashSet<>();
```

### Параметры аннотации

#### fetch
Определяет стратегию загрузки:
```java
@ManyToMany(fetch = FetchType.LAZY)
private Set<Role> roles;
```

#### cascade
Определяет типы каскадирования:
```java
@ManyToMany(cascade = CascadeType.PERSIST)
private Set<Role> roles;
```

#### mappedBy
Указывает поле в связанной сущности (для двунаправленных связей):
```java
@ManyToMany(mappedBy = "roles")
private Set<User> users;
```

---

## <a name="промежуточная-таблица"></a>Промежуточная таблица

### Структура промежуточной таблицы

Промежуточная таблица содержит:
- **Внешние ключи**: Ссылки на обе связанные таблицы
- **Составной первичный ключ**: Комбинация внешних ключей
- **Дополнительные поля**: Метаданные связи (дата создания, статус и т.д.)

### Пример с дополнительными полями

```sql
-- Промежуточная таблица с дополнительными полями
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_by VARCHAR(255),
    is_active BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
);
```

### Настройка @JoinTable

```java
@ManyToMany
@JoinTable(
    name = "user_roles",
    joinColumns = @JoinColumn(name = "user_id"),
    inverseJoinColumns = @JoinColumn(name = "role_id"),
    uniqueConstraints = @UniqueConstraint(
        columnNames = {"user_id", "role_id"}
    )
)
private Set<Role> roles = new HashSet<>();
```

### Параметры @JoinTable

#### name
Имя промежуточной таблицы:
```java
@JoinTable(name = "user_roles")
```

#### joinColumns
Столбец для текущей сущности:
```java
joinColumns = @JoinColumn(name = "user_id")
```

#### inverseJoinColumns
Столбец для связанной сущности:
```java
inverseJoinColumns = @JoinColumn(name = "role_id")
```

#### uniqueConstraints
Уникальные ограничения:
```java
uniqueConstraints = @UniqueConstraint(
    columnNames = {"user_id", "role_id"}
)
```

---

## <a name="двунаправленные"></a>Двунаправленные связи

### Принцип работы

В двунаправленной связи обе сущности имеют коллекции связанных объектов:
- **Первая сущность**: Содержит коллекцию связанных сущностей с `@JoinTable`
- **Вторая сущность**: Содержит коллекцию связанных сущностей с `mappedBy`

### Правила управления связью

1. **Владелец связи**: Сущность с `@JoinTable` является владельцем связи
2. **Синхронизация**: Изменения должны быть синхронизированы с обеих сторон
3. **Методы управления**: Используйте специальные методы для управления связью

### Пример двунаправленной связи

```java
@Entity
public class User {
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
    
    // Методы для управления связью
    public void addRole(Role role) {
        roles.add(role);
        role.getUsers().add(this);
    }
    
    public void removeRole(Role role) {
        roles.remove(role);
        role.getUsers().remove(this);
    }
}

@Entity
public class Role {
    @ManyToMany(mappedBy = "roles", fetch = FetchType.LAZY)
    private Set<User> users = new HashSet<>();
    
    // Геттеры и сеттеры
    public Set<User> getUsers() { return users; }
    public void setUsers(Set<User> users) { this.users = users; }
}
```

### Использование

```java
User user = new User("john_doe", "john@example.com");
Role adminRole = new Role("ADMIN", "Administrator");
Role userRole = new Role("USER", "Regular user");

// Правильное управление связью
user.addRole(adminRole);
user.addRole(userRole);

em.persist(user);
em.persist(adminRole);
em.persist(userRole);
```

---

## <a name="тестирование"></a>Тестирование связи

### Базовое тестирование

```java
import javax.persistence.*;
import java.util.List;
import java.util.Set;

public class ManyToManyTest {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("pu");
        EntityManager em = emf.createEntityManager();
        
        try {
            em.getTransaction().begin();
            
            // Создание ролей
            Role adminRole = new Role("ADMIN", "Administrator");
            Role userRole = new Role("USER", "Regular user");
            Role moderatorRole = new Role("MODERATOR", "Moderator");
            
            em.persist(adminRole);
            em.persist(userRole);
            em.persist(moderatorRole);
            
            // Создание пользователей
            User user1 = new User("john_doe", "john@example.com");
            User user2 = new User("jane_smith", "jane@example.com");
            User user3 = new User("bob_wilson", "bob@example.com");
            
            // Назначение ролей
            user1.addRole(adminRole);
            user1.addRole(userRole);
            
            user2.addRole(userRole);
            user2.addRole(moderatorRole);
            
            user3.addRole(userRole);
            
            em.persist(user1);
            em.persist(user2);
            em.persist(user3);
            
            em.getTransaction().commit();
            
            // Проверка связи
            User foundUser = em.find(User.class, user1.getId());
            System.out.println("Пользователь: " + foundUser.getUsername());
            System.out.println("Роли:");
            foundUser.getRoles().forEach(role -> 
                System.out.println("  - " + role.getName() + ": " + role.getDescription())
            );
            
            Role foundRole = em.find(Role.class, adminRole.getId());
            System.out.println("Роль: " + foundRole.getName());
            System.out.println("Пользователи с этой ролью:");
            foundRole.getUsers().forEach(u -> 
                System.out.println("  - " + u.getUsername())
            );
            
            // Поиск пользователей по роли
            List<User> admins = em.createQuery(
                "SELECT u FROM User u JOIN u.roles r WHERE r.name = :roleName", User.class)
                .setParameter("roleName", "ADMIN")
                .getResultList();
            
            System.out.println("Администраторы:");
            admins.forEach(u -> System.out.println("  - " + u.getUsername()));
            
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
    @ManyToMany(cascade = CascadeType.PERSIST)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
    
    // ... остальной код ...
}

// Тестирование
public class CascadeTest {
    public static void main(String[] args) {
        EntityManager em = // получение EntityManager
        
        em.getTransaction().begin();
        
        User user = new User("alice", "alice@example.com");
        Role newRole = new Role("EDITOR", "Content editor");
        user.addRole(newRole);
        
        // Сохраняется только user, role сохраняется автоматически
        em.persist(user);
        
        // Удаление роли
        user.removeRole(newRole);
        // Role остается в БД, удаляется только связь
        
        em.getTransaction().commit();
    }
}
```

---

## <a name="примеры"></a>Практические примеры

### Пример 1: Студенты и курсы

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
    
    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
    
    // Методы управления связью
    public void addCourse(Course course) {
        courses.add(course);
        course.getStudents().add(this);
    }
    
    public void removeCourse(Course course) {
        courses.remove(course);
        course.getStudents().remove(this);
    }
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "courses")
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "code", nullable = false, unique = true)
    private String code;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "credits")
    private Integer credits;
    
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
    
    // Геттеры и сеттеры
}
```

### Пример 2: Продукты и категории

```java
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
    
    @ManyToMany
    @JoinTable(
        name = "product_categories",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "category_id")
    )
    private Set<Category> categories = new HashSet<>();
    
    // Методы управления связью
    public void addCategory(Category category) {
        categories.add(category);
        category.getProducts().add(this);
    }
    
    public void removeCategory(Category category) {
        categories.remove(category);
        category.getProducts().remove(this);
    }
    
    // Геттеры и сеттеры
}

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
    
    @ManyToMany(mappedBy = "categories")
    private Set<Product> products = new HashSet<>();
    
    // Геттеры и сеттеры
}
```

### Пример 3: Авторы и книги

```java
@Entity
@Table(name = "authors")
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "biography", columnDefinition = "TEXT")
    private String biography;
    
    @ManyToMany
    @JoinTable(
        name = "author_books",
        joinColumns = @JoinColumn(name = "author_id"),
        inverseJoinColumns = @JoinColumn(name = "book_id")
    )
    private Set<Book> books = new HashSet<>();
    
    // Методы управления связью
    public void addBook(Book book) {
        books.add(book);
        book.getAuthors().add(this);
    }
    
    public void removeBook(Book book) {
        books.remove(book);
        book.getAuthors().remove(this);
    }
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "books")
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "title", nullable = false)
    private String title;
    
    @Column(name = "isbn", unique = true)
    private String isbn;
    
    @Column(name = "publication_year")
    private Integer publicationYear;
    
    @ManyToMany(mappedBy = "books")
    private Set<Author> authors = new HashSet<>();
    
    // Геттеры и сеттеры
}
```

### Пример 4: Теги и посты

```java
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
    
    @ManyToMany
    @JoinTable(
        name = "post_tags",
        joinColumns = @JoinColumn(name = "post_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();
    
    // Методы управления связью
    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getPosts().add(this);
    }
    
    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getPosts().remove(this);
    }
    
    // Геттеры и сеттеры
}

@Entity
@Table(name = "tags")
public class Tag {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false, unique = true)
    private String name;
    
    @Column(name = "description")
    private String description;
    
    @ManyToMany(mappedBy = "tags")
    private Set<Post> posts = new HashSet<>();
    
    // Геттеры и сеттеры
}
```

---

## <a name="практики"></a>Лучшие практики

### 1. Правильное управление двунаправленными связями

```java
@Entity
public class User {
    @ManyToMany
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
    
    public void addRole(Role role) {
        roles.add(role);
        role.getUsers().add(this);
    }
    
    public void removeRole(Role role) {
        roles.remove(role);
        role.getUsers().remove(this);
    }
}
```

### 2. Использование Set вместо List

```java
// Предпочтительно использовать Set для избежания дублирования
@ManyToMany
@JoinTable(name = "user_roles")
private Set<Role> roles = new HashSet<>();

// Избегайте List для ManyToMany связей
// @ManyToMany
// private List<Role> roles = new ArrayList<>();
```

### 3. Оптимизация загрузки

```java
// Использование JOIN FETCH для избежания N+1 проблем
List<User> users = em.createQuery(
    "SELECT u FROM User u JOIN FETCH u.roles", User.class)
    .getResultList();
```

### 4. Использование индексов

```sql
-- Создание индексов для промежуточной таблицы
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_user_roles_role_id ON user_roles(role_id);
```

### 5. Обработка исключений

```java
try {
    User user = new User("john_doe", "john@example.com");
    Role role = new Role("ADMIN", "Administrator");
    user.addRole(role);
    em.persist(user);
    em.persist(role);
} catch (Exception e) {
    // Обработка ошибок
    em.getTransaction().rollback();
    e.printStackTrace();
}
```

### 6. Выбор стратегии загрузки

```java
// Для больших коллекций используйте LAZY
@ManyToMany(fetch = FetchType.LAZY)
private Set<Role> roles = new HashSet<>();

// Для небольших коллекций можно использовать EAGER
@ManyToMany(fetch = FetchType.EAGER)
private Set<Role> roles = new HashSet<>();
```

### 7. Уникальные ограничения

```java
@ManyToMany
@JoinTable(
    name = "user_roles",
    joinColumns = @JoinColumn(name = "user_id"),
    inverseJoinColumns = @JoinColumn(name = "role_id"),
    uniqueConstraints = @UniqueConstraint(
        columnNames = {"user_id", "role_id"}
    )
)
private Set<Role> roles = new HashSet<>();
```

---

**Связанные разделы:**
- [01_Основы_ассоциаций.md](01_Основы_ассоциаций.md) - Основы ассоциаций
- [02_ManyToOne_связи.md](02_ManyToOne_связи.md) - Связи "многие к одному"
- [03_OneToMany_связи.md](03_OneToMany_связи.md) - Связи "один ко многим"
- [04_OneToOne_связи.md](04_OneToOne_связи.md) - Связи "один к одному"
- [06_Управление_загрузкой.md](06_Управление_загрузкой.md) - Управление загрузкой данных
- [07_Каскадирование.md](07_Каскадирование.md) - Каскадирование операций 