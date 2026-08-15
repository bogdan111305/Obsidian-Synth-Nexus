# Schema Generation и Flyway

## Оглавление
1. [hibernate.hbm2ddl.auto: все режимы](#hbm2ddl)
2. [Почему update опасен в production](#update-danger)
3. [validate как правильный выбор для prod](#validate)
4. [Интеграция с Flyway](#flyway)
5. [Интеграция с Liquibase](#liquibase)

---

## 1. hibernate.hbm2ddl.auto: все режимы <a name="hbm2ddl"></a>

`hibernate.hbm2ddl.auto` (или `spring.jpa.hibernate.ddl-auto` в Spring Boot) управляет тем, как Hibernate работает со схемой БД при старте приложения.

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # рекомендуется для production
```

### none

```yaml
ddl-auto: none
```

Hibernate **не трогает** схему БД. Никаких DDL-операций. Используй в production с внешними миграциями.

### validate

```yaml
ddl-auto: validate
```

Hibernate **проверяет** соответствие схемы БД маппингу entity при старте. Если есть расхождение — выбрасывает исключение и приложение не стартует.

```
SchemaManagementException: Schema-validation: missing column [new_column] in table [users]
```

**Не изменяет** схему — только читает метаданные и сравнивает.

### update

```yaml
ddl-auto: update
```

Hibernate пытается **привести схему к соответствию** с текущим маппингом:
- Добавляет новые таблицы и колонки
- Добавляет индексы
- **НЕ** удаляет таблицы и колонки
- **НЕ** переименовывает колонки
- **НЕ** меняет типы данных

### create

```yaml
ddl-auto: create
```

При каждом старте **создаёт схему заново**. Существующие данные удаляются. Используется только в разработке.

### create-drop

```yaml
ddl-auto: create-drop
```

Создаёт схему при старте, **удаляет при остановке** `SessionFactory`. Идеально для тестов.

### Сравнительная таблица

| Режим | Создаёт | Изменяет | Удаляет | Валидирует | Применение |
|-------|---------|---------|---------|-----------|------------|
| none | Нет | Нет | Нет | Нет | Prod с внешними миграциями |
| validate | Нет | Нет | Нет | Да | Prod с Flyway/Liquibase |
| update | Да | Частично | Нет | Нет | Dev (осторожно) |
| create | Да | Да | Да (при старте) | Нет | Dev/local |
| create-drop | Да | Да | Да (при стопе) | Нет | Тесты |

---

## 2. Почему update опасен в production <a name="update-danger"></a>

> [!WARNING]
> **Никогда не используй `ddl-auto: update` в production!**

**Проблема 1: Не удаляет колонки**

```java
// Было:
@Column(name = "old_field")
private String oldField;

// Стало (переименовали поле):
@Column(name = "new_field")
private String newField;
```

`update` создаст новую колонку `new_field`, но `old_field` останется в БД навсегда. Данные потеряны — они остались в `old_field`, новые пишутся в `new_field`.

**Проблема 2: Не меняет типы данных**

```java
// Было: VARCHAR(50)
@Column(length = 50)
private String code;

// Стало: VARCHAR(100)
@Column(length = 100)
private String code;
```

`update` не изменит тип колонки — молча проигнорирует изменение длины.

**Проблема 3: Не контролируемые DDL**

DDL-операции `update` нигде не логируются и не версионируются. Невозможно откатить изменение схемы. Нет code review DDL изменений.

**Проблема 4: Race condition при горизонтальном масштабировании**

Если запускается несколько инстансов одновременно — все пытаются выполнить DDL. Результат непредсказуем.

---

## 3. validate как правильный выбор для prod <a name="validate"></a>

`validate` — оптимальный выбор для production:
- **Быстрый фейл**: если схема не соответствует — приложение не стартует немедленно (fail-fast)
- **Безопасный**: никаких DDL-изменений
- **Явный**: разработчик обязан управлять схемой через миграции

```yaml
# production application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

При несоответствии получишь чёткую ошибку при деплое:
```
HibernateException: Missing table: users
HibernateException: Missing column [email] in table [users]
HibernateException: Wrong column type: expected varchar(255), got text
```

---

## 4. Интеграция с Flyway <a name="flyway"></a>

**Flyway** — инструмент версионированных миграций БД. Миграции — SQL-файлы с именами `V1__description.sql`, `V2__description.sql`, ...

### Базовая конфигурация

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <!-- версия берётся из Spring Boot BOM -->
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: secret
  jpa:
    hibernate:
      ddl-auto: validate  # Hibernate только валидирует, Flyway управляет схемой
  flyway:
    enabled: true
    locations: classpath:db/migration  # папка с SQL-файлами
    baseline-on-migrate: false         # true только для существующих БД
```

### Структура миграций

```
src/main/resources/
└── db/
    └── migration/
        ├── V1__create_users_table.sql
        ├── V2__add_email_to_users.sql
        ├── V3__create_orders_table.sql
        └── V4__add_status_index.sql
```

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name  VARCHAR(100) NOT NULL,
    active     BOOLEAN      NOT NULL DEFAULT true,
    created_at TIMESTAMP    NOT NULL DEFAULT now()
);

-- V2__add_email_to_users.sql
ALTER TABLE users
    ADD COLUMN email VARCHAR(255) UNIQUE NOT NULL;

-- V3__create_orders_table.sql
CREATE TABLE orders (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT    NOT NULL REFERENCES users (id),
    total      NUMERIC(19, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_user_id ON orders (user_id);
```

### Порядок выполнения при старте

```
1. Spring Boot запускает DataSource
2. Flyway подключается и проверяет таблицу flyway_schema_history
3. Flyway применяет все непримённые миграции в порядке версий
4. Hibernate (ddl-auto=validate) проверяет схему — должна совпасть с entity
5. Приложение стартует
```

> [!INFO]
> Flyway применяет миграции **до** инициализации JPA. Это гарантирует, что к моменту валидации схема уже актуальна.

### Именование файлов

```
V{version}__{description}.sql    — версионная миграция (необратимая)
U{version}__{description}.sql    — undo-миграция (Flyway Teams)
R__{description}.sql             — repeatable (применяется при изменении)
```

> [!WARNING]
> После применения `V`-миграции **нельзя изменять** SQL-файл — Flyway проверяет чек-суммы. Для исправления создай новую миграцию.

---

## 5. Интеграция с Liquibase <a name="liquibase"></a>

Liquibase — альтернатива Flyway с поддержкой XML/YAML/JSON форматов changesets и rollback-операций.

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/v1-create-users.yaml
  - include:
      file: db/changelog/v2-add-email.yaml
```

**Когда выбрать Flyway, а когда Liquibase**:

| | Flyway | Liquibase |
|---|--------|-----------|
| Формат | Только SQL | SQL, XML, YAML, JSON |
| Rollback | Undo-файлы (платно) | Встроенный rollback |
| Простота | Выше | Ниже |
| Community | Больше | Меньше |
| Рекомендация | Spring-проекты | Нужен rollback |

---

**Связанные файлы:**
- [[First Level Cache vs Second Level Cache]] — конфигурация Hibernate
- [[Подключение зависимостей ehcache]] — конфигурация кэша
