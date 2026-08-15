# HQL и JPQL: подробное руководство

## Оглавление
1. [Что такое HQL и JPQL](#intro)
2. [Объект Query и его использование](#query)
3. [Параметризация: индексная и именованная](#params)
4. [Явные и неявные Joins](#joins)
5. [Именованные HQL-запросы](#named)
6. [Query Hints](#hints)
7. [HQL-запросы на update/delete/insert](#dml)
8. [Native query](#native)
9. [Практические задачи](#tasks)
10. [Best practices](#best-practices)
11. [Вопросы для собеседования](#вопросы)

---

## 1. Что такое HQL и JPQL <a name="intro"></a>

- **HQL (Hibernate Query Language)** — объектно-ориентированный язык запросов, похожий на SQL, но оперирует сущностями и их полями, а не таблицами и колонками.
- **JPQL (Java Persistence Query Language)** — стандартный язык запросов JPA, синтаксис почти идентичен HQL.
- Позволяет писать переносимые запросы, не зависящие от конкретной СУБД.

## 2. Объект Query и его использование <a name="query"></a>

### Пример:
```java
Query query = entityManager.createQuery("SELECT u FROM User u WHERE u.age > :age");
query.setParameter("age", 18);
List<User> users = query.getResultList();
```
- В Hibernate: session.createQuery(...)
- В JPA: entityManager.createQuery(...)

## 3. Параметризация: индексная и именованная <a name="params"></a>

- **Индексная**:
```java
Query q = em.createQuery("SELECT u FROM User u WHERE u.age > ?1");
q.setParameter(1, 18);
```
- **Именованная**:
```java
Query q = em.createQuery("SELECT u FROM User u WHERE u.age > :age");
q.setParameter("age", 18);
```

## 4. Явные и неявные Joins <a name="joins"></a>

- **Явный join**:
```java
SELECT u FROM User u JOIN u.company c WHERE c.name = :name
```
- **Неявный join** (property navigation):
```java
SELECT u.company.name FROM User u
```
- **LEFT JOIN, FETCH JOIN**:
```java
SELECT u FROM User u LEFT JOIN FETCH u.payments
```

## 5. Именованные HQL-запросы <a name="named"></a>

- Описываются через аннотацию @NamedQuery:
```java
@NamedQuery(name = "User.findAll", query = "SELECT u FROM User u")
```
- Использование:
```java
Query q = em.createNamedQuery("User.findAll");
```

## 6. Query Hints <a name="hints"></a>

- Позволяют управлять поведением запроса (кэширование, таймауты и др.)
```java
query.setHint("org.hibernate.cacheable", true);
query.setHint("javax.persistence.query.timeout", 5000);
```

## 7. HQL-запросы на update/delete/insert <a name="dml"></a>

- **UPDATE**:
```java
Query q = em.createQuery("UPDATE User u SET u.active = false WHERE u.lastLogin < :date");
q.setParameter("date", ...);
q.executeUpdate();
```
- **DELETE**:
```java
Query q = em.createQuery("DELETE FROM User u WHERE u.active = false");
q.executeUpdate();
```
- **INSERT** (только в Hibernate):
```java
Query q = session.createQuery("INSERT INTO UserArchive (id, name) SELECT u.id, u.name FROM User u WHERE ...");
q.executeUpdate();
```

## 8. Native query <a name="native"></a>

- Позволяет выполнять SQL-запросы напрямую:
```java
Query q = em.createNativeQuery("SELECT * FROM users WHERE age > ?", User.class);
q.setParameter(1, 18);
List<User> users = q.getResultList();
```

## 9. Практические задачи <a name="tasks"></a>

1. **findAll**
```java
SELECT u FROM User u
```
2. **findAllByFirstName**
```java
SELECT u FROM User u WHERE u.firstName = :firstName
```
3. **findLimitedUsersOrderedByBirthday**
```java
SELECT u FROM User u ORDER BY u.birthday
// setMaxResults(N)
```
4. **findAllByCompanyName**
```java
SELECT u FROM User u WHERE u.company.name = :companyName
```
5. **findAllPaymentsByCompanyName**
```java
SELECT p FROM Payment p WHERE p.user.company.name = :companyName
```
6. **findAveragePaymentAmountByFirstAndLastNames**
```java
SELECT AVG(p.amount) FROM Payment p WHERE p.user.firstName = :firstName AND p.user.lastName = :lastName
```
7. **findCompanyNamesWithAvgUserPaymentsOrderedByCompanyName**
```java
SELECT c.name, AVG(p.amount) FROM Company c JOIN c.users u JOIN u.payments p GROUP BY c.name ORDER BY c.name
```
8. **isItPossible**
- Проверка возможности выполнения сложных запросов (например, подзапросы, exists, in, not in)

## 10. Best practices <a name="best-practices"></a>
- Используйте именованные параметры для читаемости
- Не используйте HQL для сложных SQL-специфичных операций — используйте native query
- Для массовых изменений используйте batch-операции
- Следите за N+1 проблемой (используйте fetch join)
- Используйте кэширование для часто используемых запросов

## 11. Вопросы для собеседования <a name="вопросы"></a>
1. Чем отличается HQL от SQL?
2. Как работают именованные параметры?
3. Как реализовать join в HQL?
4. Как выполнить массовое обновление через HQL?
5. Как использовать native query?
6. Как избежать N+1 проблемы в HQL?

---

## 12. Senior-insights: Hibernate 6+ и подводные камни <a name="senior"></a>

### Новый API Hibernate 6: createSelectionQuery() и createMutationQuery()

В Hibernate 6 появился типобезопасный API вместо устаревшего `createQuery()`:

```java
// Hibernate 6+: типобезопасные варианты
Session session = entityManager.unwrap(Session.class);

// Для SELECT-запросов
List<User> users = session.createSelectionQuery(
    "FROM User u WHERE u.active = true", User.class)
    .getResultList();

// Для UPDATE/DELETE
int updated = session.createMutationQuery(
    "UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoff")
    .setParameter("cutoff", LocalDate.now().minusYears(1))
    .executeUpdate();

// Устаревший способ (всё ещё работает, но без type-safety):
// session.createQuery("FROM User", User.class)
```

> [!INFO]
> `createSelectionQuery()` выбросит исключение если попытаться передать DML-запрос. Это помогает поймать ошибки на этапе разработки.

### Window Functions в Hibernate 6+ (ROW_NUMBER, RANK)

Hibernate 6 добавил поддержку оконных функций SQL:

```java
// ROW_NUMBER для нумерации строк внутри группы
List<Object[]> results = entityManager.createQuery(
    """
    SELECT u.name,
           u.department,
           ROW_NUMBER() OVER (PARTITION BY u.department ORDER BY u.salary DESC) AS rank
    FROM User u
    """)
    .getResultList();

// RANK для ранжирования (с пропусками при одинаковых значениях)
List<Object[]> ranked = entityManager.createQuery(
    """
    SELECT p.productName,
           p.category,
           RANK() OVER (PARTITION BY p.category ORDER BY p.sales DESC) AS salesRank
    FROM Product p
    """)
    .getResultList();
```

### CTE (WITH clause) в HQL

```java
// Common Table Expression в Hibernate 6
List<User> topUsers = entityManager.createQuery(
    """
    WITH top_users AS (
        SELECT u FROM User u WHERE u.totalOrders > 100
    )
    SELECT u FROM top_users u ORDER BY u.totalOrders DESC
    """, User.class)
    .setMaxResults(10)
    .getResultList();
```

### FILTER предложение для агрегатных функций

```java
// FILTER позволяет считать разные агрегаты с разными условиями за один запрос
List<Object[]> stats = entityManager.createQuery(
    """
    SELECT
        u.department,
        COUNT(u),
        COUNT(u) FILTER (WHERE u.active = true) AS activeCount,
        AVG(u.salary) FILTER (WHERE u.role = 'SENIOR') AS avgSeniorSalary
    FROM User u
    GROUP BY u.department
    """)
    .getResultList();
```

### Bulk UPDATE/DELETE и их влияние на L2 Cache

> [!WARNING]
> **Bulk UPDATE/DELETE через HQL не инвалидируют L2 Cache!** Это частая ловушка при использовании Second Level Cache.

```java
// После этого bulk update — L2 Cache содержит устаревшие данные!
entityManager.createQuery(
    "UPDATE Product p SET p.price = p.price * 1.1 WHERE p.category = :cat")
    .setParameter("cat", "electronics")
    .executeUpdate();

// Product из L2 Cache будут возвращать СТАРЫЕ цены до истечения TTL
Product product = entityManager.find(Product.class, someId); // HIT в L2 Cache!

// ПРАВИЛЬНО: инвалидировать вручную
entityManager.createQuery("UPDATE Product p SET p.price = p.price * 1.1 WHERE ...")
    .executeUpdate();
sessionFactory.getCache().evictEntityData(Product.class); // инвалидировать кэш
```

> [!INFO]
> Аналогичная ситуация с `DELETE FROM Entity`: bulk delete не вызывает `@PreRemove` callbacks и не инвалидирует L2 Cache.

---

**Дополнительные ресурсы:**
- [Hibernate HQL Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#hql)
- [JPA JPQL Guide](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#jpql)

**Связанные файлы:**
- [[Criteria API]] — типобезопасный способ построения запросов
- [[Batch операции]] — bulk операции и их ограничения
- [[Cache Concurrency Strategy]] — инвалидация L2 Cache