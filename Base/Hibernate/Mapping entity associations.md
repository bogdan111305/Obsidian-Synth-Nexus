# Ассоциации сущностей в Hibernate

Ассоциации сущностей в Hibernate позволяют моделировать отношения между объектами в Java-приложении и соответствующими таблицами в базе данных. Они охватывают связи типа "многие к одному" (`@ManyToOne`), "один ко многим" (`@OneToMany`), "один к одному" (`@OneToOne`) и "многие ко многим" (`@ManyToMany`), а также управление коллекциями, их сортировкой и производительностью. Эта статья подробно описывает механизмы маппинга ассоциаций, включая аннотации, типы выборки, каскадирование, обработку ошибок и оптимизацию работы с коллекциями.
# ManyToOne

Связь "многие к одному" (`@ManyToOne`) используется, когда несколько сущностей ссылаются на одну общую сущность. Например, множество пользователей (`User`) могут быть связаны с одной компанией (`Company`). Эта связь реализуется через внешний ключ в таблице дочерней сущности.

## Создание DDL и сущности Company

Для работы с `@ManyToOne` создадим таблицу `companies` для хранения компаний и таблицу `users` для пользователей с внешним ключом `company_id`.

```sql
CREATE TABLE companies (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    company_id BIGINT,
    FOREIGN KEY (company_id) REFERENCES companies(id)
);
```

Сущность `Company` представляет компанию с первичным ключом и названием:

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;

@Entity
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

- **`id`**: Первичный ключ с автоинкрементом (`GenerationType.IDENTITY` соответствует PostgreSQL `SERIAL`).
- **`name`**: Название компании, хранимое в столбце `name`.

## Аннотация @ManyToOne

Аннотация `@ManyToOne` указывает, что поле сущности ссылается на другую сущность через внешний ключ. Она часто используется с аннотацией `@JoinColumn`, которая задаёт имя столбца внешнего ключа в таблице.

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.ManyToOne;
import javax.persistence.JoinColumn;

@Entity
public class User {
    @Id
    private Long id;
    private String name;
    @ManyToOne
    @JoinColumn(name = "company_id")
    private Company company;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Company getCompany() { return company; }
    public void setCompany(Company company) { this.company = company; }
}
```

В этом примере поле `company` в сущности `User` связано с сущностью `Company` через столбец `company_id`. Аннотация `@JoinColumn` явно указывает имя столбца внешнего ключа, что делает маппинг более прозрачным.

## Тестирование новой сущности

Тестирование связи `@ManyToOne` включает создание, сохранение и извлечение сущностей для проверки корректности маппинга.

```java
import javax.persistence.EntityManager;
import javax.persistence.EntityManagerFactory;
import javax.persistence.Persistence;

public class ManyToOneTest {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("pu");
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();

        // Создание компании
        Company company = new Company();
        company.setName("TechCorp");
        em.persist(company);

        // Создание пользователя
        User user = new User();
        user.setId(1L);
        user.setName("John");
        user.setCompany(company);
        em.persist(user);

        em.getTransaction().commit();

