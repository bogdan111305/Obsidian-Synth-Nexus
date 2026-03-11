# Ошибки и диагностика в JDBC

> [!QUOTE] `SQLException` — базовый класс всех JDBC-ошибок. Содержит три диагностических поля: `message`, `SQLState` (стандартный 5-символьный код), `errorCode` (специфичный для СУБД).

## Оглавление
1. [[#Иерархия исключений]]
2. [[#SQLState и коды ошибок]]
3. [[#Диагностика и логирование]]
4. [[#Типовые сценарии и решения]]
5. [[#Best practices]]

---

## Иерархия исключений

```
SQLException
├── SQLNonTransientException
│   ├── SQLSyntaxErrorException      — ошибка синтаксиса SQL
│   ├── SQLDataException             — ошибки данных (тип, диапазон)
│   └── SQLNonTransientConnectionException  — соединение не восстановить
├── SQLTransientException
│   ├── SQLTimeoutException          — таймаут запроса или соединения
│   ├── SQLTransactionRollbackException     — deadlock, откат
│   └── SQLTransientConnectionException     — соединение потеряно, retry возможен
└── SQLWarning                       — предупреждение (data truncation и др.)
```

> [!INFO] `SQLTransientException` — сигнал, что операцию можно повторить (retry). `SQLNonTransientException` — повтор бесполезен, нужно исправить код или данные.

---

## SQLState и коды ошибок

SQLState — стандартный 5-символьный код (ISO/ANSI SQL):

| Префикс | Категория |
|---------|-----------|
| `08xxx` | Ошибки соединения |
| `22xxx` | Ошибки данных (тип, диапазон, null) |
| `23xxx` | Нарушение ограничений (unique, FK) |
| `40xxx` | Rollback транзакции (deadlock) |
| `42xxx` | Синтаксические ошибки SQL |

Специфичные коды СУБД:
- MySQL deadlock: `SQLState=40001`, `ErrorCode=1213`
- MySQL connection refused: `SQLState=08001`
- PostgreSQL unique violation: `SQLState=23505`

```java
catch (SQLException e) {
    String state = e.getSQLState();
    if ("40001".equals(state)) {
        // deadlock — можно повторить
    } else if (state != null && state.startsWith("08")) {
        // ошибка соединения
    }
}
```

---

## Диагностика и логирование

Базовый шаблон обработки:

```java
try {
    // JDBC-операции
} catch (SQLException e) {
    logger.error(
        "JDBC error. SQLState: {}, ErrorCode: {}, Message: {}",
        e.getSQLState(), e.getErrorCode(), e.getMessage(), e
    );
    // обработка или re-throw
}
```

Цепочка ошибок — `SQLException` может содержать связанные исключения:

```java
SQLException ex = e;
while (ex != null) {
    logger.error("SQLState: {}, Code: {}, Msg: {}",
        ex.getSQLState(), ex.getErrorCode(), ex.getMessage());
    ex = ex.getNextException();
}
```

Инструменты диагностики:
- **MySQL**: `SHOW ENGINE INNODB STATUS` — для анализа deadlock
- **PostgreSQL**: `pg_stat_activity` — активные запросы и блокировки
- **VisualVM / YourKit** — профилирование соединений
- **P6Spy / datasource-proxy** — логирование SQL с параметрами

> [!INFO] Включите SQL-логирование в драйвере для разработки. MySQL: добавьте `logger=com.mysql.cj.log.Slf4JLogger&profileSQL=true` в URL. Не оставляйте в production — генерирует огромные логи.

---

## Типовые сценарии и решения

### Deadlock

```
SQLState=40001, ErrorCode=1213 (MySQL)
```

- Причина: два потока ждут ресурсов, занятых друг другом
- Решение: retry с экспоненциальным backoff

```java
int maxRetries = 3;
for (int attempt = 0; attempt < maxRetries; attempt++) {
    try {
        performTransaction(conn);
        break;
    } catch (SQLException e) {
        if ("40001".equals(e.getSQLState()) && attempt < maxRetries - 1) {
            Thread.sleep(100L * (1L << attempt)); // 100ms, 200ms, 400ms
        } else {
            throw e;
        }
    }
}
```

### Connection refused / timeout

```
SQLState=08001 / SQLState=08S01
```

- Причина: СУБД недоступна, firewall, исчерпан пул
- Решение: проверьте сетевую доступность, параметры пула, `connectionTimeout`

> [!WARNING] Connection pool exhausted (`connectionTimeout` истёк) — признак, что либо `maximumPoolSize` мал, либо соединения утекают (не закрываются). Диагностируйте через метрики HikariCP.

### Data truncation / constraint violation

```
SQLState=22001 (data too long) / SQLState=23000 (constraint violation)
```

- Причина: данные не соответствуют схеме БД
- Решение: валидация данных до отправки в БД

### Connection is closed

- Причина: СУБД закрыла соединение по `wait_timeout`, а пул не знал об этом
- Решение: настройте `maxLifetime` в пуле меньше `wait_timeout` СУБД

---

## Best practices

- Логируйте все `SQLException` с `SQLState` и `ErrorCode`
- Для `SQLTransientException` реализуйте retry с backoff
- Для deadlock — анализируйте порядок блокировок и меняйте логику
- Используйте централизованное логирование (ELK, Loki) в production
- Не проглатывайте исключения пустым `catch`
- Проверяйте `getNextException()` — может содержать корневую причину

---

**Связанные заметки:** [[Основы]] | [[Транзакции в JDBC]] | [[Пула Соединений с JDBC]]

**Материалы:**
- [JDBC Exception Handling](https://docs.oracle.com/javase/tutorial/jdbc/basics/sqlexception.html)
- [MySQL Error Codes](https://dev.mysql.com/doc/refman/8.0/en/server-error-reference.html)
- [PostgreSQL Error Codes](https://www.postgresql.org/docs/current/errcodes-appendix.html)
