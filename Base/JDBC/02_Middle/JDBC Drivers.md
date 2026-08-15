# JDBC Drivers: Справочник по драйверам

> [!QUOTE] JDBC-драйвер — библиотека, реализующая интерфейсы JDBC для конкретной СУБД. Преобразует вызовы Java API в протокол конкретной базы данных.

## Оглавление
1. [[#Типы драйверов]]
2. [[#Таблица популярных драйверов]]
3. [[#Особенности для разных СУБД]]
4. [[#Best practices]]

---

## Типы драйверов

| Тип | Описание | Рекомендуется |
|-----|----------|---------------|
| 1 | JDBC-ODBC Bridge | Нет (устарел) |
| 2 | Native-API / partly Java | Нет |
| 3 | Network Protocol | Нет |
| 4 | Pure Java | **Да** |

> [!INFO] Type 4 — единственный тип, рекомендованный для современных приложений. Работает без нативных библиотек, переносим между ОС.

---

## Таблица популярных драйверов

| СУБД | Maven Artifact | Класс драйвера |
|------|---------------|----------------|
| MySQL | `mysql:mysql-connector-java` | `com.mysql.cj.jdbc.Driver` |
| PostgreSQL | `org.postgresql:postgresql` | `org.postgresql.Driver` |
| Oracle | `com.oracle.database.jdbc:ojdbc8` | `oracle.jdbc.OracleDriver` |
| SQL Server | `com.microsoft.sqlserver:mssql-jdbc` | `com.microsoft.sqlserver.jdbc.SQLServerDriver` |
| H2 | `com.h2database:h2` | `org.h2.Driver` |
| SQLite | `org.xerial:sqlite-jdbc` | `org.sqlite.JDBC` |
| MariaDB | `org.mariadb.jdbc:mariadb-java-client` | `org.mariadb.jdbc.Driver` |

> [!WARNING] Версия драйвера должна соответствовать версии СУБД. Несовместимость версий — частая причина ошибок протокола и SSL.

---

## Особенности для разных СУБД

- **MySQL**: параметры URL — `useSSL=true`, `serverTimezone=UTC`, `characterEncoding=UTF-8`
- **PostgreSQL**: поддержка расширенных типов (`array`, `JSONB`), SSL через `sslmode=require`
- **Oracle**: требует лицензии; поддержка RAC и Advanced Security
- **SQL Server**: интеграция с Windows Auth (`integratedSecurity=true`), AlwaysOn
- **H2 / SQLite**: встраиваемые БД, используются в тестах; не подходят для production

> [!INFO] H2 поддерживает режим совместимости с MySQL/PostgreSQL: `MODE=MySQL` в URL. Удобно для тестирования без поднятия реальной СУБД.

---

## Best practices

- Только Type 4 драйверы
- Управляйте драйверами через Maven/Gradle — не кладите jar в репозиторий
- Проверяйте changelog драйвера при обновлении (особенно для MySQL Connector/J — были breaking changes)
- Совместимость: драйвер PostgreSQL 42.x+ поддерживает PostgreSQL 8.2+

---

**Связанные заметки:** [[Основы]] | [[Пула Соединений с JDBC]] | [[Безопасность в JDBC]]

**Материалы:**
- [JDBC Driver List (Wikipedia)](https://en.wikipedia.org/wiki/List_of_JDBC_drivers)
- [Spring Data JDBC Reference](https://docs.spring.io/spring-data/jdbc/docs/current/reference/html/)