        // Проверка
        User foundUser = em.find(User.class, 1L);
        System.out.println("User: " + foundUser.getName() + ", Company: " + foundUser.getCompany().getName());
        em.close();
        emf.close();
    }
}
```

Этот код создаёт компанию, связывает её с пользователем, сохраняет их в базе данных и проверяет, что связь корректно сохраняется и извлекается. Вывод показывает имя пользователя и название связанной компании.

## Класс ManyToOneType

`ManyToOneType` — это внутренний класс Hibernate, который управляет маппингом связи `@ManyToOne`. Он отвечает за:

- Определение внешнего ключа для связи.
- Обработку стратегий выборки (ленивой или немедленной).
- Управление каскадированием операций (например, сохранение или удаление).  
    Разработчикам редко приходится взаимодействовать с `ManyToOneType` напрямую, но понимание его роли полезно при отладке сложных сценариев, таких как проблемы с производительностью или неожиданные запросы к базе данных.

---

# Fetch Types

Тип выборки (`fetch`) определяет, когда связанные данные загружаются из базы данных. Hibernate поддерживает два типа выборки: `EAGER` и `LAZY`.

## Optional

Параметр `optional` в аннотации `@ManyToOne` указывает, может ли связанная сущность быть `null`:

- `optional = true` (по умолчанию): Связь может быть `null`, то есть пользователь может не принадлежать никакой компании.
- `optional = false`: Связь обязательна, и Hibernate будет ожидать, что внешний ключ всегда указывает на существующую запись в таблице `companies`.

```java
@ManyToOne(optional = false)
@JoinColumn(name = "company_id")
private Company company;
```

Если `optional = false`, попытка сохранить `User` с `company = null` вызовет исключение `ConstraintViolationException`.

## Fetch Type Lazy и Proxy объекты

Параметр `fetch` в `@ManyToOne` определяет стратегию загрузки связанных данных:

- `FetchType.EAGER` (по умолчанию для `@ManyToOne`): Связанная сущность (`Company`) загружается сразу при загрузке основной сущности (`User`).
- `FetchType.LAZY`: Связанная сущность загружается только при первом обращении к ней (например, вызов `user.getCompany().getName()`).

При использовании `FetchType.LAZY` Hibernate создаёт **прокси-объект** для связанной сущности, чтобы отложить загрузку данных из базы до момента их фактического использования.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "company_id")
private Company company;
```

Ленивая загрузка снижает количество запросов к базе данных, что улучшает производительность, особенно при работе с большими наборами данных.

---

# Hibernate Proxy

Hibernate использует прокси-объекты для реализации ленивой загрузки, что позволяет избежать ненужных запросов к базе данных.

## Dynamic Proxy

Hibernate создаёт прокси-объекты, которые выступают как "заглушки" для реальных сущностей. Эти прокси могут быть основаны на:

- Интерфейсах (если сущность реализует интерфейс).
- Классах (если сущность — обычный класс).

Прокси перехватывает вызовы методов и загружает данные из базы только при необходимости.

## Proxy через наследование (ByteBuddy)

Начиная с Hibernate 5.2, для создания прокси-объектов используется библиотека **ByteBuddy**. Она генерирует подкласс сущности, который перехватывает вызовы методов для выполнения ленивой загрузки. Это позволяет Hibernate работать с классами, не реализующими интерфейсы.

## ByteBuddyInterceptor

`ByteBuddyInterceptor` — внутренний механизм Hibernate, который обрабатывает вызовы методов прокси-объекта. Он определяет, когда нужно загрузить данные из базы, и передаёт управление реальной сущности. Разработчикам не нужно напрямую взаимодействовать с `ByteBuddyInterceptor`, но понимание его роли помогает при отладке проблем с ленивой загрузкой.

## Инициализация Proxy объекта

Прокси-объект остаётся неинициализированным, пока не будет вызван метод, требующий данных (например, `getName()`). При первом обращении Hibernate выполняет SQL-запрос для загрузки данных сущности.

## Утилитный класс Hibernate

Утилитный класс `org.hibernate.Hibernate` предоставляет методы для работы с прокси-объектами:

- `Hibernate.initialize(Object)`: Принудительно инициализирует прокси или коллекцию, выполняя запрос к базе.
- `Hibernate.isInitialized(Object)`: Проверяет, инициализирован ли объект или коллекция.

```java
User user = em.find(User.class, 1L);
Hibernate.initialize(user.getCompany()); // Принудительная инициализация
if (Hibernate.isInitialized(user.getCompany())) {
    System.out.println(user.getCompany().getName());
}
```

Этот код гарантирует, что связанная сущность `Company` будет загружена в рамках активной сессии, избегая `LazyInitializationException`.

---

# Cascade Types

Каскадирование определяет, какие операции (например, сохранение, удаление) применяются к связанным сущностям автоматически.

## CascadeType enum

Перечисление `CascadeType` включает следующие значения:

- `PERSIST`: Сохранение родительской сущности сохраняет связанные сущности.
- `MERGE`: Обновление родительской сущности обновляет связанные сущности.
- `REMOVE`: Удаление родительской сущности удаляет связанные сущности.
- `DETACH`: Отсоединение родительской сущности от сессии отсоединяет связанные сущности.
- `REFRESH`: Обновление родительской сущности из базы обновляет связанные сущности.
- `ALL`: Включает все вышеперечисленные операции.

