# Пессимистические блокировки в JPA/Hibernate

## Оглавление
1. [Что такое пессимистическая блокировка](#что-это)
2. [LockModeType: виды пессимистических блокировок](#lockmodetype)
3. [Пессимистические блокировки в PostgreSQL](#postgres)
4. [Пессимистические блокировки в HQL и Criteria API](#hql)
5. [Query timeouts и deadlocks](#timeouts)
6. [Best practices](#best-practices)
7. [Вопросы для собеседования](#вопросы)

---

## 1. Что такое пессимистическая блокировка <a name="что-это"></a>

**Пессимистическая блокировка** — это механизм, при котором запись блокируется для других транзакций до завершения текущей. Используется для предотвращения конфликтов при одновременной работе с одними и теми же данными.

- Блокировка на уровне базы данных (SELECT ... FOR UPDATE)
- Гарантирует, что никто не изменит/прочитает данные до завершения транзакции

## 2. LockModeType: виды пессимистических блокировок <a name="lockmodetype"></a>

- **PESSIMISTIC_READ** — блокировка для чтения (другие могут читать, но не изменять)
- **PESSIMISTIC_WRITE** — блокировка для записи (другие не могут читать и изменять)
- **PESSIMISTIC_FORCE_INCREMENT** — блокировка с увеличением версии (для @Version)

### Пример (JPA):
```java
entityManager.find(User.class, id, LockModeType.PESSIMISTIC_WRITE);
```

### Пример (Spring Data JPA):
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
User findById(Long id);
```

## 3. Пессимистические блокировки в PostgreSQL <a name="postgres"></a>

- **SELECT ... FOR UPDATE** — блокирует строки для других транзакций
- **SELECT ... FOR SHARE** — блокировка для чтения

### Пример:
```sql
SELECT * FROM users WHERE id = 1 FOR UPDATE;
```

- В PostgreSQL пессимистические блокировки реализуются через MVCC и блокировки строк
- Возможны deadlocks при неправильном порядке блокировок

## 4. Пессимистические блокировки в HQL и Criteria API <a name="hql"></a>

### Пример (HQL):
```java
List<User> users = session.createQuery("from User where id = :id")
    .setParameter("id", 1L)
    .setLockMode(LockMode.PESSIMISTIC_WRITE)
    .list();
```

### Пример (Criteria API):
```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.select(root).where(cb.equal(root.get("id"), 1L));
TypedQuery<User> query = em.createQuery(cq);
query.setLockMode(LockModeType.PESSIMISTIC_WRITE);
User user = query.getSingleResult();
```

## 5. Query timeouts и deadlocks <a name="timeouts"></a>

- **Query timeout** — максимальное время ожидания блокировки
- В JPA/Hibernate можно задать через hint:
```java
query.setHint("javax.persistence.lock.timeout", 5000); // 5 секунд
```
- В PostgreSQL можно задать через:
```sql
SET lock_timeout = '5s';
```
- Deadlock — взаимная блокировка транзакций, приводит к откату одной из них

## 6. Best practices <a name="best-practices"></a>
- Используйте пессимистические блокировки только при реальной необходимости
- Минимизируйте время удержания блокировки
- Всегда задавайте timeout для блокировок
- Следите за порядком блокировок для предотвращения deadlock
- Логируйте случаи deadlock и анализируйте их причины

## 7. Вопросы для собеседования <a name="вопросы"></a>
1. Чем отличается PESSIMISTIC_READ от PESSIMISTIC_WRITE?
2. Как реализуются пессимистические блокировки в PostgreSQL?
3. Как задать timeout для пессимистической блокировки в JPA?
4. Как избежать deadlock при использовании пессимистических блокировок?
5. Когда стоит использовать пессимистические блокировки, а когда — оптимистические?

---

**Дополнительные ресурсы:**
- [JPA Locking](https://docs.oracle.com/javaee/7/tutorial/persistence-locking001.htm)
- [Hibernate Locking](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#locking)
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
