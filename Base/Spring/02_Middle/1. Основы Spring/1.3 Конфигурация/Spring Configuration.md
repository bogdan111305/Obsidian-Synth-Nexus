# Spring Configuration

Spring Framework предоставляет несколько способов конфигурации приложения. В этом разделе мы рассмотрим основные подходы к настройке Spring приложений.

## Содержание

1. [Аннотационная конфигурация](#аннотационная-конфигурация)
2. [XML конфигурация](#xml-конфигурация)
3. [Комбинированный подход](#комбинированный-подход)
4. [@ConfigurationProperties](#configurationproperties)
5. [Property Placeholders](#property-placeholders)
6. [Expression Language (SpEL)](#expression-language-spel)

## Аннотационная конфигурация

Аннотационная конфигурация является современным и рекомендуемым подходом в Spring Framework.

### Основные аннотации конфигурации

```java
@Configuration
@ComponentScan("com.example")
@PropertySource("classpath:application.properties")
public class AppConfig {
    
    @Bean
    public UserService userService() {
        return new UserService();
    }
    
    @Bean
    @Primary
    public DataSource dataSource() {
        return new DriverManagerDataSource();
    }
}
```

### @Configuration

Основная аннотация для обозначения класса как конфигурационного:

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("org.postgresql.Driver");
        dataSource.setUrl("jdbc:postgresql://localhost:5432/mydb");
        dataSource.setUsername("user");
        dataSource.setPassword("password");
        return dataSource;
    }
}
```

### @ComponentScan

Автоматическое сканирование и регистрация компонентов:

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {
    // Конфигурация
}
```

## XML конфигурация

Хотя XML конфигурация считается устаревшей, она все еще поддерживается:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context
                           http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- Включение аннотаций -->
    <context:annotation-config/>
    
    <!-- Сканирование компонентов -->
    <context:component-scan base-package="com.example"/>
    
    <!-- Bean определения -->
    <bean id="userService" class="com.example.service.UserService"/>
    
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="org.postgresql.Driver"/>
        <property name="url" value="jdbc:postgresql://localhost:5432/mydb"/>
        <property name="username" value="user"/>
        <property name="password" value="password"/>
    </bean>
</beans>
```

## Комбинированный подход

Можно комбинировать XML и аннотации:

### Включение аннотаций в XML

```xml
<!-- Включение обработки аннотаций -->
<context:annotation-config/>

<!-- Или более специфично -->
<bean class="org.springframework.context.annotation.CommonAnnotationBeanPostProcessor"/>
```

### Использование аннотаций в XML конфигурации

```java
@Component
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Value("${app.name}")
    private String appName;
    
    @PostConstruct
    public void init() {
        System.out.println("UserService initialized");
    }
    
    @PreDestroy
    public void cleanup() {
        System.out.println("UserService destroyed");
    }
}
```

## @ConfigurationProperties

`@ConfigurationProperties` позволяет связывать свойства из конфигурационных файлов с Java объектами.

### Создание POJO класса

```java
@Data
@NoArgsConstructor
public class DatabaseProperties {
    
    private String username;
    private String password;
    private String driver;
    private String hosts;
    private Map<String, Object> properties;
    private PoolProperties pool;
    private List<PoolProperties> pools;
    
    @Data
    @NoArgsConstructor
    public static class PoolProperties {
        private Integer size;
        private Integer timeout;
    }
}
```

### Способы связывания с конфигурацией

#### 1. Через @Bean и @ConfigurationProperties

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @ConfigurationProperties("db")
    public DatabaseProperties databaseProperties() {
        return new DatabaseProperties();
    }
}
```

#### 2. @ConfigurationProperties над классом

```java
@Data
@NoArgsConstructor
@ConfigurationProperties("db")
@Component
public class DatabaseProperties {
    // Поля класса
}
```

#### 3. С @ConfigurationPropertiesScan

```java
@Configuration
@ConfigurationPropertiesScan
public class AppConfig {
    // Конфигурация
}

@ConfigurationProperties("db")
public class DatabaseProperties {
    // Поля класса
}
```

#### 4. Immutable конфигурация с @ConstructorBinding

```java
@ConstructorBinding
@ConfigurationProperties(prefix = "db")
public record DatabaseProperties(
    String username,
    String password,
    String driver,
    String hosts,
    Map<String, Object> properties,
    PoolProperties pool,
    List<PoolProperties> pools
) {
    public static record PoolProperties(Integer size, Integer timeout) {}
}
```

### Пример YAML конфигурации

```yaml
db:
  username: ${username.value:postgres}
  password: pass
  driver: PostgresDriver
  url: postgres:5432
  hosts: localhost,127.0.0.1
  properties:
    first: 123
    second: 567
    third.value: Third
  pool:
    size: 12
    timeout: 10
  pools:
    - size: 1
      timeout: 1
    - size: 2
      timeout: 2
    - size: 3
      timeout: 3
```

## Property Placeholders

### Использование ${...} синтаксиса

```properties
# application.properties
app.name=MyApp
app.description=${app.name}
app.description=${username:Unknown}
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
</bean>
```

## Expression Language (SpEL)

SpEL позволяет использовать выражения в конфигурации:

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
}
```

## Лучшие практики

1. **Используйте аннотационную конфигурацию** для новых проектов
2. **Разделяйте конфигурацию** по функциональным модулям
3. **Используйте профили** для разных окружений
4. **Применяйте @ConfigurationProperties** для сложных конфигураций
5. **Документируйте конфигурацию** с помощью комментариев
6. **Валидируйте конфигурацию** с помощью @Validated

## Пример полной конфигурации

```java
@Configuration
@ComponentScan("com.example")
@PropertySource("classpath:application.properties")
@EnableConfigurationProperties(DatabaseProperties.class)
public class AppConfig {
    
    @Bean
    @ConfigurationProperties("app")
    public AppProperties appProperties() {
        return new AppProperties();
    }
    
    @Bean
    public DataSource dataSource(DatabaseProperties dbProps) {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName(dbProps.getDriver());
        dataSource.setUrl(dbProps.getUrl());
        dataSource.setUsername(dbProps.getUsername());
        dataSource.setPassword(dbProps.getPassword());
        return dataSource;
    }
}
```

## Связанные темы

- [Properties и YAML конфигурация](./Properties and YAML.md)
- [Аннотации Spring](../1.2 Аннотации Spring/README.md)
- [Spring Boot](../1.4 Spring Boot/README.md)
- [IoC и Dependency Injection](../1.1 IoC и Dependency Injection/README.md) 