## Тестирование метода evict (CascadeType.DETACH)

`CascadeType.DETACH` отсоединяет связанную сущность от сессии Hibernate, если родительская сущность отсоединяется.

```java
@Entity
public class User {
    @Id
    private Long id;
    @ManyToOne(cascade = CascadeType.DETACH)
    @JoinColumn(name = "company_id")
    private Company company;
    // Геттеры и сеттеры
}

em.getTransaction().begin();
User user = em.find(User.class, 1L);
em.detach(user); // Отсоединяет user и связанную company
em.getTransaction().commit();
```

После вызова `em.detach(user)`, сущность `Company`, связанная с `user`, также становится отсоединённой, и дальнейшие изменения не синхронизируются с базой.

## Тестирование метода save (CascadeType.PERSIST)

`CascadeType.PERSIST` автоматически сохраняет связанную сущность при сохранении родительской.

```java
@Entity
public class User {
    @Id
    private Long id;
    @ManyToOne(cascade = CascadeType.PERSIST)
    @JoinColumn(name = "company_id")
    private Company company;
    // Геттеры и сеттеры
}

em.getTransaction().begin();
Company company = new Company();
company.setName("NewCorp");
User user = new User();
user.setId(1L);
user.setName("John");
user.setCompany(company);
em.persist(user); // Сохраняет user и company
em.getTransaction().commit();
```

В этом примере `em.persist(user)` сохраняет как `User`, так и связанную `Company`, даже если последняя не была явно сохранена.

## CascadeType.ALL

`CascadeType.ALL` включает все операции каскадирования, что удобно, но требует осторожности, так как может привести к неожиданным удалениям или обновлениям.

```java
@ManyToOne(cascade = CascadeType.ALL)
@JoinColumn(name = "company_id")
private Company company;
```

---

# OneToMany

Связь "один ко многим" (`@OneToMany`) позволяет одной сущности (например, `Company`) быть связанной с несколькими другими (например, `User`).

## Unidirectional @OneToMany (users в Company)

Однонаправленная связь не требует обратной ссылки в дочерней сущности. Она реализуется через внешний ключ в таблице дочерней сущности.

```java
@Entity
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    @OneToMany
    @JoinColumn(name = "company_id")
    private List<User> users = new ArrayList<>();

    // Геттеры и сеттеры
}
```

Здесь коллекция `users` в `Company` мапится через внешний ключ `company_id` в таблице `users`.

## Bidirectional @OneToMany (users в Company)

Двунаправленная связь включает обратную ссылку в дочерней сущности с помощью `@ManyToOne`. Это позволяет синхронизировать данные с обеих сторон.

```java
@Entity
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    @OneToMany(mappedBy = "company")
    private List<User> users = new ArrayList<>();

    public void addUser(User user) {
        users.add(user);
        user.setCompany(this); // Синхронизация обратной связи
    }

    // Геттеры и сеттеры
}

@Entity
public class User {
    @Id
    private Long id;
    private String name;
    @ManyToOne
    @JoinColumn(name = "company_id")
    private Company company;

    // Геттеры и сеттеры
}
```

Метод `addUser` синхронизирует обе стороны связи, что предотвращает несогласованность данных.

## Тестирование @OneToMany

Тестирование проверяет корректность сохранения и извлечения коллекции.

```java
em.getTransaction().begin();
Company company = new Company();
company.setName("TechCorp");
User user = new User();
user.setId(1L);
user.setName("John");
company.addUser(user);
em.persist(company);
em.getTransaction().commit();

Company found = em.find(Company.class, company.getId());
assertEquals(1, found.getUsers().size());
assertEquals("John", found.getUsers().get(0).getName());
```

Этот код создаёт компанию, добавляет пользователя в коллекцию `users` и проверяет, что данные корректно сохраняются и извлекаются.

## StackOverflowError в bidirectional связи

