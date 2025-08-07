# Класс TransactionAutoConfiguration

## Обзор

`TransactionAutoConfiguration` - это ключевой класс Spring Boot, который автоматически настраивает инфраструктуру транзакций в приложении. Он является частью Spring Boot Auto-Configuration и обеспечивает готовую к использованию конфигурацию транзакций.

## Основная функциональность

### Автоматическая конфигурация

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(PlatformTransactionManager.class)
@EnableConfigurationProperties(TransactionProperties.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, HibernateJpaAutoConfiguration.class })
public class TransactionAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
    
    @Bean
    @ConditionalOnMissingBean
    public TransactionTemplate transactionTemplate(PlatformTransactionManager transactionManager) {
        return new TransactionTemplate(transactionManager);
    }
}
```

## Структура класса

### Аннотации конфигурации

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(PlatformTransactionManager.class)
@EnableConfigurationProperties(TransactionProperties.class)
@AutoConfigureAfter({ 
    DataSourceAutoConfiguration.class, 
    HibernateJpaAutoConfiguration.class 
})
public class TransactionAutoConfiguration {
    // Реализация
}
```

### Условная конфигурация

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(PlatformTransactionManager.class) // Только если класс доступен
public class TransactionAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(name = "transactionManager") // Только если нет существующего bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

## Автоматически создаваемые компоненты

### 1. PlatformTransactionManager

```java
@Bean
@ConditionalOnMissingBean
public PlatformTransactionManager transactionManager(DataSource dataSource) {
    DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
    transactionManager.setDataSource(dataSource);
    return transactionManager;
}
```

### 2. TransactionTemplate

```java
@Bean
@ConditionalOnMissingBean
public TransactionTemplate transactionTemplate(PlatformTransactionManager transactionManager) {
    TransactionTemplate template = new TransactionTemplate(transactionManager);
    template.setIsolationLevel(Isolation.DEFAULT);
    template.setPropagationBehavior(Propagation.REQUIRED);
    template.setTimeout(TransactionTemplate.TIMEOUT_DEFAULT);
    return template;
}
```

### 3. TransactionProperties

```java
@ConfigurationProperties(prefix = "spring.transaction")
public class TransactionProperties {
    
    private Duration defaultTimeout = Duration.ofSeconds(30);
    private boolean rollbackOnCommitFailure = false;
    
    // Геттеры и сеттеры
}
```

## Интеграция с различными технологиями

### JPA/Hibernate

```java
@Configuration
@ConditionalOnClass({ LocalContainerEntityManagerFactoryBean.class, EntityManager.class })
public class JpaTransactionAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(name = "transactionManager")
    public PlatformTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(entityManagerFactory);
        return transactionManager;
    }
}
```

### JDBC

```java
@Configuration
@ConditionalOnClass({ DataSource.class, JdbcTemplate.class })
public class DataSourceTransactionAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(name = "transactionManager")
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

### Hibernate

```java
@Configuration
@ConditionalOnClass({ SessionFactory.class, HibernateTransactionManager.class })
public class HibernateTransactionAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(name = "transactionManager")
    public PlatformTransactionManager transactionManager(SessionFactory sessionFactory) {
        HibernateTransactionManager transactionManager = new HibernateTransactionManager();
        transactionManager.setSessionFactory(sessionFactory);
        return transactionManager;
    }
}
```

## Настройка через properties

### application.properties

```properties
# Настройки транзакций
spring.transaction.default-timeout=30s
spring.transaction.rollback-on-commit-failure=false

# Настройки для JPA
spring.jpa.properties.hibernate.connection.provider_disables_autocommit=true
spring.jpa.properties.hibernate.current_session_context_class=org.springframework.orm.hibernate5.SpringSessionContext
```

### application.yml

```yaml
spring:
  transaction:
    default-timeout: 30s
    rollback-on-commit-failure: false
  jpa:
    properties:
      hibernate:
        connection:
          provider_disables_autocommit: true
        current_session_context_class: org.springframework.orm.hibernate5.SpringSessionContext
```

## Кастомизация конфигурации

### Переопределение TransactionManager

```java
@Configuration
public class CustomTransactionConfig {
    
