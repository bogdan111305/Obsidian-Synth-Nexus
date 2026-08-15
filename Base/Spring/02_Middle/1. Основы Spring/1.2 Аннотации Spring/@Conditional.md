# @Conditional

**Теги:** #spring #springboot #dmdev

## Обзор

Аннотация `@Conditional` позволяет условно регистрировать бины в Spring контексте. Это мощный механизм для создания конфигураций, которые активируются только при определенных условиях.

## Определение аннотации

```java
@Target({ElementType.TYPE, ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Conditional {  
  
    Class<? extends Condition>[] value();  
  
}
```

### Применение

Аннотацию можно ставить над:
- **Типом (классом)** - для условной регистрации компонентов
- **Методом** - для условной регистрации бинов (методы с `@Bean`)

> **Важно:** Под методом имеется в виду метод, помеченный аннотацией `@Bean`

<i style="color:yellow;">То есть с помощью этой аннотации мы можем определить, добавлять бин в контекст или нет</i>

## Condition интерфейс

`Condition` - это функциональный интерфейс, единственный метод которого возвращает `true` или `false`.

```java
@FunctionalInterface  
public interface Condition {  
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);  
}
```

### Параметры метода matches

#### ConditionContext
Контекст, с помощью которого можно получить доступ к:

- **BeanDefinitionRegistry** - доступ ко всем BeanDefinitions
- **BeanFactory** - любые готовые бины
- **Environment** - окружение приложения
- **ResourceLoader** - любой файл в classPath
- **ClassLoader** - загрузчик классов

#### AnnotatedTypeMetadata
Метаинформация об аннотациях, которые стоят над классом или методом.

## Примеры использования

### 1. Проверка наличия драйвера PostgreSQL

```java
@Conditional(JpaCondition.class)  
@Configuration  
public class JpaConfiguration {  
    @Bean
    public DataSource dataSource() {
        // Конфигурация DataSource
        return new DriverManagerDataSource();
    }
}
```

Реализация условия:

```java
public class JpaCondition implements Condition {  
    @Override  
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {  
        ClassLoader classLoader = context.getClassLoader();  
        try {  
            classLoader.loadClass("org.postgresql.Driver");  
            return true;        
        } catch (ClassNotFoundException e) {  
            return false;  
        }  
    }  
}
```

### 2. Проверка профиля

```java
public class OnProdCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return context.getEnvironment().acceptsProfiles(Profiles.of("prod"));
    }
}

@Service
@Conditional(OnProdCondition.class)
public class ProdOrderService implements OrderService {
    // Реализация для продакшена
}
```

### 3. Проверка свойств

```java
public class OnPropertyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String property = context.getEnvironment().getProperty("app.feature.enabled");
        return "true".equals(property);
    }
}

@Component
@Conditional(OnPropertyCondition.class)
public class FeatureService {
    // Сервис, активируемый только при включенной функции
}
```

### 4. Проверка наличия бина

```java
public class OnBeanCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return context.getBeanFactory().containsBean("dataSource");
    }
}

@Configuration
@Conditional(OnBeanCondition.class)
public class CacheConfiguration {
    // Конфигурация кэша только при наличии DataSource
}
```

## Встроенные условия Spring Boot

Spring Boot предоставляет множество готовых условий:

### @ConditionalOnClass
```java
@ConditionalOnClass(name = "org.postgresql.Driver")
@Configuration
public class PostgresConfiguration {
    // Конфигурация только при наличии PostgreSQL драйвера
}
```

### @ConditionalOnProperty
```java
@ConditionalOnProperty(name = "app.cache.enabled", havingValue = "true")
@Configuration
public class CacheConfiguration {
    // Конфигурация кэша только при включенном свойстве
}
```

### @ConditionalOnBean
```java
@ConditionalOnBean(DataSource.class)
@Configuration
public class JpaConfiguration {
    // Конфигурация JPA только при наличии DataSource
}
```

### @ConditionalOnMissingBean
```java
@ConditionalOnMissingBean(DataSource.class)
@Bean
public DataSource defaultDataSource() {
    // DataSource по умолчанию только если нет другого
}
```

## Связь с @Profile

> **Интересный факт:** `@Profile` на самом деле тоже работает через механизм `@Conditional`

```java
@Profile("dev")
public class DevService {
    // Эквивалентно:
    // @Conditional(OnProfileCondition.class)
}
```

## Лучшие практики

1. **Создавайте переиспользуемые условия** - выносите логику в отдельные классы
2. **Используйте встроенные условия Spring Boot** - они покрывают большинство случаев
3. **Документируйте условия** - объясняйте, когда и почему активируется конфигурация
4. **Тестируйте условия** - убедитесь, что они работают корректно

## Связи с другими темами

- [[Spring Configuration]] - Способы конфигурации
- [[Spring Boot]] - Автоконфигурация
- [[Environment и @Profile]] - Профили приложения
- [[BeanPostProcessor]] - Расширение функциональности 