# equals и hashCode для Entity

## Оглавление
1. [Почему это важно](#почему-важно)
2. [Проблема с id в equals() до persist](#проблема-id)
3. [Три стратегии реализации](#стратегии)
4. [Проблема с Set и detached entities](#проблема-set)
5. [Проблема с Hibernate proxy](#проблема-proxy)
6. [Правило использования](#правило)

---

## 1. Почему это важно <a name="почему-важно"></a>

По умолчанию Java использует `Object.equals()` (сравнение по ссылке) и `Object.hashCode()` (идентификатор объекта). Для entity это создаёт серьёзные проблемы при работе с `Set`, `Map`, и при merge/detach/reattach.

> [!WARNING]
> Если entity помещается в `Set` или используется как ключ `Map` — **обязательно** переопределяй `equals()` и `hashCode()`. Без этого поведение непредсказуемо.

---

## 2. Проблема с id в equals() до persist <a name="проблема-id"></a>

Самая распространённая ошибка — использование `id` в `equals()`/`hashCode()`:

```java
// ПЛОХО: id до persist == null
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User)) return false;
    User user = (User) o;
    return Objects.equals(id, user.id); // id == null до persist!
}

@Override
public int hashCode() {
    return Objects.hash(id); // hashCode == 0 до persist!
}
```

**Что пойдёт не так:**

```java
Set<User> users = new HashSet<>();

User user = new User("alice@example.com");
users.add(user);          // hashCode() → 0 (id == null), bucket #0

entityManager.persist(user); // теперь id = 42

users.contains(user);     // ЛОЖЬ! hashCode() теперь 42, ищет в bucket #42
// Объект "потерялся" в Set!
```

> [!INFO]
> Проблема в том, что `hashCode()` изменился после `persist()`. Объект был добавлен в Set по одному хэшу, а после получения id хранится в "неправильной" корзине.

---

## 3. Три стратегии реализации <a name="стратегии"></a>

### Стратегия 1: Бизнес-ключ (рекомендуется)

Используй поле с бизнес-уникальностью (email, username, код продукта):

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String email; // бизнес-ключ

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        // instanceof вместо getClass() — работает корректно с Hibernate proxy
        if (!(o instanceof User other)) return false;
        return email != null && email.equals(other.getEmail());
    }

    @Override
    public int hashCode() {
        // Константа: hashCode не меняется — объект всегда найдётся в Set
        return getClass().hashCode();
    }
}
```

> [!INFO]
> `hashCode()` возвращает константу для всех экземпляров класса. Это ухудшает производительность HashMap (все в одной корзине), но **гарантирует корректность**. Для entity в Set это оправданный компромисс.

### Стратегия 2: UUID как суррогатный ключ

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    // UUID генерируется ДО persist — можно использовать в equals/hashCode

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Product other)) return false;
        return id != null && id.equals(other.getId());
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

> [!INFO]
> UUID присваивается в конструкторе или через `@GeneratedValue(strategy = GenerationType.UUID)` в Hibernate 6 — ещё до вставки в БД. Поэтому он стабилен и подходит для equals/hashCode.

### Стратегия 3: Суррогатный Long ID (с осторожностью)

```java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order other)) return false;
        // Работает только если id != null (т.е. после persist)
        return id != null && id.equals(other.getId());
    }

    @Override
    public int hashCode() {
        return getClass().hashCode(); // константа!
    }
}
```

> [!WARNING]
> Если добавить entity в Set **до** `persist()`, а потом сделать `persist()` — `equals()` начнёт работать корректно, но объект уже лежит в Set. Будь осторожен с этой стратегией.

---

## 4. Проблема с Set при работе с detached/merged entities <a name="проблема-set"></a>

```java
@Entity
public class Department {
    @OneToMany(mappedBy = "department")
    private Set<Employee> employees = new HashSet<>();
}
```

**Сценарий проблемы:**

```java
Employee emp = entityManager.find(Employee.class, 1L);
department.getEmployees().add(emp); // добавили в Set

entityManager.detach(emp);          // emp стал detached

Employee merged = entityManager.merge(emp); // новый объект!

// Если equals по умолчанию (по ссылке):
department.getEmployees().contains(emp);    // true  (по ссылке — есть)
department.getEmployees().contains(merged); // false (другая ссылка)
// Несогласованность!

// С правильным бизнес-ключом:
department.getEmployees().contains(merged); // true (email совпадает)
```

---

## 5. Проблема с Hibernate proxy <a name="проблема-proxy"></a>

При lazy loading Hibernate создаёт **proxy-подкласс** твоей entity:

```java
User user = entityManager.getReference(User.class, 1L);
// user.getClass() == UserHibernateProxy$... (не User.class!)
```

> [!WARNING]
> Никогда не используй `getClass()` в `equals()` для entity! Это сломает сравнение managed entity с её proxy.

```java
// ПЛОХО:
@Override
public boolean equals(Object o) {
    if (getClass() != o.getClass()) return false; // сломается с proxy!
    ...
}

// ХОРОШО: instanceof проверяет иерархию — proxy наследует User
@Override
public boolean equals(Object o) {
    if (!(o instanceof User other)) return false; // работает и с proxy
    ...
}
```

**Дополнительно**: для получения реального класса proxy используй:

```java
// Hibernate API
Class<?> realClass = Hibernate.getClass(entity);

// Или через HibernateProxyHelper
Class<?> realClass = HibernateProxyHelper.getClassWithoutInitializingProxy(entity);
```

```java
// Почему entity.getClass() может ввести в заблуждение:
User proxy = entityManager.getReference(User.class, 1L);
proxy.getClass();                // → User$HibernateProxy$xyz
Hibernate.getClass(proxy);       // → User
Hibernate.isInitialized(proxy);  // → false (данные ещё не загружены)
```

---

## 6. Правило использования <a name="правило"></a>

**Правило**: если entity когда-либо будет в `Set` или `Map` — `equals()`/`hashCode()` обязательны.

### Шаблон для большинства случаев

```java
@Entity
public class MyEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String naturalKey; // бизнес-идентификатор

    @Override
    public final boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof MyEntity other)) return false;
        // null-safe сравнение бизнес-ключа
        return naturalKey != null && naturalKey.equals(other.naturalKey);
    }

    @Override
    public final int hashCode() {
        // Константный hashCode: стабилен до и после persist
        return getClass().hashCode();
    }
}
```

> [!INFO]
> `final` на `equals()`/`hashCode()` рекомендован Hibernate — чтобы proxy-подкласс не мог переопределить эти методы и сломать логику.

### Быстрый выбор стратегии

```
Есть уникальный бизнес-атрибут (email, SKU, username)?
  → Используй бизнес-ключ (Стратегия 1)

Нет бизнес-ключа, но нужна стабильность до persist?
  → Используй UUID (Стратегия 2)

Простая entity, в Set только после persist?
  → Surrogate Long с guard (id != null) (Стратегия 3)
```

---

**Связанные файлы:**
- [[Основы]] — жизненный цикл entity
- [[Управление_загрузкой]] — Hibernate proxy и getClass()
- [[ManyToMany_связи]] — коллекции и Set-проблемы