В двунаправленных связях `StackOverflowError` может возникнуть из-за циклических вызовов в методах `toString()`, `equals()` или `hashCode()`.

```java
@Entity
public class Company {
    @OneToMany(mappedBy = "company")
    private List<User> users;

    @Override
    public String toString() {
        return "Company{users=" + users + "}"; // Циклический вызов
    }
}

@Entity
public class User {
    @ManyToOne
    private Company company;

    @Override
    public String toString() {
        return "User{company=" + company + "}"; // Циклический вызов
    }
}
```

**Решение**:

- Исключите связанные поля из `toString()`, `equals()` и `hashCode()`.
- Используйте аннотацию `@ToString.Exclude` (если используется Lombok).
- Реализуйте методы с использованием нециклических полей, таких как `name`.

```java
@Override
public String toString() {
    return "Company{name=" + name + "}";
}
```

---

# Cascade Types with Collections

Каскадирование в коллекциях определяет, как операции с родительской сущностью влияют на элементы коллекции.

## Декартово произведение

При использовании `FetchType.EAGER` и SQL `JOIN` для коллекций Hibernate может создать декартово произведение (все возможные комбинации строк), что увеличивает объём возвращаемых данных и снижает производительность. Например, если компания имеет 100 пользователей, а каждый пользователь связан с несколькими другими таблицами, запрос может вернуть избыточное количество строк.

**Решение**:

- Используйте `FetchType.LAZY` для коллекций.
- Применяйте `JOIN FETCH` в JPQL-запросах только при необходимости.

## Вспомогательный метод addUser

Для поддержания целостности двунаправленной связи полезно использовать вспомогательные методы, которые синхронизируют обе стороны.

```java
public class Company {
    @OneToMany(mappedBy = "company")
    private List<User> users = new ArrayList<>();

    public void addUser(User user) {
        users.add(user);
        user.setCompany(this); // Синхронизация обратной связи
    }
}
```

Этот метод добавляет пользователя в коллекцию `users` и устанавливает обратную ссылку на компанию.

## Cascade types при удалении сущности

`CascadeType.REMOVE` удаляет все связанные сущности при удалении родительской.

```java
@OneToMany(mappedBy = "company", cascade = CascadeType.REMOVE)
private List<User> users;
```

```java
em.getTransaction().begin();
Company company = em.find(Company.class, 1L);
em.remove(company); // Удаляет компанию и всех связанных пользователей
em.getTransaction().commit();
```

Здесь удаление компании приводит к удалению всех пользователей, связанных через коллекцию `users`.

---

# Entity equals and hashCode

Методы `equals()` и `hashCode()` критически важны для корректной работы сущностей, особенно при использовании их в коллекциях (`Set`, `Map`).

## Почему не следует использовать id для equals and hashCode

Использование первичного ключа (`id`) в `equals()` и `hashCode()` проблематично:

- Для новых сущностей (ещё не сохранённых) `id` может быть `null`, что нарушает контракт `equals()`.
- Если `id` меняется (например, при сохранении), хэш-код сущности изменится, что может нарушить целостность коллекций, таких как `HashSet`.

## Почему следует использовать натуральный id для equals and hashCode

Натуральный ключ — это уникальное бизнес-поле (например, `email` или `username`), которое:

- Остаётся неизменным в течение жизненного цикла сущности.
- Не зависит от состояния сущности (новая или сохранённая).
- Обеспечивает стабильное сравнение объектов.

## Что если нет натурального id?

Если натурального ключа нет, возможны следующие подходы:

- Использовать все поля сущности для сравнения в `equals()` и `hashCode()`.
- Полагаться на стандартное поведение `Object.equals()` и `Object.hashCode()` (сравнение по ссылке).
- Использовать `id` только после сохранения сущности, но с осторожностью.

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User)) return false;
    User user = (User) o;
    return Objects.equals(name, user.name);
}

