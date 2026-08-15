# Read-Only транзакции и нетранзакционный доступ в JPA/Hibernate

## Оглавление
1. [Read-Only транзакции: зачем нужны и как работают](#readonly)
2. [ReadOnly mode на уровне приложения](#app-readonly)
3. [ReadOnly mode на уровне базы данных](#db-readonly)
4. [Nontransactional Data Access: AutoCommit](#autocommit)
5. [Особенности для IDENTITY сущностей](#identity)
6. [Best practices](#best-practices)
7. [Вопросы для собеседования](#вопросы)

---

## 1. Read-Only транзакции: зачем нужны и как работают <a name="readonly"></a>

**Read-only транзакция** — это транзакция, в рамках которой гарантируется только чтение данных без их изменения. Используется для оптимизации производительности и предотвращения случайных изменений.

- Позволяет БД оптимизировать выполнение запросов
- Hibernate может не отслеживать dirty state сущностей

## 2. ReadOnly mode на уровне приложения <a name="app-readonly"></a>

### В Spring:
```java
@Transactional(readOnly = true)
public List<User> findAllUsers() {
    return userRepository.findAll();
}
```
- Hibernate не будет отслеживать изменения сущностей (flush не выполняется)
- Повышает производительность при больших выборках

## 3. ReadOnly mode на уровне базы данных <a name="db-readonly"></a>

- Некоторые СУБД поддерживают read-only транзакции на уровне SQL:
```sql
START TRANSACTION READ ONLY;
SELECT * FROM users;
COMMIT;
```
- В PostgreSQL:
```sql
BEGIN TRANSACTION READ ONLY;
-- ...
COMMIT;
```
- БД может оптимизировать блокировки и кэширование

## 4. Nontransactional Data Access: AutoCommit <a name="autocommit"></a>

- **AutoCommit** — режим, при котором каждая операция выполняется в своей транзакции
- В JDBC по умолчанию включён AutoCommit
- В Hibernate/JPA рекомендуется отключать AutoCommit и управлять транзакциями вручную

### AutoCommit для чтения сущностей
- Можно использовать для простых SELECT-запросов, но не рекомендуется для сложных операций

### AutoCommit для изменения сущностей
- Не рекомендуется: возможны проблемы с целостностью данных и производительностью

### Особенность: AutoCommit работает только для IDENTITY сущностей
- Для сущностей с генерацией ключа через IDENTITY (например, PostgreSQL SERIAL) Hibernate может выполнять insert сразу после persist
- Это может привести к неожиданным коммитам вне общей транзакции

## 5. Особенности для IDENTITY сущностей <a name="identity"></a>

- При использовании генерации ключа через IDENTITY (например, @GeneratedValue(strategy = GenerationType.IDENTITY))
- Hibernate выполняет insert сразу после persist, чтобы получить сгенерированный ключ
- Это может привести к частичному коммиту даже в рамках общей транзакции
- Для batch-операций лучше использовать SEQUENCE или TABLE генерацию

## 6. Best practices <a name="best-practices"></a>
- Для запросов только на чтение используйте @Transactional(readOnly = true)
- Не используйте AutoCommit для сложных операций
- Для batch-операций избегайте IDENTITY-генерации
- Явно управляйте транзакциями в коде
- Проверяйте, что read-only транзакции действительно не изменяют данные

## 7. Вопросы для собеседования <a name="вопросы"></a>
1. Зачем нужны read-only транзакции?
2. Как включить read-only режим в Spring/Hibernate?
3. Чем опасен AutoCommit для изменения данных?
4. Почему генерация ключа через IDENTITY может быть проблемной?
5. Как оптимизировать batch-операции с точки зрения транзакций?

---

**Дополнительные ресурсы:**
- [Spring: Read-Only Transactions](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction-declarative-annotations)
- [Hibernate: Read-Only Entities](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#readonly)
- [JPA: Transaction Management](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#transactions)
