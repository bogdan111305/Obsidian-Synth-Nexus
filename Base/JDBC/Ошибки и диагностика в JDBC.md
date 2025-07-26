# Ошибки и диагностика в JDBC

## Оглавление
1. [Типовые ошибки](#ошибки)
2. [SQLState и коды ошибок](#sqlstate)
3. [Диагностика и логирование](#диагностика)
4. [Примеры логов и обработки](#примеры)
5. [Best practices](#best-practices)
6. [FAQ](#faq)

---

## 1. Типовые ошибки <a name="ошибки"></a>
- SQLException (общая ошибка)
- SQLSyntaxErrorException (ошибка синтаксиса SQL)
- SQLTimeoutException (таймаут соединения/запроса)
- SQLNonTransientConnectionException (потеря соединения)
- Deadlock (взаимная блокировка)
- Connection refused (база недоступна)
- Data truncation (обрезка данных)

## 2. SQLState и коды ошибок <a name="sqlstate"></a>
- SQLState — стандартный 5-символьный код ошибки
- Примеры:
  - 08xxx — ошибки соединения
  - 22xxx — ошибки данных
  - 23xxx — нарушения ограничений
- Используйте e.getSQLState() и e.getErrorCode() для диагностики

## 3. Диагностика и логирование <a name="диагностика"></a>
- Включайте логирование драйвера (например, logger=com.mysql.cj.log.Slf4JLogger)
- Анализируйте stacktrace и SQLState
- Используйте профилировщики (VisualVM, YourKit)
- Для deadlock — анализируйте логи СУБД (SHOW ENGINE INNODB STATUS для MySQL)

## 4. Примеры логов и обработки <a name="примеры"></a>
```java
try {
    // ...
} catch (SQLException e) {
    System.err.println("SQLState: " + e.getSQLState());
    System.err.println("Error Code: " + e.getErrorCode());
    e.printStackTrace();
}
```
- Пример deadlock в MySQL: SQLState=40001, ErrorCode=1213
- Пример connection refused: SQLState=08001

## 5. Best practices <a name="best-practices"></a>
- Логируйте все ошибки с SQLState и ErrorCode
- Для production — используйте централизованное логирование
- Для deadlock — реализуйте retry с backoff
- Для timeout — увеличьте таймаут или оптимизируйте запрос
- Не игнорируйте исключения!

## 6. FAQ <a name="faq"></a>
- Как узнать причину SQLException?
- Как диагностировать deadlock?
- Как отличить ошибку соединения от ошибки данных?
- Как логировать ошибки в Spring?

---

**Рекомендуемые материалы:**
- [JDBC Exception Handling](https://docs.oracle.com/javase/tutorial/jdbc/basics/sqlexception.html)
- [MySQL Error Codes](https://dev.mysql.com/doc/refman/8.0/en/server-error-reference.html)
- [PostgreSQL Error Codes](https://www.postgresql.org/docs/current/errcodes-appendix.html) 