@Override
public int hashCode() {
    return Objects.hash(name);
}
```

В этом примере `name` используется как натуральный ключ для сравнения.

---

# PersistentCollection

Hibernate использует специальный механизм для управления коллекциями, связанными с базой данных.

## CollectionType

`CollectionType` — внутренний тип Hibernate, который управляет маппингом коллекций (`List`, `Set`, `Map`) на базу данных. Он определяет, как коллекция синхронизируется с базой, включая добавление, удаление и обновление элементов.

## Базовый класс PersistentCollection

`PersistentCollection` — абстрактный класс Hibernate, представляющий коллекции, связанные с базой данных. Он отслеживает изменения в коллекции (например, добавление или удаление элементов) и синхронизирует их с базой при фиксации транзакции.

## Утилитный класс Hibernate

Утилитный класс `org.hibernate.Hibernate` предоставляет методы для работы с коллекциями:

- `Hibernate.isInitialized(Object)`: Проверяет, загружена ли коллекция.
- `Hibernate.initialize(Object)`: Принудительно загружает коллекцию.

```java
Company company = em.find(Company.class, 1L);
if (!Hibernate.isInitialized(company.getUsers())) {
    Hibernate.initialize(company.getUsers()); // Принудительная загрузка
}
```

---

# LazyInitializationException

`LazyInitializationException` — одна из самых распространённых ошибок при работе с ленивой загрузкой в Hibernate.

## Воспроизведение ошибки LazyInitializationException

Ошибка возникает, когда пытаются получить доступ к лениво загружаемой коллекции или прокси после закрытия сессии Hibernate.

```java
User user = em.find(User.class, 1L);
em.close();
user.getCompany().getName(); // Ошибка: LazyInitializationException
```

## Почему LazyInitializationException возникает?

Hibernate создаёт прокси-объекты или лениво загружаемые коллекции, которые требуют активной сессии для загрузки данных из базы. Если сессия закрыта, Hibernate не может выполнить запрос, что приводит к исключению.

## Как поступают в реальных приложениях?

Для предотвращения `LazyInitializationException` используют следующие подходы:

- **Использование `FetchType.EAGER`**: Загружает данные сразу, но увеличивает нагрузку на базу.
- **Принудительная инициализация**: Используйте `Hibernate.initialize()` в рамках активной сессии.
- **Open Session in View**: Держит сессию открытой во время обработки запроса (не рекомендуется для высоконагруженных систем из-за риска утечек ресурсов).
- **JPQL с JOIN FETCH**: Загружает связанные данные в одном запросе.

```java
// JPQL с JOIN FETCH
Query query = em.createQuery("SELECT u FROM User u JOIN FETCH u.company WHERE u.id = :id");
query.setParameter("id", 1L);
User user = (User) query.getSingleResult();
```

## Метод Session.getReference

Метод `EntityManager.getReference()` (или `Session.getReference()` в Hibernate) возвращает прокси-объект без выполнения запроса к базе данных.

```java
Company company = em.getReference(Company.class, 1L);
System.out.println(Hibernate.isInitialized(company)); // false
```

Этот метод полезен, когда нужно передать ссылку на сущность без немедленной загрузки данных.

---

# OrphanRemoval

Аннотация `orphanRemoval = true` в `@OneToMany` или `@OneToOne` автоматически удаляет элементы коллекции или связанные сущности, которые были удалены из связи в родительской сущности.

## Суть работы OrphanRemoval

Когда элемент удаляется из коллекции (например, `company.getUsers().remove(user)`), Hibernate удаляет соответствующую запись из базы данных. Это отличается от `CascadeType.REMOVE`, который удаляет связанные сущности только при удалении родительской.

## Тестирование OrphanRemoval

```java
@Entity
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @OneToMany(mappedBy = "company", orphanRemoval = true)
    private List<User> users = new ArrayList<>();

    // Геттеры и сеттеры
}

