## 1. Что такое Транзакции?

Транзакция — это последовательность операций с базой данных, которая рассматривается как единое действие. Если хотя бы одна операция завершается с ошибкой, все изменения откатываются.

### Принципы ACID

- **Atomicity (Атомарность)**: Все операции выполняются целиком или не выполняются вообще.
- **Consistency (Согласованность)**: Данные остаются в валидном состоянии после транзакции.
- **Isolation (Изолированность)**: Одни транзакции не влияют на другие, выполняющиеся параллельно.
- **Durability (Долговечность)**: Изменения сохраняются даже при сбое системы после подтверждения.

### Пример в Контексте

В вашей социальной сети, если пользователь обновляет профиль и одновременно создает заказ, обе операции должны либо успешно завершиться, либо откатиться, если одна из них провалится.

## 2. Управление Транзакциями в JDBC

JDBC предоставляет низкоуровневый контроль над транзакциями через объект `Connection`. По умолчанию транзакции автокоммиты (автоматически подтверждаются после каждой операции), но их можно отключить для ручного управления.

### 2.1. Базовая Настройка

- **Отключение Автокоммита**:

```java
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb", "root", "password");
conn.setAutoCommit(false); // Отключаем автокоммит
```

- **Подтверждение Транзакции**:

```java
conn.commit(); // Подтверждаем изменения
```

- **Откат Транзакции**:

```java
conn.rollback(); // Откатываем изменения при ошибке
```

- **Закрытие**:

```java
conn.close(); // Освобождаем ресурсы
```


### 2.2. Полный Пример

```java
import java.sql.*;

public class JdbcTransactionExample {
    public static void main(String[] args) {
        String url = "jdbc:mysql://localhost:3306/mydb";
        String user = "root";
        String password = "password";

        Connection conn = null;
        try {
            // Устанавливаем соединение
            conn = DriverManager.getConnection(url, user, password);
            conn.setAutoCommit(false); // Отключаем автокоммит

            // Выполняем операции
            Statement stmt = conn.createStatement();
            stmt.executeUpdate("INSERT INTO users (name) VALUES ('John')");
            stmt.executeUpdate("INSERT INTO orders (user_id) VALUES (1)");

            // Подтверждаем транзакцию
            conn.commit();
            System.out.println("Транзакция успешно завершена.");
        } catch (SQLException e) {
            System.out.println("Ошибка: " + e.getMessage());
            try {
                if (conn != null) conn.rollback(); // Откат при ошибке
            } catch (SQLException ex) {
                ex.printStackTrace();
            }
        } finally {
            try {
                if (conn != null) conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
}
```

- **Объяснение**: Если одна из операций (например, `INSERT INTO orders`) провалится, `rollback` отменит все изменения, включая `INSERT INTO users`.

## 3. Уровни Изоляции Транзакций

JDBC поддерживает четыре уровня изоляции, которые определяют, как транзакции взаимодействуют друг с другом:

- **READ_UNCOMMITTED**: Позволяет читать не подтвержденные изменения (грязное чтение).
- **READ_COMMITTED**: Запрещает грязное чтение, но допускает фантомные чтения.
- **REPEATABLE_READ**: Запрещает фантомные чтения, но возможны изменения другими транзакциями.
- **SERIALIZABLE**: Полная изоляция, самая строгая, но медленная.

### Настройка

```java
conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
```

- **Рекомендация**: Используйте `READ_COMMITTED` как баланс между производительностью и согласованностью.

## 4. Расширенное Управление Транзакциями

### 4.1. Сохраненные Точки (Savepoints)

- Позволяют откатить только часть транзакции.

```java
Savepoint savepoint = conn.setSavepoint("savepoint1");
stmt.executeUpdate("INSERT INTO users (name) VALUES ('Jane')");
conn.rollback(savepoint); // Откат только до savepoint1
conn.commit();
```

### 4.2. Вложенные Транзакции

