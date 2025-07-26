# Транзакции в JDBC: Полное руководство

## Оглавление
1. [Что такое транзакция? ACID](#acid)
2. [Управление транзакциями в JDBC](#управление)
3. [Уровни изоляции транзакций](#изоляция)
4. [Savepoints и вложенные транзакции](#savepoints)
5. [Интеграция с Spring](#spring)
6. [Распределённые транзакции (XA, Saga)](#распределённые)
7. [Ошибки и диагностика](#ошибки)
8. [Best practices](#best-practices)
9. [FAQ и вопросы для собеседования](#faq)

---

## 1. Что такое транзакция? ACID <a name="acid"></a>
- **Atomicity**: всё или ничего
- **Consistency**: данные всегда валидны
- **Isolation**: параллельные транзакции не мешают друг другу
- **Durability**: изменения не теряются после commit

## 2. Управление транзакциями в JDBC <a name="управление"></a>
- По умолчанию: autocommit=true (каждый запрос — отдельная транзакция)
- Для ручного управления:
```java
conn.setAutoCommit(false);
// ... SQL ...
conn.commit(); // или conn.rollback();
```
- Всегда используйте try-with-resources и finally для возврата соединения и восстановления autocommit

## 3. Уровни изоляции транзакций <a name="изоляция"></a>
- READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE
- Пример:
```java
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
```
- Рекомендация: READ_COMMITTED — баланс между производительностью и согласованностью

## 4. Savepoints и вложенные транзакции <a name="savepoints"></a>
- Savepoint позволяет откатить только часть транзакции
```java
Savepoint sp = conn.setSavepoint("sp1");
// ...
conn.rollback(sp);
conn.commit();
```
- JDBC не поддерживает настоящие вложенные транзакции, но savepoints позволяют частичный откат

## 5. Интеграция с Spring <a name="spring"></a>
- Используйте @Transactional и JdbcTemplate
```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void updateProfileAndOrder(Long userId, String name) {
    jdbcTemplate.update("UPDATE users SET name = ? WHERE id = ?", name, userId);
    jdbcTemplate.update("INSERT INTO orders (user_id) VALUES (?)", userId);
}
```
- Spring автоматически откатывает транзакцию при исключениях

## 6. Распределённые транзакции (XA, Saga) <a name="распределённые"></a>
- Для микросервисов используйте XA-транзакции или Saga-паттерн
- Пример XA:
```java
import javax.transaction.xa.XAResource;
XAResource xaResource = conn.getXAResource();
// ...
```
- Saga: последовательность локальных транзакций с компенсацией

## 7. Ошибки и диагностика <a name="ошибки"></a>
- Типовые ошибки: deadlock, timeout, connection lost, inconsistent state
- Диагностика: анализируйте SQLState, используйте логирование, тестируйте rollback
- Пример:
```java
try {
    conn.setAutoCommit(false);
    // ...
    conn.commit();
} catch (SQLException e) {
    if (conn != null) conn.rollback();
    throw e;
}
```

## 8. Best practices <a name="best-practices"></a>
- Минимизируйте время выполнения транзакций
- Всегда реализуйте обработку rollback
- Используйте пул соединений для многопоточных приложений
- Не смешивайте бизнес-логику и управление транзакциями
- Логируйте ошибки и rollback

## 9. FAQ и вопросы для собеседования <a name="faq"></a>
### Часто задаваемые вопросы
- Как реализовать транзакцию в JDBC?
- Как выбрать уровень изоляции?
- Как откатить только часть транзакции?
- Как интегрировать транзакции с Spring?
- Как реализовать распределённую транзакцию?

### Вопросы для собеседования
1. Что такое ACID?
2. Как работает commit/rollback в JDBC?
3. Какой уровень изоляции выбрать для OLTP?
4. Как реализовать savepoint?
5. Как Spring управляет транзакциями?
6. Как диагностировать deadlock?

---

**Рекомендуемые материалы:**
- [JDBC Transactions](https://docs.oracle.com/javase/tutorial/jdbc/basics/transactions.html)
- [Spring Transaction Management](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)
- [XA Transactions](https://docs.oracle.com/javase/8/docs/api/javax/transaction/xa/XAResource.html)