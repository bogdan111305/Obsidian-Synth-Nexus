# JDBC Drivers: Справочник по драйверам для популярных СУБД

## Оглавление
1. [Что такое JDBC-драйвер?](#что-это)
2. [Типы драйверов](#типы)
3. [Таблица популярных драйверов](#таблица)
4. [Особенности для разных СУБД](#особенности)
5. [Best practices](#best-practices)
6. [FAQ](#faq)

---

## 1. Что такое JDBC-драйвер? <a name="что-это"></a>
JDBC-драйвер — это библиотека, реализующая интерфейсы JDBC для конкретной СУБД. Он преобразует вызовы Java в протокол СУБД.

## 2. Типы драйверов <a name="типы"></a>
- Type 1: JDBC-ODBC Bridge (устарел)
- Type 2: Native-API/partly Java
- Type 3: Network Protocol
- Type 4: Pure Java (рекомендуется)

## 3. Таблица популярных драйверов <a name="таблица"></a>
| СУБД         | Группа/Артефакт (Maven)                | Класс драйвера                      | Ссылка |
|--------------|----------------------------------------|-------------------------------------|--------|
| MySQL        | mysql:mysql-connector-java             | com.mysql.cj.jdbc.Driver            | [MySQL](https://dev.mysql.com/downloads/connector/j/)
| PostgreSQL   | org.postgresql:postgresql              | org.postgresql.Driver               | [PostgreSQL](https://jdbc.postgresql.org/)
| Oracle       | com.oracle.database.jdbc:ojdbc8        | oracle.jdbc.OracleDriver            | [Oracle](https://www.oracle.com/database/technologies/appdev/jdbc.html)
| SQL Server   | com.microsoft.sqlserver:mssql-jdbc     | com.microsoft.sqlserver.jdbc.SQLServerDriver | [MS SQL](https://docs.microsoft.com/sql/connect/jdbc/download-microsoft-jdbc-driver-for-sql-server)
| H2           | com.h2database:h2                      | org.h2.Driver                       | [H2](https://www.h2database.com/html/main.html)
| SQLite       | org.xerial:sqlite-jdbc                 | org.sqlite.JDBC                     | [SQLite](https://github.com/xerial/sqlite-jdbc)
| MariaDB      | org.mariadb.jdbc:mariadb-java-client   | org.mariadb.jdbc.Driver             | [MariaDB](https://mariadb.com/kb/en/mariadb/about-mariadb-connector-j/)

## 4. Особенности для разных СУБД <a name="особенности"></a>
- **MySQL**: поддержка SSL, timezone, Unicode, параметры useSSL, serverTimezone
- **PostgreSQL**: поддержка расширенных типов, array, JSONB, SSL
- **Oracle**: требует лицензии, поддержка RAC, advanced security
- **SQL Server**: интеграция с Windows Auth, поддержка AlwaysOn
- **H2/SQLite**: встраиваемые, удобны для тестов

## 5. Best practices <a name="best-practices"></a>
- Используйте только Type 4 драйверы
- Следите за совместимостью версии драйвера и СУБД
- Не храните jar-драйверы в репозитории, используйте dependency manager
- Проверяйте changelog драйвера при обновлении

## 6. FAQ <a name="faq"></a>
- Как выбрать драйвер для своей СУБД?
- Как узнать класс драйвера?
- Как подключить драйвер в Maven/Gradle?
- Как проверить версию драйвера в рантайме?

---

**Рекомендуемые материалы:**
- [JDBC Driver List](https://en.wikipedia.org/wiki/List_of_JDBC_drivers)
- [Spring Data JDBC Reference](https://docs.spring.io/spring-data/jdbc/docs/current/reference/html/) 