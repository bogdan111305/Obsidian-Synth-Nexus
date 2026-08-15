# Класс JpaRepositoryAutoConfiguration

## Обзор

`JpaRepositoryAutoConfiguration` - это класс автоматической конфигурации Spring Boot, который настраивает JPA репозитории без необходимости ручной конфигурации.

## Основные возможности

### Автоматическая настройка
- Автоматическое сканирование репозиториев
- Настройка EntityManagerFactory
- Конфигурация TransactionManager
- Регистрация JpaRepositoryFactoryBean

### Структура класса

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(EntityManagerFactory.class)
@ConditionalOnClass({ JpaRepository.class })
@ConditionalOnMissingBean({ JpaRepositoryFactoryBean.class, 
                           JpaRepositoryConfigExtension.class })
@EnableJpaRepositories
public class JpaRepositoryAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public JpaRepositoryConfigExtension jpaRepositoryConfigExtension() {
        return new JpaRepositoryConfigExtension();
    }
}
```

## Условия активации

### @ConditionalOnBean(EntityManagerFactory.class)
Конфигурация активируется только при наличии настроенного EntityManagerFactory.

### @ConditionalOnClass({ JpaRepository.class })
Требует наличие класса JpaRepository в classpath.

### @ConditionalOnMissingBean
Активируется только если не определены собственные бины для JPA репозиториев.

## Настройка через свойства

```yaml
spring:
  data:
    jpa:
      repositories:
        enabled: true
        bootstrap-mode: default
        repository-base-package: com.example.repository
```

## Свойства конфигурации

| Свойство | Описание | По умолчанию |
|----------|----------|--------------|
| `spring.data.jpa.repositories.enabled` | Включение JPA репозиториев | `true` |
| `spring.data.jpa.repositories.bootstrap-mode` | Режим загрузки | `default` |
| `spring.data.jpa.repositories.repository-base-package` | Базовый пакет для сканирования | Автоопределение |

## Режимы bootstrap-mode

### DEFAULT
Стандартный режим - репозитории создаются при старте приложения.

### LAZY
Ленивая инициализация - репозитории создаются при первом обращении.

### DEFERRED
Отложенная инициализация - репозитории создаются после завершения контекста.

## Кастомизация конфигурации

### Отключение автоконфигурации

```java
@SpringBootApplication(exclude = {
    JpaRepositoryAutoConfiguration.class
})
public class Application {
    // ручная конфигурация
}
```

### Переопределение настроек

```java
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.custom.repository",
    entityManagerFactoryRef = "customEntityManagerFactory",
    transactionManagerRef = "customTransactionManager"
)
public class CustomJpaConfig {
    // кастомная конфигурация
}
```

## Интеграция с другими компонентами

### EntityManagerFactory
```java
@Bean
public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
    LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
    factory.setDataSource(dataSource());
    factory.setPackagesToScan("com.example.entity");
    factory.setJpaVendorAdapter(jpaVendorAdapter());
    return factory;
}
```

### TransactionManager
```java
@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {
    return new JpaTransactionManager(entityManagerFactory);
}
```

## Логирование и отладка

### Включение логирования
```properties
logging.level.org.springframework.data.jpa=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### Проверка конфигурации
```java
@EventListener(ApplicationReadyEvent.class)
public void checkJpaConfiguration() {
    System.out.println("JPA Repositories configured successfully");
}
```

## Лучшие практики

1. **Используйте автоконфигурацию** для стандартных случаев
2. **Настраивайте свойства** через application.yml
3. **Проверяйте условия активации** при проблемах
4. **Используйте кастомную конфигурацию** для сложных случаев
5. **Мониторьте логи** для отладки проблем 