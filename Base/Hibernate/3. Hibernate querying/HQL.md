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

**Дополнительные ресурсы:**
- [Hibernate HQL Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#hql)
- [JPA JPQL Guide](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#jpql)