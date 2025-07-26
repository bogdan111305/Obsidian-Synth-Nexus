# JPA Transactions: подробное руководство

## Оглавление
1. [Что такое транзакция в JPA и Hibernate](#что-такое)
2. [Интерфейсы Transaction и EntityTransaction](#интерфейсы)
3. [Работа с транзакциями в Hibernate](#hibernate)
4. [Декларативное управление транзакциями: @Transactional](#transactional)
5. [Уровни изоляции транзакций](#isolation)
6. [Best practices](#best-practices)
7. [Вопросы для собеседования](#вопросы)

---

## 1. Что такое транзакция в JPA и Hibernate <a name="что-такое"></a>

Транзакция — это последовательность операций, которые должны быть выполнены как единое целое. В JPA/Hibernate транзакции управляют целостностью данных при работе с EntityManager/Session.

## 2. Интерфейсы Transaction и EntityTransaction <a name="интерфейсы"></a>

- **javax.persistence.EntityTransaction** — для управления транзакциями в JPA (standalone, RESOURCE_LOCAL)
- **org.hibernate.Transaction** — для управления транзакциями в Hibernate (Session API)

### Пример (JPA):
```java
EntityManager em = emf.createEntityManager();
EntityTransaction tx = em.getTransaction();
try {
    tx.begin();
    // ... работа с сущностями ...
    tx.commit();
} catch (Exception e) {
    tx.rollback();
}
```

### Пример (Hibernate):
```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
try {
    // ... работа с сущностями ...
    tx.commit();
} catch (Exception e) {
    tx.rollback();
}
```

## 3. Работа с транзакциями в Hibernate <a name="hibernate"></a>

- Все изменения сущностей происходят в рамках транзакции
- В режиме AUTO-COMMIT (JDBC) Hibernate сам начинает и завершает транзакцию для каждого запроса (не рекомендуется)
- В большинстве случаев транзакции управляются вручную или через контейнер (Spring, EJB)

### Пример (Spring + Hibernate):
```java
@Service
public class UserService {
    @Transactional
    public void updateUser(User user) {
        // ...
    }
}
```

## 4. Декларативное управление транзакциями: @Transactional <a name="transactional"></a>

- Аннотация `@Transactional` (Spring, Jakarta EE) позволяет управлять транзакциями декларативно
- Можно указывать propagation, isolation, readOnly, rollbackFor и др.

### Пример:
```java
@Transactional(isolation = Isolation.REPEATABLE_READ, rollbackFor = Exception.class)
public void processOrder(Order order) {
    // ...
}
```

#### Основные параметры:
- **propagation** — как транзакция будет вести себя при вложенных вызовах
- **isolation** — уровень изоляции
- **readOnly** — только для чтения
- **timeout** — таймаут транзакции
- **rollbackFor** — список исключений для отката

## 5. Уровни изоляции транзакций <a name="isolation"></a>

- **DEFAULT** — уровень по умолчанию (обычно READ_COMMITTED)
- **READ_UNCOMMITTED**
- **READ_COMMITTED**
- **REPEATABLE_READ**
- **SERIALIZABLE**

### Пример установки уровня изоляции:
```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void criticalOperation() {
    // ...
}
```

## 6. Best practices <a name="best-practices"></a>
- Открывайте транзакцию только на время работы с БД
- Не держите транзакцию дольше, чем нужно (не вызывайте внешние сервисы внутри транзакции)
- Используйте readOnly для запросов только на чтение
- Обрабатывайте rollback в catch-блоках
- Для batch-операций используйте chunking и периодический commit
- Логируйте ошибки и откаты транзакций

## 7. Вопросы для собеседования <a name="вопросы"></a>
1. Как управлять транзакциями в JPA/Hibernate?
2. Чем отличается EntityTransaction от Transaction?
3. Как работает @Transactional? Какие параметры можно указать?
4. Какой уровень изоляции по умолчанию?
5. Как откатить транзакцию вручную?
6. Какие best practices при работе с транзакциями?

---

**Дополнительные ресурсы:**
- [Spring: Transaction Management](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)
- [Hibernate: Transactions and Concurrency](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#transactions)