- JDBC не поддерживает вложенные транзакции напрямую, но их можно эмулировать с savepoints.

## 5. Интеграция с Spring

Spring упрощает управление транзакциями через аннотацию `@Transactional` и `JdbcTemplate`.

### 5.1. Настройка

```java
@Configuration
public class JdbcConfig {
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setUrl("jdbc:mysql://localhost:3306/mydb");
        dataSource.setUsername("root");
        dataSource.setPassword("password");
        return dataSource;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### 5.2. Использование @Transactional

```java
@Service
public class UserService {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Transactional
    public void updateProfileAndOrder(Long userId, String name) {
        jdbcTemplate.update(
	        "UPDATE users SET name = ? WHERE id = ?", name, userId
		);
        jdbcTemplate.update("INSERT INTO orders (user_id) VALUES (?)", userId);
        // Если ошибка, Spring автоматически откатит транзакцию
    }
}
```

- **Преимущество**: Автоматический rollback при исключениях.

### 5.3. Настройка Уровня Изоляции

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void updateProfileAndOrder(Long userId, String name) {
    // Логика
}
```

## 6. Как Работают Транзакции в JDBC?

### 6.1. Механизм

- **Начало**: При вызове `setAutoCommit(false)` JDBC начинает новую транзакцию.
- **Операции**: Все SQL-команды группируются в транзакцию до вызова `commit` или `rollback`.
- **Подтверждение/Откат**: База данных фиксирует изменения или откатывает их на основе команды.
- **Изоляция**: Уровень изоляции определяет, как транзакции видят изменения друг друга.

### 6.2. Взаимодействие с Базой

- Драйвер JDBC (например, MySQL Connector/J) преобразует команды в протокол базы данных (например, MySQL Protocol).
- Транзакции управляются на уровне базы данных (например, InnoDB в MySQL).

## 7. Лучшие Практики

### 7.1. Управление Ресурсами

- Используйте try-with-resources для автоматического закрытия соединений:

```java
try (Connection conn = DriverManager.getConnection(url, user, password)) {
	conn.setAutoCommit(false);
	// Логика
	conn.commit();
} catch (SQLException e) {
	e.printStackTrace();
}
```


### 7.2. Производительность

- Минимизируйте использование транзакций для коротких операций.
- Используйте пул соединений (например, HikariCP) для многопоточных приложений.

### 7.3. Обработка Ошибок

- Всегда предусматривайте `rollback` в блоке `catch`:

```java
try {
	conn.setAutoCommit(false);
	// Операции
	conn.commit();
} catch (SQLException e) {
	if (conn != null) conn.rollback();
	throw e;
}
```


### 7.4. Тестирование

- Имитируйте ошибки (например, отключение сети) для проверки rollback.

## 8. Ограничения и Решения

### Ограничения

- **Ручное Управление**: Требует явного вызова `commit` и `rollback`.
- **Производительность**: Длительные транзакции могут блокировать ресурсы.
- **Сложность**: Низкоуровневое управление усложняет код.

### Решения

- Используйте Spring `@Transactional` для автоматизации.
- Оптимизируйте длительность транзакций, разделяя операции.

## 9. Применение в Микросервисах

### 9.1. Пример в `User Management Service`

```java
@Service
public class UserService {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Transactional
    public void registerUser(String name, String email) {
        jdbcTemplate.update(
	        "INSERT INTO users (name, email) VALUES (?, ?)", name, email
        );
        jdbcTemplate.update(
	        "INSERT INTO user_audit (user_id, action) " +
				"VALUES (LAST_INSERT_ID(), 'REGISTER')"
		);
    }
}
```

- **Объяснение**: Регистрация пользователя и аудит происходят в одной транзакции.

### 9.2. Распределенные Транзакции

- Для микросервисов используйте XA-транзакции или Saga-паттерн:
    
```java
import javax.transaction.xa.XAResource;

XAResource xaResource = conn.getXAResource();
// Координация через XA
```