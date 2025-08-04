# Properties и YAML конфигурация

Spring Framework предоставляет гибкие возможности для работы с конфигурационными файлами. В этом разделе мы рассмотрим работу с properties файлами и YAML форматом.

## Содержание

1. [Properties файлы](#properties-файлы)
2. [YAML конфигурация](#yaml-конфигурация)
3. [Приоритеты конфигурации](#приоритеты-конфигурации)
4. [Property Placeholders](#property-placeholders)
5. [Внедрение свойств](#внедрение-свойств)
6. [Expression Language (SpEL)](#expression-language-spel)
7. [Лучшие практики](#лучшие-практики)

## Properties файлы

### Основные properties файлы

Spring Boot автоматически загружает следующие файлы:

- `application.properties` - основной файл конфигурации
- `application-{profile}.properties` - профиль-специфичные файлы

### Пример application.properties

```properties
# Основные настройки приложения
app.name=MyApplication
app.version=1.0.0
app.description=${app.name} v${app.version}

# Настройки базы данных
db.driver=org.postgresql.Driver
db.url=jdbc:postgresql://localhost:5432/mydb
db.username=user
db.password=password
db.pool.size=10
db.pool.timeout=30

# Настройки логирования
logging.level.com.example=INFO
logging.level.org.springframework=WARN
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n

# Настройки сервера
server.port=8080
server.servlet.context-path=/api

# Профили
spring.profiles.active=dev
```

### Spring.properties

Файл `spring.properties` используется для конфигурации самого Spring Framework:

```properties
# Отключение автоконфигурации
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

# Настройки метаданных
spring.config.use-legacy-processing=true

# Настройки профилей
spring.profiles.default=default
```

## YAML конфигурация

YAML предоставляет более читаемый и структурированный формат конфигурации.

### Преимущества YAML

1. **Читаемость** - более понятная структура
2. **Иерархия** - четкое разделение уровней
3. **Переиспользование** - возможность создания шаблонов
4. **Комментарии** - поддержка комментариев

### Пример application.yml

```yaml
# Основные настройки приложения
app:
  name: MyApplication
  version: 1.0.0
  description: ${app.name} v${app.version}

# Настройки базы данных
db:
  driver: org.postgresql.Driver
  url: jdbc:postgresql://localhost:5432/mydb
  username: ${username.value:postgres}
  password: pass
  hosts: localhost,127.0.0.1
  
  # Объект конфигурации
  pool:
    size: 12
    timeout: 10
    max-connections: 50
    min-idle: 5
  
  # Массив объектов
  pools:
    - size: 1
      timeout: 1
      name: pool1
    - size: 2
      timeout: 2
      name: pool2
    - size: 3
      timeout: 3
      name: pool3
  
  # Свойства как Map
  properties:
    first: 123
    second: 567
    third.value: Third
    connection.timeout: 5000

# Настройки логирования
logging:
  level:
    com.example: INFO
    org.springframework: WARN
    org.hibernate: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"

# Настройки сервера
server:
  port: 8080
  servlet:
    context-path: /api
  compression:
    enabled: true
    mime-types: text/html,text/xml,text/plain,text/css,text/javascript,application/javascript,application/json

# Профили
spring:
  profiles:
    active: dev
  application:
    name: my-application
```

### Многоуровневая конфигурация

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
# Профиль для разработки
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 

---
# Профиль для продакшена
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-server:5432/mydb
    driver-class-name: org.postgresql.Driver
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

## Приоритеты конфигурации

Spring Boot загружает конфигурацию в следующем порядке (от низшего к высшему приоритету):

### 1. Default properties
```java
SpringApplication.setDefaultProperties(properties);
```

### 2. @PropertySource аннотации
```java
@Configuration
@PropertySource("classpath:custom.properties")
public class AppConfig {
}
```

### 3. Файлы конфигурации в JAR
- `application.properties`
- `application.yml`
- `application-{profile}.properties`
- `application-{profile}.yml`

### 4. RandomValuePropertySource
```properties
# Генерирует случайные значения
random.number=123
random.uuid=550e8400-e29b-41d4-a716-446655440000
```

### 5. Переменные окружения
```bash
export DB_USERNAME=myuser
export DB_PASSWORD=mypassword
```

### 6. Java System properties
```bash
java -Dapp.name=MyApp -jar application.jar
```

### 7. JNDI атрибуты
```xml
<jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/myDataSource"/>
```

### 8. ServletContext init parameters
```xml
<context-param>
    <param-name>app.name</param-name>
    <param-value>MyApplication</param-value>
</context-param>
```

### 9. ServletConfig init parameters
```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-config.xml</param-value>
    </init-param>
</servlet>
```

### 10. SPRING_APPLICATION_JSON
```bash
export SPRING_APPLICATION_JSON='{"server":{"port":8081},"logging":{"level":{"com.example":"DEBUG"}}}'
```

### 11. Аргументы командной строки
```bash
java -jar app.jar --server.port=8081 --logging.level.com.example=DEBUG
```

### 12. @TestPropertySource (для тестов)
```java
@SpringBootTest
@TestPropertySource(properties = {
    "server.port=0",
    "logging.level.com.example=DEBUG"
})
class MyTest {
}
```

### 13. Devtools global settings
```properties
# ~/.config/spring-boot/spring-boot-devtools.properties
spring.devtools.restart.enabled=false
```

## Property Placeholders

### Базовый синтаксис

```properties
# Определение значения
app.name=MyApplication
app.version=1.0.0

# Использование в других свойствах
app.description=${app.name} v${app.version}

# Значения по умолчанию
app.description=${app.name:Unknown} v${app.version:1.0}
```

### В YAML

```yaml
app:
  name: MyApplication
  version: 1.0.0
  description: ${app.name} v${app.version}
  fallback: ${unknown.property:Default Value}
```

### Переменные окружения

```properties
# Использование переменных окружения
db.username=${DB_USERNAME:defaultuser}
db.password=${DB_PASSWORD:defaultpass}
```

## Внедрение свойств

### В Java коде

```java
@Component
public class AppConfig {
    
    @Value("${app.name}")
    private String appName;
    
    @Value("${app.version:1.0.0}")
    private String appVersion;
    
    @Value("${db.pool.size:10}")
    private int poolSize;
    
    @Value("${logging.level.com.example:INFO}")
    private String logLevel;
}
```

### В XML конфигурации

```xml
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="${db.driver}"/>
    <property name="url" value="${db.url}"/>
    <property name="username" value="${db.username}"/>
    <property name="password" value="${db.password}"/>
</bean>
```

### Настройка PropertySourcesPlaceholderConfigurer

```xml
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
    <property name="locations" value="classpath:application.properties"/>
    <property name="ignoreUnresolvablePlaceholders" value="true"/>
    <property name="ignoreResourceNotFound" value="true"/>
</bean>
```

## Expression Language (SpEL)

SpEL позволяет использовать выражения в конфигурации.

### В XML

```xml
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="properties">
        <map>
            <entry key="url" value="postgresurl"/>
            <entry key="password" value="123"/>
            <entry key="driver" value="#{driver.substring(3)}"/>
            <entry key="test" value="#{driver.length() > 10}"/>
            <entry key="test1" value="#{driver.length() > T(Math).random() * 10}"/>
            <entry key="hosts" value="#{'${db.hosts}'.split(',')}"/>
            <entry key="currentUser" value="#{systemProperties['user.name']}"/>
            <entry key="randomValue" value="#{T(Math).random() * 100}"/>
        </map>
    </property>
</bean>
```

### В аннотациях

```java
@Component
public class UserService {
    
    @Value("#{systemProperties['user.name']}")
    private String currentUser;
    
    @Value("#{T(Math).random() * 100}")
    private double randomValue;
    
    @Value("#{'${db.hosts}'.split(',')}")
    private String[] hosts;
    
    @Value("#{@dataSource.getConnection().getMetaData().getDatabaseProductName()}")
    private String databaseName;
}
```

### Условные выражения

```java
@Component
public class ConditionalService {
    
    @Value("#{${feature.flag:false} ? 'enabled' : 'disabled'}")
    private String featureStatus;
    
    @Value("#{${pool.size:10} > 5 ? 'large' : 'small'}")
    private String poolType;
}
```

## Лучшие практики

### 1. Организация конфигурации

```yaml
# Разделение по модулям
app:
  name: MyApplication
  version: 1.0.0

database:
  primary:
    url: jdbc:postgresql://localhost:5432/primary
    username: primary_user
  secondary:
    url: jdbc:postgresql://localhost:5432/secondary
    username: secondary_user

security:
  jwt:
    secret: ${JWT_SECRET:default-secret}
    expiration: 86400000
```

### 2. Использование профилей

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

### 3. Валидация конфигурации

```java
@ConfigurationProperties("app")
@Validated
public class AppProperties {
    
    @NotBlank
    private String name;
    
    @Min(1)
    @Max(100)
    private int version;
    
    @Valid
    private DatabaseProperties database;
    
    // Геттеры и сеттеры
}
```

### 4. Безопасность

```yaml
# Не храните секреты в коде
security:
  jwt:
    secret: ${JWT_SECRET}
  oauth2:
    client-id: ${OAUTH2_CLIENT_ID}
    client-secret: ${OAUTH2_CLIENT_SECRET}
```

### 5. Документация

```yaml
# application.yml
# Конфигурация приложения
# Автор: Team
# Дата: 2024-01-01

app:
  # Название приложения
  name: MyApplication
  # Версия приложения
  version: 1.0.0
```

## Примеры использования

### Конфигурация базы данных

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:password}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### Конфигурация логирования

```yaml
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.springframework: WARN
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/application.log
    max-size: 10MB
    max-history: 30
```

### Конфигурация безопасности

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - email
              - profile
    jwt:
      secret: ${JWT_SECRET}
      expiration: 86400000
```

## Связанные темы

- [Spring Configuration](./Spring Configuration.md)
- [Аннотации Spring](../1.2 Аннотации Spring/README.md)
- [Spring Boot](../1.4 Spring Boot/README.md)
- [IoC и Dependency Injection](../1.1 IoC и Dependency Injection/README.md) 