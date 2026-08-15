# Dirty Checking и FlushMode

## Оглавление
1. [Что такое Dirty Checking](#dirty-checking)
2. [Snapshot-механика: как Hibernate отслеживает изменения](#snapshot)
3. [ActionQueue — очередь SQL-операций](#action-queue)
4. [FlushMode: AUTO, COMMIT, MANUAL, ALWAYS](#flushmode)
5. [Подводные камни](#ловушки)

---

## 1. Что такое Dirty Checking <a name="dirty-checking"></a>

**Dirty Checking** — механизм Hibernate, который автоматически обнаруживает изменения в managed-сущностях и генерирует соответствующие SQL UPDATE без явного вызова `session.update()`.

При flush Hibernate сравнивает текущее состояние каждой managed-entity со snapshot, снятым в момент загрузки. Если есть расхождение — генерируется UPDATE.

```java
@Transactional
public void updateUserEmail(Long id, String newEmail) {
    User user = entityManager.find(User.class, id); // загрузка + snapshot
    user.setEmail(newEmail);                         // изменение поля
    // entityManager.merge() — НЕ нужен! flush сделает UPDATE автоматически
}
// При выходе из метода: Spring вызывает flush → commit
```

> [!INFO]
> Dirty Checking работает только с **managed entities** (в рамках открытого Persistence Context). Detached-объекты не отслеживаются.

---

## 2. Snapshot-механика: как Hibernate отслеживает изменения <a name="snapshot"></a>

### Как создаётся snapshot

При загрузке entity через `find()` или запрос Hibernate:
1. Выполняет SQL SELECT
2. Создаёт объект entity
3. **Сохраняет копию состояния полей** в `EntityEntry` в рамках `StatefulPersistenceContext`

Snapshot хранит примитивные значения и ссылки — **не глубокие копии объектов**.

### Как работает сравнение при flush

```
1. Hibernate обходит все managed entities в Persistence Context
2. Для каждой entity сравнивает текущие значения полей со snapshot
3. Если нашёл отличие → добавляет EntityUpdateAction в ActionQueue
4. Выполняет все операции из ActionQueue
```

> [!INFO]
> Сравнение полей происходит через метод `equals()` типа. Для `byte[]` это `Arrays.equals()`. Для пользовательских типов важно правильно реализовать `equals()`.

### Стоимость Dirty Checking

Проход по всем полям **каждой** managed entity выполняется при каждом flush. При больших Persistence Context (тысячи объектов) это создаёт заметный overhead.

```java
// Плохо: держим в PC тысячи объектов
for (int i = 0; i < 10000; i++) {
    User user = entityManager.find(User.class, (long) i);
    user.setActive(false);
    // После 10000 итераций PC содержит 10000 объектов + 10000 snapshots
}

// Хорошо: bulk UPDATE через HQL
entityManager.createQuery("UPDATE User u SET u.active = false")
    .executeUpdate();
```

---

## 3. ActionQueue — очередь SQL-операций <a name="action-queue"></a>

`ActionQueue` — внутренняя очередь Hibernate, в которую складываются все DML-операции до момента flush.

### Типы операций в очереди

| Action | SQL | Когда добавляется |
|--------|-----|-------------------|
| `EntityInsertAction` | INSERT | `session.persist()` |
| `EntityUpdateAction` | UPDATE | Dirty Checking при flush |
| `EntityDeleteAction` | DELETE | `session.remove()` |
| `CollectionRecreateAction` | DELETE + INSERT | Пересоздание коллекции |
| `CollectionUpdateAction` | UPDATE | Изменение элементов коллекции |

### Порядок выполнения операций

Hibernate выполняет операции в очереди в фиксированном порядке:
1. OrphanRemovalAction (удаление orphans)
2. EntityInsertAction (INSERT)
3. EntityUpdateAction (UPDATE)
4. CollectionRemoveAction (DELETE из join-таблицы)
5. CollectionRecreateAction (INSERT в join-таблицу)
6. EntityDeleteAction (DELETE)

> [!INFO]
> Именно поэтому `hibernate.order_inserts=true` — важная настройка для batch: без неё INSERT-ы могут перемежаться с UPDATE-ами, разбивая batch-группы.

---

## 4. FlushMode: AUTO, COMMIT, MANUAL, ALWAYS <a name="flushmode"></a>

`FlushMode` определяет **когда** Hibernate отправляет SQL из ActionQueue в базу данных.

```java
// Установка FlushMode на уровне EntityManager
entityManager.setFlushMode(FlushModeType.COMMIT);

// Или через Session (Hibernate-специфично)
session.setHibernateFlushMode(FlushMode.MANUAL);
```

### FlushMode.AUTO (по умолчанию)

Flush выполняется:
- Перед выполнением JPQL/HQL-запроса, если запрос затрагивает изменённые таблицы
- При commit транзакции

```java
@Transactional
public void example() {
    User user = entityManager.find(User.class, 1L);
    user.setName("New Name");
    
    // AUTO: Hibernate сделает flush ПЕРЕД этим запросом,
    // потому что users таблица "грязная"
    List<User> users = entityManager.createQuery("FROM User", User.class)
        .getResultList();
    // Запрос вернёт уже обновлённые данные
}
```

> [!WARNING]
> FlushMode.AUTO может вызвать неожиданный flush внутри транзакции при выполнении JPQL-запроса. Это создаёт промежуточный SQL до commit — может нарушить логику транзакции при сложных сценариях.

### FlushMode.COMMIT

Flush выполняется **только при commit**. Запросы внутри транзакции не триггерят flush.

```java
session.setHibernateFlushMode(FlushMode.COMMIT);

User user = session.get(User.class, 1L);
user.setName("New Name");

// При COMMIT — flush НЕ произойдёт перед этим запросом
// Запрос вернёт СТАРОЕ значение name из БД (или из L1 cache)
List<User> users = session.createQuery("FROM User", User.class).list();
```

**Когда использовать:** Long-running transactions, read-mostly сценарии, когда нужно избежать промежуточных flush-ов.

### FlushMode.MANUAL

Flush происходит **только при явном вызове** `session.flush()` или `entityManager.flush()`.

```java
session.setHibernateFlushMode(FlushMode.MANUAL);

User user = session.get(User.class, 1L);
user.setName("New Name");

// Flush не произойдёт даже при commit!
// Нужен явный вызов:
session.flush();
```

**Когда использовать:** Batch-операции с ручным контролем, read-only сессии (совместно с `setReadOnly(true)`), тесты.

### FlushMode.ALWAYS

Flush выполняется **перед каждым запросом** (даже если нет изменений).

**Когда использовать:** Редко. Только если нужна абсолютная гарантия видимости изменений для каждого запроса. Негативно влияет на производительность.

### Сравнительная таблица

| FlushMode | При JPQL-запросе | При commit | Управление |
|-----------|-----------------|------------|------------|
| AUTO | Если таблица dirty | Да | Автоматически |
| COMMIT | Нет | Да | Автоматически |
| MANUAL | Нет | Нет | Вручную |
| ALWAYS | Всегда | Да | Автоматически |

---

## 5. Подводные камни <a name="ловушки"></a>

### session.update() на attached entity — лишняя операция

```java
// ПЛОХО: session.update() на attached entity — избыточно
@Transactional
public void updateUser(Long id) {
    User user = entityManager.find(User.class, id); // теперь managed
    user.setEmail("new@mail.com");
    entityManager.merge(user); // лишняя операция! entity и так managed
}

// ХОРОШО: просто меняем поле — dirty checking сделает всё сам
@Transactional
public void updateUser(Long id) {
    User user = entityManager.find(User.class, id);
    user.setEmail("new@mail.com");
    // flush произойдёт автоматически при commit
}
```

> [!WARNING]
> `merge()` нужен только для **detached** entities (загруженных вне текущей транзакции). Вызов `merge()` на managed entity — лишний вызов, но не ошибка: Hibernate просто вернёт тот же managed объект.

### Flush при JPQL внутри транзакции (AUTO)

```java
@Transactional
public void dangerousExample() {
    User user = entityManager.find(User.class, 1L);
    user.setRole(Role.ADMIN); // изменение

    // FlushMode.AUTO: Hibernate видит, что users таблица изменена
    // → делает flush ПЕРЕД запросом → UPDATE выполняется в середине транзакции
    long count = entityManager.createQuery(
        "SELECT COUNT(u) FROM User u WHERE u.role = :r", Long.class)
        .setParameter("r", Role.ADMIN)
        .getSingleResult();
    
    if (count > 10) {
        // rollback... но UPDATE уже выполнен в БД в рамках той же транзакции
        throw new RuntimeException("Too many admins!");
        // Исключение → транзакция откатится → UPDATE тоже откатится. Всё ок.
        // Но если между flush и rollback произошло нечто асинхронное — могут быть нюансы
    }
}
```

### Read-only оптимизация: отключение Dirty Checking

```java
// Если сессия только читает — отключаем dirty checking
session.setDefaultReadOnly(true);
// Hibernate не будет делать snapshot и не будет проверять изменения при flush
// Экономия памяти и CPU при больших наборах данных
```

---

**Связанные файлы:**
- [[Batch операции]] — bulk операции без Dirty Checking
- [[StatelessSession]] — сессия без Dirty Checking и PC
- [[JPA Transactions]] — управление транзакциями
