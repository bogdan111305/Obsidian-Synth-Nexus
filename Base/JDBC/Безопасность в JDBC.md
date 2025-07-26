# Безопасность в JDBC

## Оглавление
1. [SQL-инъекции и защита](#sql-инъекции)
2. [SSL и защищённые соединения](#ssl)
3. [Хранение паролей и credentials](#пароли)
4. [Best practices](#best-practices)
5. [FAQ](#faq)

---

## 1. SQL-инъекции и защита <a name="sql-инъекции"></a>
- Используйте PreparedStatement вместо Statement
- Не конкатенируйте строки в SQL
- Пример:
```java
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, userInput);
```
- Валидация входных данных

## 2. SSL и защищённые соединения <a name="ssl"></a>
- Включайте SSL для production:
  - MySQL: ?useSSL=true
  - PostgreSQL: ?sslmode=require
- Используйте сертификаты, не доверяйте self-signed в бою
- Проверяйте настройки firewall

## 3. Хранение паролей и credentials <a name="пароли"></a>
- Не храните пароли в коде/репозитории
- Используйте переменные окружения, vault, секреты CI/CD
- Для Spring Boot: application.properties + externalized secrets

## 4. Best practices <a name="best-practices"></a>
- Всегда используйте PreparedStatement
- Включайте SSL для production
- Не храните пароли в коде
- Минимизируйте права пользователя БД
- Логируйте попытки неудачного доступа

## 5. FAQ <a name="faq"></a>
- Как защититься от SQL-инъекций?
- Как включить SSL для JDBC?
- Как безопасно хранить пароли?
- Как ограничить права пользователя БД?

---

**Рекомендуемые материалы:**
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [Spring Security Data Access](https://docs.spring.io/spring-security/reference/servlet/integrations/data.html) 