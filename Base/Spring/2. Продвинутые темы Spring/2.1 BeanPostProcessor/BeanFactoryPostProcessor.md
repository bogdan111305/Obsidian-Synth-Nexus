# BeanFactoryPostProcessor

BeanFactoryPostProcessor - это интерфейс Spring Framework, который позволяет модифицировать BeanFactory и BeanDefinition до создания экземпляров бинов. Это мощный механизм для кастомизации конфигурации на уровне метаданных.

## Содержание

1. [Основы BeanFactoryPostProcessor](#основы-beanfactorypostprocessor)
2. [Интерфейс и методы](#интерфейс-и-методы)
3. [Порядок выполнения](#порядок-выполнения)
4. [Практические примеры](#практические-примеры)
5. [Встроенные BeanFactoryPostProcessor](#встроенные-beanfactorypostprocessor)
6. [Лучшие практики](#лучшие-практики)

## Основы BeanFactoryPostProcessor

BeanFactoryPostProcessor работает на уровне **BeanDefinition**, а не экземпляров бинов. Это означает, что он может модифицировать метаданные бинов до их создания.

### Ключевые особенности

- **Выполняется до создания бинов** - работает с BeanDefinition
- **Может модифицировать свойства** - изменять конфигурацию бинов
- **Функциональный интерфейс** - имеет только один метод
- **Выполняется раньше BeanPostProcessor** - в жизненном цикле Spring

## Интерфейс и методы

### Основной интерфейс

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

### Пример реализации

```java
@Component
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("Processing BeanFactory...");
        
        // Получение всех имен бинов
        String[] beanNames = beanFactory.getBeanDefinitionNames();
        
        for (String beanName : beanNames) {
            BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
            
            // Модификация BeanDefinition
            if (beanDefinition.getBeanClassName() != null && 
                beanDefinition.getBeanClassName().contains("Service")) {
                
                // Добавление свойства
                beanDefinition.getPropertyValues().add("customProperty", "customValue");
            }
        }
    }
}
```

## Порядок выполнения

BeanFactoryPostProcessor может реализовывать интерфейсы для управления порядком выполнения:

### 1. Ordered интерфейс

```java
@Component
public class OrderedBeanFactoryPostProcessor implements BeanFactoryPostProcessor, Ordered {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("Ordered BFPP executed");
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

### 2. PriorityOrdered интерфейс

```java
@Component
public class PriorityBeanFactoryPostProcessor implements BeanFactoryPostProcessor, PriorityOrdered {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("Priority BFPP executed");
    }
    
    @Override
    public int getOrder() {
        return 100;
    }
}
```

### Приоритеты выполнения

1. **PriorityOrdered** - наивысший приоритет
2. **Ordered** - обычный приоритет
3. **Без приоритета** - выполняется в порядке регистрации

## Практические примеры

### 1. Модификация свойств бинов

```java
@Component
public class PropertyModificationBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        String[] beanNames = beanFactory.getBeanDefinitionNames();
        
        for (String beanName : beanNames) {
            BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
            
            // Добавление свойства к DataSource бинам
            if (beanDefinition.getBeanClassName() != null && 
                beanDefinition.getBeanClassName().contains("DataSource")) {
                
                MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();
                propertyValues.add("validationQuery", "SELECT 1");
                propertyValues.add("testOnBorrow", "true");
            }
        }
    }
}
```

### 2. Условная регистрация бинов

```java
@Component
public class ConditionalBeanRegistrationPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // Проверяем наличие определенного свойства
        Environment environment = beanFactory.getBean(Environment.class);
        boolean enableFeature = environment.getProperty("app.feature.enabled", Boolean.class, false);
        
        if (!enableFeature) {
            // Удаляем бин если свойство не включено
            if (beanFactory.containsBean("featureService")) {
                ((DefaultListableBeanFactory) beanFactory).removeBeanDefinition("featureService");
            }
        }
    }
}
```

### 3. Добавление кастомных свойств

```java
@Component
public class CustomPropertyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        String[] beanNames = beanFactory.getBeanDefinitionNames();
        
        for (String beanName : beanNames) {
            BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
            
            // Добавление кастомного свойства ко всем бинам
            MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();
            propertyValues.add("createdBy", "BeanFactoryPostProcessor");
            propertyValues.add("creationTime", System.currentTimeMillis());
        }
    }
}
```

### 4. Валидация конфигурации

```java
@Component
public class ConfigurationValidationBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // Проверяем наличие обязательных бинов
        String[] requiredBeans = {"dataSource", "transactionManager"};
        
        for (String requiredBean : requiredBeans) {
            if (!beanFactory.containsBean(requiredBean)) {
                throw new BeanDefinitionStoreException(
                    "Required bean '" + requiredBean + "' is not defined"
                );
            }
        }
        
        // Проверяем конфигурацию DataSource
        if (beanFactory.containsBean("dataSource")) {
            BeanDefinition dataSourceDef = beanFactory.getBeanDefinition("dataSource");
            MutablePropertyValues properties = dataSourceDef.getPropertyValues();
            
            if (!properties.contains("url")) {
                throw new BeanDefinitionStoreException(
                    "DataSource must have 'url' property defined"
                );
            }
        }
    }
}
```

## Встроенные BeanFactoryPostProcessor

Spring использует несколько встроенных BeanFactoryPostProcessor:

### 1. PropertySourcesPlaceholderConfigurer

```java
@Configuration
public class AppConfig {
    
