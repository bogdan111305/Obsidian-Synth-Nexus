# Безопасность в JDBC

> [!QUOTE] Три основных вектора угроз в JDBC: SQL-инъекции через `Statement`, утечка credentials из кода/репозитория, незашифрованное соединение с БД.

## Оглавление
1. [[#SQL-инъекции и защита]]
2. [[#SSL и защищённые соединения]]
3. [[#Хранение паролей и credentials]]
4. [[#Минимальные права пользователя БД]]
5. [[#Best practices]]

---

## SQL-инъекции и защита

SQL-инъекция — подмена SQL-логики через пользовательский ввод.

> [!WARNING] **Никогда** не конкатенируйте пользовательский ввод в строку SQL:
> ```java
> // УЯЗВИМО — SQL Injection
> String sql = "SELECT * FROM users WHERE name = '" + userInput + "'";
> Statement stmt = conn.createStatement();
> stmt.executeQuery(sql);
> ```
> Атакующий передаст `' OR '1'='1` и получит все записи. Или `'; DROP TABLE users; --`.

Защита — только `PreparedStatement` с параметрами `?`:

```java
// БЕЗОПАСНО
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE name = ?"
);
ps.setString(1, userInput);
ResultSet rs = ps.executeQuery();
```

`PreparedStatement` передаёт параметры отдельно от SQL — драйвер экранирует их автоматически.

> [!INFO] `PreparedStatement` также быстрее `Statement` при повторных запросах: SQL компилируется один раз, затем переиспользуется с разными параметрами.

---

## SSL и защищённые соединения

Без SSL данные (включая пароли) передаются в открытом виде по сети.

- **MySQL**: добавьте `?useSSL=true&requireSSL=true` в JDBC URL
- **PostgreSQL**: добавьте `?sslmode=require` в JDBC URL
- **Oracle / SQL Server**: используйте соответствующие параметры SSL в URL

> [!WARNING] Self-signed сертификаты допустимы только в dev/test окружениях. В production используйте сертификаты от доверенного CA или корпоративного PKI.

---

## Хранение паролей и credentials

> [!WARNING] Пароль БД в исходном коде или `application.properties`, закоммиченный в репозиторий — критическая уязвимость. Git-история не удаляется обычными средствами.

Правильный порядок:
1. **Переменные окружения**: `DB_PASSWORD=...` в окружении процесса
2. **Secrets manager**: HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets
3. **CI/CD secrets**: GitHub Actions secrets, GitLab CI variables

```properties
# application.properties
spring.datasource.password=${DB_PASSWORD}
```

---

## Минимальные права пользователя БД

Пользователь БД, от имени которого работает приложение, должен иметь только необходимые права:

- `SELECT`, `INSERT`, `UPDATE`, `DELETE` на нужные таблицы
- Запрет `DROP`, `ALTER`, `CREATE` — если не нужны приложению
- Отдельный пользователь для migration-инструментов (Flyway, Liquibase)

> [!INFO] Принцип минимальных привилегий: если приложение скомпрометировано, атакующий получит только права БД-пользователя. `DROP TABLE` будет невозможен.

---

## Best practices

- `PreparedStatement` везде, без исключений
- SSL для всех production-соединений
- Credentials только через переменные окружения или secrets manager
- Минимальные права пользователя БД
- Логируйте неудачные попытки подключения (на уровне приложения и СУБД)
- Регулярно ротируйте пароли БД

---

**Связанные заметки:** [[Основы]] | [[JDBC Drivers]] | [[Пула Соединений с JDBC]]

**Материалы:**
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [Spring Security Data Access](https://docs.spring.io/spring-security/reference/servlet/integrations/data.html)
