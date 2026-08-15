# Spring Boot

Spring Boot - это фреймворк для быстрого создания Spring приложений с минимальной конфигурацией. Он предоставляет автоконфигурацию, встроенные серверы и готовые к использованию стартеры.

## Содержание

1. [Введение в Spring Boot](#введение-в-spring-boot)
2. [@SpringBootApplication](#springbootapplication)
3. [Автоконфигурация](#автоконфигурация)
4. [Spring Boot Starters](#spring-boot-starters)
5. [Конфигурация логирования](#конфигурация-логирования)
6. [Сборка проекта](#сборка-проекта)
7. [Лучшие практики](#лучшие-практики)

## Введение в Spring Boot

Spring Boot упрощает разработку Spring приложений, предоставляя:

- **Автоконфигурацию** - автоматическая настройка на основе classpath
- **Встроенные серверы** - Tomcat, Jetty, Undertow
- **Starter зависимости** - готовые наборы зависимостей
- **Actuator** - мониторинг и управление приложением
- **DevTools** - инструменты для разработки

### Основные преимущества

```java
// Минимальная конфигурация
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

## @SpringBootApplication

`@SpringBootApplication` - это составная аннотация, которая включает в себя:

- `@Configuration` - обозначает класс как источник определений бинов
- `@EnableAutoConfiguration` - включает автоконфигурацию
- `@ComponentScan` - включает сканирование компонентов

### Структура main класса

```java
@SpringBootApplication
public class ApplicationRunner {
    
    public static void main(String[] args) {
        ConfigurableApplicationContext context = 
                SpringApplication.run(ApplicationRunner.class, args);
        
        // Получение бинов из контекста
        UserService userService = context.getBean(UserService.class);
        
        // Закрытие контекста
        context.close();
    }
}
```

### Автоматическая загрузка конфигурации

Spring Boot автоматически загружает следующие файлы:

- `application.properties`
- `application.yml`
- `application-{profile}.properties`
- `application-{profile}.yml`

### Размещение main класса

Main класс должен находиться в корневом пакете приложения:

```java
// Правильно
package com.example.myapp;

@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}

// Неправильно
package com.example.myapp.config;

@SpringBootApplication
public class MyApplication {
    // ...
}
```

## Автоконфигурация

Spring Boot использует условную конфигурацию для автоматического подключения модулей.

### Условия автоконфигурации

```java
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.enabled", havingValue = "true", matchIfMissing = true)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        // Автоматическая конфигурация DataSource
        return new DriverManagerDataSource();
    }
}
```

### Типы условий

#### 1. Classpath условия

```java
@ConditionalOnClass(PostgreSQLDriver.class)
public class PostgresAutoConfiguration {
    // Конфигурация только если PostgreSQL драйвер в classpath
}
```

#### 2. Property условия

```java
@ConditionalOnProperty(name = "app.feature.enabled", havingValue = "true")
public class FeatureAutoConfiguration {
    // Конфигурация только если свойство установлено
}
```

#### 3. Bean условия

```java
@ConditionalOnMissingBean(DataSource.class)
public DataSource dataSource() {
    // Создать бин только если он еще не существует
}
```

#### 4. SpEL условия

```java
@ConditionalOnExpression("${app.debug:false}")
public class DebugAutoConfiguration {
    // Конфигурация на основе выражения
}
```

### Отключение автоконфигурации

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class MyApplication {
    // ...
}
```

## Spring Boot Starters

Starters - это зависимости, которые добавляют функциональность в проект.

### Основные starters

```xml
<!-- Web приложение -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- База данных -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Безопасность -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Тестирование -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### Создание custom starter

```java
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties);
    }
}
```

## Конфигурация логирования

Spring Boot использует SLF4J API с Logback по умолчанию.

### Стартер для логирования

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

### Использование в коде

```java
@Slf4j
public class UserService {
    
    public void createUser(User user) {
        log.info("Creating user: {}", user.getUsername());
        
        try {
            // Логика создания пользователя
            log.debug("User created successfully");
        } catch (Exception e) {
            log.error("Failed to create user: {}", user.getUsername(), e);
            throw e;
        }
    }
}
```

### Конфигурация в YAML

```yaml
logging:
  # Кодировка консоли
  charset:
    console: utf-8
  
  # Уровни логирования
  level:
    root: WARN
    com.example: DEBUG
    org.springframework: INFO
    org.hibernate.SQL: DEBUG
  
  # Паттерны форматирования
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  
  # Файловое логирование
  file:
    name: logs/application.log
    max-size: 10MB
    max-history: 30
```

### Custom конфигурация Logback

Создайте файл `logback-spring.xml` в `src/main/resources`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <springProfile name="dev">
        <logger name="com.example" level="DEBUG"/>
    </springProfile>
    
    <springProfile name="prod">
        <logger name="com.example" level="WARN"/>
    </springProfile>
    
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

## Сборка проекта

### Maven

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

### Gradle

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### Создание исполняемого JAR

```bash
# Maven
mvn clean package

# Gradle
./gradlew bootJar
```

## Лучшие практики

### 1. Структура проекта

```
src/
├── main/
│   ├── java/
│   │   └── com/example/myapp/
│   │       ├── MyApplication.java
│   │       ├── controller/
│   │       ├── service/
│   │       ├── repository/
│   │       └── config/
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       └── application-prod.yml
└── test/
    └── java/
        └── com/example/myapp/
```

### 2. Конфигурация по профилям

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-server:5432/mydb
```

### 3. Использование @ConfigurationProperties

```java
@ConfigurationProperties("app")
@Data
public class AppProperties {
    private String name;
    private String version;
    private Database database;
    
    @Data
    public static class Database {
        private String url;
        private String username;
        private String password;
    }
}
```

### 4. Кастомная автоконфигурация

```java
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties);
    }
}
```

### 5. Тестирование

```java
@SpringBootTest
@ActiveProfiles("test")
class MyApplicationTests {
    
    @Autowired
    private UserService userService;
    
    @Test
    void contextLoads() {
        assertThat(userService).isNotNull();
    }
}
```

## Примеры использования

### Простое REST API

```java
@SpringBootApplication
public class RestApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(RestApiApplication.class, args);
    }
}

@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping
    public List<User> getUsers() {
        return userService.findAll();
    }
    
    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

### Конфигурация с профилями

```java
@Configuration
@Profile("dev")
public class DevConfig {
    
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                .url("jdbc:postgresql://prod-server:5432/mydb")
                .username("prod_user")
                .password("prod_pass")
                .build();
    }
}
```

## Связанные темы

- [Spring Configuration](../1.3 Конфигурация/README.md)
- [Аннотации Spring](../1.2 Аннотации Spring/README.md)
- [IoC и Dependency Injection](../1.1 IoC и Dependency Injection/README.md)
- [Spring Boot Actuator](../2. Продвинутые темы Spring/README.md)
- [Spring Boot DevTools](../2. Продвинутые темы Spring/README.md)