    @Bean
    public static PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer() {
        PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
        configurer.setLocations("classpath:application.properties");
        return configurer;
    }
}
```

### 2. PropertyOverrideConfigurer

```java
@Configuration
public class AppConfig {
    
    @Bean
    public static PropertyOverrideConfigurer propertyOverrideConfigurer() {
        PropertyOverrideConfigurer configurer = new PropertyOverrideConfigurer();
        configurer.setLocation("classpath:overrides.properties");
        return configurer;
    }
}
```

### 3. CustomEditorConfigurer

```java
@Configuration
public class AppConfig {
    
    @Bean
    public static CustomEditorConfigurer customEditorConfigurer() {
        CustomEditorConfigurer configurer = new CustomEditorConfigurer();
        
        Map<Class<?>, Class<? extends PropertyEditor>> customEditors = new HashMap<>();
        customEditors.put(Date.class, CustomDateEditor.class);
        
        configurer.setCustomEditors(customEditors);
        return configurer;
    }
}
```

## Лучшие практики

### 1. Производительность

```java
@Component
public class EfficientBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // Кэшируем имена бинов для производительности
        String[] beanNames = beanFactory.getBeanDefinitionNames();
        
        // Обрабатываем только нужные бины
        Arrays.stream(beanNames)
            .filter(beanName -> beanName.contains("Service"))
            .forEach(beanName -> {
                BeanDefinition definition = beanFactory.getBeanDefinition(beanName);
                // Логика обработки
            });
    }
}
```

### 2. Безопасность

```java
@Component
public class SafeBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        try {
            // Логика обработки
        } catch (Exception e) {
            throw new BeanDefinitionStoreException("Error in BeanFactoryPostProcessor", e);
        }
    }
}
```

### 3. Логирование

```java
@Component
public class LoggingBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingBeanFactoryPostProcessor.class);
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        logger.info("Processing BeanFactory with {} beans", beanFactory.getBeanDefinitionCount());
        
        String[] beanNames = beanFactory.getBeanDefinitionNames();
        for (String beanName : beanNames) {
            logger.debug("Processing bean: {}", beanName);
            // Логика обработки
        }
    }
}
```

### 4. Условная обработка

```java
@Component
@ConditionalOnProperty(name = "app.beanfactory.postprocessor.enabled", havingValue = "true")
public class ConditionalBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // Обработка только при включенном свойстве
    }
}
```

## Связанные темы

- [BeanPostProcessor](./BeanPostProcessor.md)
- [Lifecycle Callbacks](./Lifecycle Callbacks.md)
- [IoC и Dependency Injection](../../1. Основы Spring/1.1 IoC и Dependency Injection/README.md)
- [Конфигурация Spring](../../1. Основы Spring/1.3 Конфигурация/README.md) 