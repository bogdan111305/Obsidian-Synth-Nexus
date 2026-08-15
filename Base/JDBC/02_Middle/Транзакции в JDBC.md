# Транзакции в JDBC

> [!QUOTE] Транзакция — группа SQL-операций, которые выполняются как единое целое: либо все успешно, либо все откатываются. Гарантии — ACID: Atomicity, Consistency, Isolation, Durability.

## Оглавление
1. [[#ACID]]
2. [[#Управление транзакциями в JDBC]]
3. [[#Уровни изоляции]]
4. [[#Savepoints]]
5. [[#Интеграция с Spring]]
6. [[#Распределённые транзакции]]
7. [[#Ошибки и диагностика]]
8. [[#Best practices]]
9. [[#Вопросы для собеседования]]

---

## ACID

- **Atomicity** — всё или ничего: если одна операция провалилась, откатывается вся группа
- **Consistency** — данные всегда переходят из одного валидного состояния в другое
- **Isolation** — параллельные транзакции не видят незафиксированных изменений друг друга
- **Durability** — после `commit` данные сохранены даже при сбое системы

---

## Управление транзакциями в JDBC

> [!WARNING] По умолчанию `autocommit = true`: каждый SQL-запрос немедленно фиксируется. Для работы с транзакцией необходимо явно отключить autocommit.

```java
try (Connection conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);                          // начало транзакции
    try {
        // SQL-операции
        conn.prepareStatement("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
            .executeUpdate();
        conn.prepareStatement("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
            .executeUpdate();
        conn.commit();                                  // фиксация
    } catch (SQLException e) {
        conn.rollback();                                // откат при ошибке
        throw e;
    }
}
```

> [!INFO] `try-with-resources` закрывает `Connection`, но не откатывает незавершённую транзакцию автоматически — это нужно сделать явно в `catch`. HikariCP откатит её при возврате в пул, но полагаться на это — плохая практика.

---

## Уровни изоляции

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read |
|---------|------------|----------------------|--------------|
| `READ_UNCOMMITTED` | Да | Да | Да |
| `READ_COMMITTED` | Нет | Да | Да |
| `REPEATABLE_READ` | Нет | Нет | Да |
| `SERIALIZABLE` | Нет | Нет | Нет |

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

- **Dirty Read** — читаем незафиксированные данные другой транзакции
- **Non-Repeatable Read** — два одинаковых `SELECT` в одной транзакции дают разный результат
- **Phantom Read** — повторный запрос возвращает другое количество строк

> [!INFO] `READ_COMMITTED` — рекомендуемый уровень для большинства OLTP-систем: баланс между согласованностью и производительностью. `SERIALIZABLE` — максимальная изоляция, но высокий риск deadlock и низкая пропускная способность.

---

## Savepoints

Savepoint позволяет откатить только часть транзакции, не отменяя всё целиком.

```java
conn.setAutoCommit(false);

Savepoint sp = conn.setSavepoint("checkpoint1");
try {
    conn.prepareStatement("INSERT INTO logs (msg) VALUES ('step1')").executeUpdate();
    // рискованная операция
    conn.prepareStatement("UPDATE critical_table SET x = 1").executeUpdate();
    conn.commit();
} catch (SQLException e) {
    conn.rollback(sp);      // откат только до savepoint, не всей транзакции
    conn.commit();          // зафиксировать то, что было до savepoint
}
```

> [!INFO] JDBC не поддерживает настоящие вложенные транзакции. Savepoints — ближайший эквивалент для частичного отката.

---

## Интеграция с Spring

Spring управляет транзакциями через `@Transactional`:

```java
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    jdbcTemplate.update(
        "UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, fromId
    );
    jdbcTemplate.update(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, toId
    );
}
```

```java
// Уровень изоляции через аннотацию
@Transactional(isolation = Isolation.SERIALIZABLE)
public void criticalOperation() { ... }
```

> [!WARNING] `@Transactional` работает только при вызове метода **через Spring-прокси** (то есть из другого бина). Вызов `this.method()` внутри того же класса обходит прокси — транзакция не откроется.

> [!INFO] По умолчанию Spring откатывает транзакцию только при `RuntimeException` и `Error`. Для `checked Exception` нужно явно указать: `@Transactional(rollbackFor = Exception.class)`.

---

## Распределённые транзакции

**XA-транзакции** — для координации нескольких ресурсов (несколько БД, JMS и БД):

```java
XADataSource xaDs = ...;
XAConnection xaConn = xaDs.getXAConnection();
XAResource xaResource = xaConn.getXAResource();
// координируется через TransactionManager (Atomikos, Bitronix)
```

**Saga-паттерн** — для микросервисов: последовательность локальных транзакций с компенсирующими операциями при сбое. Нет двухфазного коммита — выше производительность, но сложнее логика компенсации.

> [!INFO] XA-транзакции в микросервисной архитектуре не рекомендуются: высокая сложность и latency. Предпочтительнее Saga через Kafka-события или Orchestration (Temporal, Axon).

---

## Ошибки и диагностика

```java
try {
    conn.setAutoCommit(false);
    // ...
    conn.commit();
} catch (SQLException e) {
    try { conn.rollback(); } catch (SQLException re) { /* логировать */ }
    logger.error("Transaction failed. SQLState: {}, Code: {}",
        e.getSQLState(), e.getErrorCode(), e);
    throw e;
}
```

Типовые проблемы:
- **Deadlock** — SQLState `40001`, код MySQL `1213`. Решение: retry с backoff
- **Timeout** — SQLState `08xxx`. Оптимизируйте запрос или увеличьте timeout
- **Connection lost** — транзакция прервана. Требует rollback и повтора

Подробнее: [[Ошибки и диагностика в JDBC]]

---

## Best practices

- Минимизируйте время выполнения транзакции — держите её как можно короче
- Всегда реализуйте `rollback` в `catch`
- Не смешивайте бизнес-логику и управление транзакциями в одном методе
- Используйте пул соединений — [[Пула Соединений с JDBC]]
- Логируйте откаты с `SQLState` и контекстом операции
- Для deadlock реализуйте retry с экспоненциальным backoff

---

## Вопросы для собеседования

1. Что такое ACID? Объясните каждый принцип.
2. Что происходит при `autocommit = true`?
3. Как реализовать транзакцию в чистом JDBC?
4. Какой уровень изоляции выбрать для OLTP?
5. Что такое Savepoint и когда его использовать?
6. Как Spring управляет транзакциями? Подводные камни `@Transactional`.
7. Чем Saga отличается от XA-транзакций?

---

**Связанные заметки:** [[Основы]] | [[Пула Соединений с JDBC]] | [[Ошибки и диагностика в JDBC]]

**Материалы:**
- [JDBC Transactions](https://docs.oracle.com/javase/tutorial/jdbc/basics/transactions.html)
- [Spring Transaction Management](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)
- [XA Transactions](https://docs.oracle.com/javase/8/docs/api/javax/transaction/xa/XAResource.html)