em.getTransaction().begin();
Company company = em.find(Company.class, 1L);
company.getUsers().remove(0); // Удаляет User из базы
em.getTransaction().commit();
```

В этом примере удаление пользователя из коллекции `users` приводит к удалению записи из таблицы `users`.

---

# OneToOne. Primary Key

Связь "один к одному" с первичным ключом используется, когда первичный ключ одной таблицы одновременно является внешним ключом для другой.

## Создание таблицы profile

```sql
CREATE TABLE profiles (
    user_id BIGINT PRIMARY KEY,
    bio TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

## Создание Entity Profile

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.OneToOne;
import javax.persistence.PrimaryKeyJoinColumn;

@Entity
public class Profile {
    @Id
    private Long userId;
    private String bio;
    @OneToOne
    @PrimaryKeyJoinColumn
    private User user;

    // Геттеры и сеттеры
    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public String getBio() { return bio; }
    public void setBio(String bio) { this.bio = bio; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}
```

## Аннотация @OneToOne

Аннотация `@OneToOne` определяет связь "один к одному" между двумя сущностями.

## Аннотация @PrimaryKeyJoinColumn

`@PrimaryKeyJoinColumn` указывает, что первичный ключ одной таблицы (`profiles.user_id`) используется как внешний ключ для связи с другой таблицей (`users.id`).

## Bidirectional @OneToOne

Двунаправленная связь включает обратную ссылку в сущности `User`.

```java
@Entity
public class User {
    @Id
    private Long id;
    private String name;
    @OneToOne(mappedBy = "user")
    private Profile profile;

    // Геттеры и сеттеры
}
```

## Тестирование @OneToOne

```java
em.getTransaction().begin();
User user = new User();
user.setId(1L);
user.setName("John");
Profile profile = new Profile();
profile.setUserId(1L);
profile.setBio("Bio");
profile.setUser(user);
user.setProfile(profile);
em.persist(user);
em.persist(profile);
em.getTransaction().commit();

Profile found = em.find(Profile.class, 1L);
assertEquals("John", found.getUser().getName());
```

---

# OneToOne. Foreign Key

Связь "один к одному" с внешним ключом использует отдельный столбец для хранения ссылки на связанную сущность.

## OneToOneType

`OneToOneType` — внутренний класс Hibernate, управляющий маппингом связи `@OneToOne`. Он отвечает за обработку внешнего ключа и выборки данных.

## Синтетический ключ в таблице profile

```sql
CREATE TABLE profiles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE,
    bio TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

## Тестирование OneToOne с синтетическим id

```java
@Entity
public class Profile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String bio;
    @OneToOne
    @JoinColumn(name = "user_id")
    private User user;

    // Геттеры и сеттеры
}
```

## Lazy работает в OneToOne только с натуральным id

Ленивая загрузка (`FetchType.LAZY`) в `@OneToOne` поддерживается, если первичный ключ совпадает с внешним ключом (как в случае с `@PrimaryKeyJoinColumn`). При использовании синтетического ключа (`id`) Hibernate может игнорировать `FetchType.LAZY`, загружая данные немедленно.

---

# ManyToMany

Связь "многие ко многим" (`@ManyToMany`) используется, когда множество сущностей одной таблицы связано с множеством сущностей другой таблицы через промежуточную таблицу.

## Создание таблиц chat и users_chat

```sql
CREATE TABLE chats (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE users_chats (
    user_id BIGINT,
    chat_id BIGINT,
    PRIMARY KEY (user_id, chat_id),
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (chat_id) REFERENCES chats(id)
);
```

## Создание сущности Chat

```java
@Entity
public class Chat {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    @ManyToMany
    @JoinTable(
        name = "users_chats",
        joinColumns = @JoinColumn(name = "chat_id"),
        inverseJoinColumns = @JoinColumn(name = "user_id")
    )
    private List<User> users = new ArrayList<>();

    // Геттеры и сеттеры
}
```

## Аннотация @ManyToMany

`@ManyToMany` определяет связь "многие ко многим" через промежуточную таблицу.

## Аннотация @JoinTable

`@JoinTable` указывает имя промежуточной таблицы (`users_chats`) и столбцы для связи (`chat_id` и `user_id`).

## Тестирование @ManyToMany

```java
em.getTransaction().begin();
Chat chat = new Chat();
chat.setName("GroupChat");
User user = em.find(User.class, 1L);
chat.getUsers().add(user);
em.persist(chat);
em.getTransaction().commit();

Chat found = em.find(Chat.class, chat.getId());
assertEquals(1, found.getUsers().size());
```

---

# ManyToMany. Separate Entity

Иногда вместо `@ManyToMany` удобнее использовать отдельную сущность для промежуточной таблицы, чтобы добавить дополнительные атрибуты.

## Синтетический ключ в таблице users_chats

```sql
CREATE TABLE users_chats (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT,
    chat_id BIGINT,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (chat_id) REFERENCES chats(id)
);
```

## Создание сущности UsersChat

```java
@Entity
public class UsersChat {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
    @ManyToOne
    @JoinColumn(name = "chat_id")
    private Chat chat;

    // Геттеры и сеттеры
}
```

## Меняем @ManyToMany на @OneToMany

```java
@Entity
public class Chat {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @OneToMany(mappedBy = "chat")
    private List<UsersChat> usersChats = new ArrayList<>();

    // Геттеры и сеттеры
}
```

---

# Collection Performance

Производительность работы с коллекциями в Hibernate критически важна для больших наборов данных.

## Чем плохо получение коллекции через методы get*

Методы, такие как `getUsers()`, с `FetchType.EAGER` загружают всю коллекцию сразу, что может привести к выполнению больших SQL-запросов и снижению производительности. Например, загрузка компании с сотнями пользователей может вызвать декартово произведение при использовании `JOIN`.

## Почему желательно использовать List, а не Set

- `List` сохраняет порядок элементов и поддерживает индексацию, что упрощает управление данными.
- `Set` не гарантирует порядок и может быть менее эффективным при частых операциях добавления/удаления из-за проверки уникальности.

---

# ElementCollection

`@ElementCollection` используется для маппинга коллекций простых типов или встроенных объектов.

## Создание таблицы company_locale

```sql
CREATE TABLE company_locales (
    company_id BIGINT,
    locale VARCHAR(10),
    name VARCHAR(255),
    FOREIGN KEY (company_id) REFERENCES companies(id)
);
```

## Embeddable component LocaleInfo

```java
@Embeddable
public class LocaleInfo {
    private String locale;
    private String name;

    // Геттеры и сеттеры
    public String getLocale() { return locale; }
    public void setLocale(String locale) { this.locale = locale; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

## Аннотация @ElementCollection

`@ElementCollection` указывает, что поле является коллекцией простых типов или встроенных объектов.

## Аннотация @CollectionTable

`@CollectionTable` определяет таблицу для хранения элементов коллекции.

## Переопределение свойств через @AttributeOverride

```java
@Entity
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @ElementCollection
    @CollectionTable(name = "company_locales", joinColumns = @JoinColumn(name = "company_id"))
    @AttributeOverride(name = "name", column = @Column(name = "localized_name"))
    private List<LocaleInfo> locales = new ArrayList<>();

    // Геттеры и сеттеры
}
```

## @ElementCollection только на чтение

Параметры `updatable = false` и `insertable = false` делают коллекцию неизменяемой, защищая её от модификаций.

```java
@ElementCollection
@CollectionTable(name = "company_locales")
@AttributeOverride(name = "name", column = @Column(name = "localized_name", updatable = false, insertable = false))
private List<LocaleInfo> locales;
```

---

# Collection Ordering

Hibernate предоставляет несколько аннотаций для управления порядком элементов в коллекциях.

## Аннотация @OrderBy

`@OrderBy` сортирует коллекцию по указанному полю с помощью SQL-запроса.

```java
@OneToMany(mappedBy = "company")
@OrderBy("name ASC")
private List<User> users;
```

## Тестирование @OrderBy

```java
Company company = em.find(Company.class, 1L);
List<User> users = company.getUsers(); // Отсортированы по имени
```

## Сортировка в памяти (на уровне Java приложения)

Для сортировки в памяти можно использовать `Collections.sort()` или Stream API:

```java
company.getUsers().sort(Comparator.comparing(User::getName));
```

## Аннотация @OrderColumn

`@OrderColumn` сохраняет порядок элементов в отдельном столбце базы данных.

```java
@OneToMany(mappedBy = "company")
@OrderColumn(name = "order_index")
private List<User> users;
```

## Аннотация @SortNatural

`@SortNatural` сортирует коллекцию по естественному порядку, если элементы реализуют интерфейс `Comparable`.

## Аннотация @SortComparator

`@SortComparator` использует кастомный компаратор для сортировки.

```java
@OneToMany(mappedBy = "company")
@SortComparator(UserNameComparator.class)
private SortedSet<User> users;
```

---

# Maps in Mappings

Hibernate поддерживает маппинг коллекций типа `Map` для хранения пар ключ-значение.

## Аннотация @MapKey

`@MapKey` указывает поле сущности, используемое как ключ в `Map`.

```java
@Entity
public class Company {
    @OneToMany(mappedBy = "company")
    @MapKey(name = "id")
    private Map<Long, User> usersById = new HashMap<>();

    // Геттеры и сеттеры
}
```

## Сортировка только по ключу

`@OrderBy` можно использовать для сортировки ключей `Map`.

```java
@OneToMany(mappedBy = "company")
@MapKey(name = "id")
@OrderBy("id ASC")
private Map<Long, User> usersById;
```

## Аннотации @MapKeyClass, @MapKeyEnumerated, @MapKeyTemporal

- `@MapKeyClass`: Указывает тип ключа.
- `@MapKeyEnumerated`: Используется для ключей-перечислений.
- `@MapKeyTemporal`: Применяется для ключей-временных типов (`Date`, `Calendar`).

## Аннотация @MapsId

`@MapsId` использует идентификатор связанной сущности как ключ.

```java
@Entity
public class User {
    @Id
    private Long id;
    @OneToOne
    @MapsId
    private Profile profile;

    // Геттеры и сеттеры
}
```

## Аннотации @MapKeyJoinColumn, @MapKeyColumn

- `@MapKeyJoinColumn`: Настраивает столбец для ключа, если ключ — это связанная сущность.
- `@MapKeyColumn`: Указывает столбец для простого ключа.

```java
@OneToMany(mappedBy = "company")
@MapKeyColumn(name = "user_key")
private Map<String, User> usersByKey;
```

---

# Итог

- **ManyToOne**: Связь "многие к одному" с поддержкой ленивой загрузки через прокси-объекты и настройкой через `@JoinColumn`.
- **Fetch Types**: Ленивая (`LAZY`) и немедленная (`EAGER`) загрузка, управление прокси с помощью `Hibernate.initialize`.
- **Hibernate Proxy**: Использование ByteBuddy для создания прокси-объектов, перехват вызовов через `ByteBuddyInterceptor`.
- **Cascade Types**: Автоматизация операций (`PERSIST`, `REMOVE`, `DETACH`, `ALL`) для связанных сущностей.
- **OneToMany**: Однонаправленные и двунаправленные связи, предотвращение `StackOverflowError` через исключение циклических вызовов.
- **Entity equals and hashCode**: Использование натуральных ключей для стабильного сравнения, избегание `id` для новых сущностей.
- **PersistentCollection**: Управление коллекциями через `CollectionType`, синхронизация с базой с помощью `PersistentCollection`.
- **LazyInitializationException**: Обработка ошибок ленивой загрузки через `initialize`, `getReference` или JPQL.
- **OrphanRemoval**: Удаление "осиротевших" элементов коллекций при использовании `orphanRemoval = true`.
- **OneToOne**: Связь с первичным или внешним ключом, особенности ленивой загрузки с натуральным id.
- **ManyToMany**: Связь через промежуточную таблицу или отдельную сущность для большей гибкости.
- **Collection Performance**: Оптимизация загрузки коллекций через `LAZY` и избегание декартова произведения.
- **ElementCollection**: Маппинг коллекций встроенных объектов с настройкой через `@CollectionTable` и `@AttributeOverride`.
- **Collection Ordering**: Сортировка коллекций с помощью `@OrderBy`, `@OrderColumn`, `@SortNatural`, `@SortComparator`.
- **Maps in Mappings**: Использование `Map` с ключами через `@MapKey`, `@MapsId`, `@MapKeyJoinColumn` и другими аннотациями.

Эти механизмы обеспечивают гибкое и мощное моделирование отношений между сущностями в Hibernate, позволяя эффективно управлять сложными структурами данных.