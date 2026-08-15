# Пул соединений в JDBC

> [!QUOTE] Connection Pool — кэш готовых объектов `Connection`. Приложение берёт соединение из пула и возвращает обратно вместо закрытия. Это устраняет накладные расходы на TCP-handshake и аутентификацию при каждом запросе.

## Оглавление
1. [[#Зачем нужен пул]]
2. [[#Популярные пулы]]
3. [[#Настройка HikariCP]]
4. [[#Ключевые параметры]]
5. [[#Многопоточность]]
6. [[#Работа с транзакциями]]
7. [[#Мониторинг и диагностика]]
8. [[#Интеграция с Spring]]
9. [[#Best practices]]
10. [[#Вопросы для собеседования]]

---

## Зачем нужен пул

Создание нового `Connection` — дорогостоящая операция (TCP, SSL handshake, аутентификация, ~50-200 мс).

Без пула: каждый запрос = новое соединение = latency + нагрузка на СУБД.
С пулом: соединения переиспользуются, latency близка к нулю.

> [!WARNING] Прямой вызов `DriverManager.getConnection()` в production без пула — антипаттерн. При нагрузке СУБД получает шквал новых соединений и падает.

---

## Популярные пулы

- **HikariCP** — быстрый, лёгкий, минимальный overhead. **Рекомендуется**, дефолт в Spring Boot
- **Apache DBCP2** — стабильный, хорошо интегрируется с Tomcat
- **c3p0** — устаревший, но ещё встречается в legacy-проектах

---

## Настройка HikariCP

Maven-зависимость:

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

Программная конфигурация:

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
config.setUsername("app_user");
config.setPassword(System.getenv("DB_PASSWORD"));
config.setMaximumPoolSize(10);
config.setMinimumIdle(5);
config.setConnectionTimeout(30000);     // мс — ждать свободное соединение
config.setIdleTimeout(600000);          // мс — убрать простаивающее соединение
config.setMaxLifetime(1800000);         // мс — максимальное время жизни соединения
config.setConnectionTestQuery("SELECT 1");

HikariDataSource ds = new HikariDataSource(config);
```

Использование:

```java
try (Connection conn = ds.getConnection()) {
    // работа с БД
} // соединение автоматически возвращается в пул
```

---

## Ключевые параметры

| Параметр | Описание | Рекомендация |
|----------|----------|--------------|
| `maximumPoolSize` | Максимум соединений в пуле | CPU × 2 + количество дисков |
| `minimumIdle` | Минимум простаивающих | Равно `maximumPoolSize` для стабильной нагрузки |
| `connectionTimeout` | Ожидание свободного соединения | 20000–30000 мс |
| `idleTimeout` | Время жизни простаивающего | 600000 мс |
| `maxLifetime` | Максимальный возраст соединения | 1800000 мс (меньше, чем wait_timeout СУБД) |
| `connectionTestQuery` | Проверка живого соединения | `SELECT 1` |

> [!WARNING] `maxLifetime` должен быть **меньше** `wait_timeout` на стороне СУБД. Иначе СУБД закроет соединение раньше, чем пул узнает об этом — получите `Connection is closed` в runtime.

---

## Многопоточность

- Каждый поток берёт соединение из пула, использует и возвращает
- Не передавайте объект `Connection` между потоками — состояние соединения (autocommit, транзакция) не thread-safe

> [!WARNING] Передача одного `Connection` между несколькими потоками вызывает гонки состояний и непредсказуемое поведение транзакций.

---

## Работа с транзакциями

Транзакции работают в рамках одного `Connection`. При возврате соединения в пул HikariCP сбрасывает его состояние.

```java
try (Connection conn = ds.getConnection()) {
    conn.setAutoCommit(false);
    try {
        // SQL операции
        conn.commit();
    } catch (SQLException e) {
        conn.rollback();
        throw e;
    }
}
```

> [!INFO] HikariCP автоматически вызывает `rollback()` при возврате соединения с незавершённой транзакцией.

Подробнее о транзакциях: [[Транзакции в JDBC]]

---

## Мониторинг и диагностика

Типовые проблемы:
- **Исчерпание пула** — все соединения заняты, новые запросы ждут `connectionTimeout` и падают с исключением
- **Утечка соединений** — соединение взято из пула, но не возвращено (забыли `close()`)
- **Deadlock** — два потока ждут соединений, занятых друг другом

Диагностика:

```java
HikariPoolMXBean poolMXBean = ds.getHikariPoolMXBean();
System.out.println("Active: " + poolMXBean.getActiveConnections());
System.out.println("Idle: " + poolMXBean.getIdleConnections());
System.out.println("Total: " + poolMXBean.getTotalConnections());
System.out.println("Waiting: " + poolMXBean.getThreadsAwaitingConnection());
```

> [!INFO] HikariCP интегрируется с Micrometer/Prometheus — метрики пула доступны через `/actuator/metrics` в Spring Boot.

---

## Интеграция с Spring

```java
@Configuration
public class JdbcConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        ds.setUsername("root");
        ds.setPassword(System.getenv("DB_PASSWORD"));
        ds.setMaximumPoolSize(10);
        return ds;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

В Spring Boot достаточно указать параметры в `application.properties`:

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=${DB_PASSWORD}
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
```

---

## Best practices

- `try-with-resources` для автоматического возврата соединения в пул
- Настраивайте `maximumPoolSize` под реальную нагрузку
- `maxLifetime` < `wait_timeout` СУБД
- Включайте `connectionTestQuery` для проверки живости соединений
- Мониторьте метрики пула (активные, ожидающие, утечки)
- Не передавайте `Connection` между потоками

---

## Вопросы для собеседования

1. Зачем нужен пул соединений?
2. Как работает HikariCP внутри?
3. Что произойдёт, если пул исчерпан?
4. Как диагностировать утечку соединений?
5. Как пул работает с транзакциями?
6. Какие параметры настраивать в первую очередь?

---

**Связанные заметки:** [[Основы]] | [[Транзакции в JDBC]] | [[Безопасность в JDBC]] | [[Ошибки и диагностика в JDBC]]

**Материалы:**
- [HikariCP Documentation](https://github.com/brettwooldridge/HikariCP)
- [HikariCP Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Spring JdbcTemplate](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#jdbc)