    @Bean
    @Primary
    public PlatformTransactionManager customTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager(dataSource);
        tm.setDefaultTimeout(60); // 60 секунд
        tm.setRollbackOnCommitFailure(true);
        return tm;
    }
}
```

### Множественные TransactionManager

```java
@Configuration
public class MultipleTransactionConfig {
    
    @Bean("jdbcTransactionManager")
    public PlatformTransactionManager jdbcTransactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
    
    @Bean("jpaTransactionManager")
    public PlatformTransactionManager jpaTransactionManager(EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}
```

## Интеграция с @EnableTransactionManagement

### Автоматическое включение

```java
@Configuration
@EnableTransactionManagement // Включается автоматически в Spring Boot
public class TransactionConfig {
    // Дополнительная конфигурация
}
```

### Ручное включение

```java
@Configuration
@EnableTransactionManagement(
    proxyTargetClass = true,
    mode = AdviceMode.PROXY,
    order = Ordered.LOWEST_PRECEDENCE
)
public class ManualTransactionConfig {
    // Конфигурация
}
```

## Обработка исключений

### Автоматический откат

```java
@Configuration
public class TransactionExceptionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager(dataSource);
        tm.setRollbackOnCommitFailure(true);
        return tm;
    }
}
```

### Кастомная обработка исключений

```java
@Configuration
public class CustomTransactionExceptionConfig {
    
    @Bean
    public TransactionInterceptor transactionInterceptor(PlatformTransactionManager tm) {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionManager(tm);
        
        // Настройка правил отката
        Properties attributes = new Properties();
        attributes.setProperty("*", "PROPAGATION_REQUIRED,ISOLATION_READ_COMMITTED,-Exception");
        interceptor.setTransactionAttributes(attributes);
        
        return interceptor;
    }
}
```

## Мониторинг и отладка

### Включение логирования

```properties
# application.properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.orm.jpa=DEBUG
logging.level.org.springframework.jdbc=DEBUG
```

### Проверка конфигурации

```java
@Component
public class TransactionConfigChecker {
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkTransactionConfiguration(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        // Проверяем наличие TransactionManager
        Map<String, PlatformTransactionManager> managers = 
            context.getBeansOfType(PlatformTransactionManager.class);
        
        System.out.println("Found TransactionManagers: " + managers.keySet());
        
        // Проверяем настройки
        TransactionProperties props = context.getBean(TransactionProperties.class);
        System.out.println("Default timeout: " + props.getDefaultTimeout());
    }
}
```

## Лучшие практики

### 1. Использование профилей

```java
@Configuration
@Profile("development")
public class DevelopmentTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager(dataSource);
        tm.setDefaultTimeout(10); // Короткий таймаут для разработки
        return tm;
    }
}

@Configuration
@Profile("production")
public class ProductionTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager(dataSource);
        tm.setDefaultTimeout(60); // Длинный таймаут для продакшена
        return tm;
    }
}
```

### 2. Конфигурация для тестов

```java
@TestConfiguration
public class TestTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(embeddedDataSource());
    }
    
    @Bean
    public DataSource embeddedDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}
```

### 3. Оптимизация производительности

```java
@Configuration
public class OptimizedTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager(dataSource);
        
        // Оптимизации
        tm.setDefaultTimeout(30);
        tm.setRollbackOnCommitFailure(false);
        tm.setValidateExistingTransaction(true);
        
        return tm;
    }
}
```

## Интеграция с другими компонентами

### Spring Data JPA

```java
@Configuration
@EnableJpaRepositories
@EnableTransactionManagement
public class JpaTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
        JpaTransactionManager tm = new JpaTransactionManager();
        tm.setEntityManagerFactory(emf);
        return tm;
    }
}
```

### Spring Security

```java
@Configuration
@EnableTransactionManagement
public class SecurityTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager(dataSource);
        
        // Интеграция с Spring Security
        tm.setTransactionSynchronization(AbstractPlatformTransactionManager.SYNCHRONIZATION_ALWAYS);
        
        return tm;
    }
}
``` 