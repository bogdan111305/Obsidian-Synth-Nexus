# Свойства объекта TransactionTemplate

## Обзор

`TransactionTemplate` - это класс Spring, который предоставляет программный способ управления транзакциями. Он позволяет настраивать различные свойства транзакций и выполнять код в транзакционном контексте.

## Основные свойства

### 1. PropagationBehavior

```java
@Configuration
public class TransactionTemplateConfig {
    
    @Bean
    public TransactionTemplate transactionTemplate(PlatformTransactionManager tm) {
        TransactionTemplate template = new TransactionTemplate(tm);
        
        // Настройка типа распространения
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        
        return template;
    }
}
```

**Доступные значения:**
- `PROPAGATION_REQUIRED` - Создает новую транзакцию или участвует в существующей
- `PROPAGATION_REQUIRES_NEW` - Всегда создает новую транзакцию
- `PROPAGATION_MANDATORY` - Требует существующую транзакцию
- `PROPAGATION_SUPPORTS` - Работает в транзакции или без неё
- `PROPAGATION_NOT_SUPPORTED` - Приостанавливает транзакцию
- `PROPAGATION_NEVER` - Выбрасывает исключение при наличии транзакции
- `PROPAGATION_NESTED` - Создает вложенную транзакцию

### 2. IsolationLevel

```java
@Bean
public TransactionTemplate transactionTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    
    // Настройка уровня изоляции
    template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    
    return template;
}
```

**Доступные значения:**
- `ISOLATION_DEFAULT` - Уровень по умолчанию СУБД
- `ISOLATION_READ_UNCOMMITTED` - Самый низкий уровень изоляции
- `ISOLATION_READ_COMMITTED` - Стандартный уровень
- `ISOLATION_REPEATABLE_READ` - Повторяемое чтение
- `ISOLATION_SERIALIZABLE` - Самый высокий уровень изоляции

### 3. Timeout

```java
@Bean
public TransactionTemplate transactionTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    
    // Настройка таймаута (в секундах)
    template.setTimeout(30);
    
    return template;
}
```

### 4. ReadOnly

```java
@Bean
public TransactionTemplate readOnlyTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    
    // Настройка только для чтения
    template.setReadOnly(true);
    
    return template;
}
```

## Конфигурация различных шаблонов

### 1. Шаблон для операций записи

```java
@Configuration
public class WriteTransactionConfig {
    
    @Bean("writeTransactionTemplate")
    public TransactionTemplate writeTransactionTemplate(PlatformTransactionManager tm) {
        TransactionTemplate template = new TransactionTemplate(tm);
        
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
        template.setTimeout(60);
        template.setReadOnly(false);
        
        return template;
    }
}
```

### 2. Шаблон для операций чтения

```java
@Bean("readTransactionTemplate")
public TransactionTemplate readTransactionTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    
    template.setPropagationBehavior(TransactionDefinition.PROPAGATION_SUPPORTS);
    template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    template.setTimeout(10);
    template.setReadOnly(true);
    
    return template;
}
```

### 3. Шаблон для критических операций

```java
@Bean("criticalTransactionTemplate")
public TransactionTemplate criticalTransactionTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    
    template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    template.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
    template.setTimeout(120);
    template.setReadOnly(false);
    
    return template;
}
```

### 4. Шаблон для быстрых операций

```java
@Bean("fastTransactionTemplate")
public TransactionTemplate fastTransactionTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    
    template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    template.setTimeout(5);
    template.setReadOnly(true);
    
    return template;
}
```

## Использование в сервисах

### 1. Базовое использование

```java
@Service
public class UserService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void createUser(User user) {
        transactionTemplate.execute(status -> {
            // Код выполняется в транзакции
            userRepository.save(user);
            return null;
        });
    }
}
```

### 2. Использование с возвращаемым значением

```java
@Service
public class UserService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public User createUser(User user) {
        return transactionTemplate.execute(status -> {
            User savedUser = userRepository.save(user);
            return savedUser;
        });
    }
}
```

### 3. Использование с обработкой исключений

```java
@Service
public class UserService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void createUser(User user) {
        try {
            transactionTemplate.execute(status -> {
                userRepository.save(user);
                
                // Проверяем статус транзакции
                if (status.isNewTransaction()) {
                    System.out.println("Новая транзакция создана");
                }
                
                return null;
            });
        } catch (TransactionException e) {
            // Обработка транзакционных исключений
            throw new RuntimeException("Ошибка при создании пользователя", e);
        }
    }
}
```

### 4. Использование разных шаблонов

```java
@Service
public class MultiTemplateService {
    
    @Autowired
    @Qualifier("writeTransactionTemplate")
    private TransactionTemplate writeTemplate;
    
    @Autowired
    @Qualifier("readTransactionTemplate")
    private TransactionTemplate readTemplate;
    
    @Autowired
    @Qualifier("criticalTransactionTemplate")
    private TransactionTemplate criticalTemplate;
    
    public void createUser(User user) {
        // Используем шаблон для записи
        writeTemplate.execute(status -> {
            userRepository.save(user);
            return null;
        });
    }
    
    public List<User> findAllUsers() {
        // Используем шаблон для чтения
        return readTemplate.execute(status -> {
            return userRepository.findAll();
        });
    }
    
    public void criticalOperation() {
        // Используем шаблон для критических операций
        criticalTemplate.execute(status -> {
            performCriticalOperation();
            return null;
        });
    }
}
```

## Программная настройка свойств

### 1. Динамическая настройка

```java
@Service
public class DynamicTransactionService {
    
    @Autowired
    private PlatformTransactionManager transactionManager;
    
    public void executeWithCustomSettings() {
        TransactionTemplate template = new TransactionTemplate(transactionManager);
        
        // Настройка свойств программно
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
        template.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
        template.setTimeout(30);
        template.setReadOnly(false);
        
        template.execute(status -> {
            // Выполнение с кастомными настройками
            return null;
        });
    }
}
```

### 2. Настройка на основе условий

```java
@Service
public class ConditionalTransactionService {
    
    @Autowired
    private PlatformTransactionManager transactionManager;
    
    public void executeBasedOnCondition(boolean isCritical) {
        TransactionTemplate template = new TransactionTemplate(transactionManager);
        
        if (isCritical) {
            template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
            template.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
            template.setTimeout(60);
        } else {
            template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
            template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
            template.setTimeout(10);
        }
        
        template.execute(status -> {
            // Выполнение с настройками на основе условий
            return null;
        });
    }
}
```

## Обработка исключений

### 1. Настройка исключений для отката

```java
@Bean
public TransactionTemplate rollbackTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    
    // Настройка исключений для отката
    template.setRollbackOnCommitFailure(true);
    
    return template;
}
```

### 2. Кастомная обработка исключений

```java
@Service
public class ExceptionHandlingService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void executeWithExceptionHandling() {
        try {
            transactionTemplate.execute(status -> {
                // Бизнес-логика
                performOperation();
                return null;
            });
        } catch (TransactionException e) {
            // Обработка транзакционных исключений
            handleTransactionException(e);
        } catch (RuntimeException e) {
            // Обработка других исключений
            handleRuntimeException(e);
        }
    }
}
```

## Мониторинг и отладка

### 1. Проверка настроек шаблона

```java
@Component
public class TransactionTemplateChecker {
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkTemplates(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        Map<String, TransactionTemplate> templates = 
            context.getBeansOfType(TransactionTemplate.class);
        
        for (Map.Entry<String, TransactionTemplate> entry : templates.entrySet()) {
            TransactionTemplate template = entry.getValue();
            
            System.out.println("Template: " + entry.getKey());
            System.out.println("Propagation: " + template.getPropagationBehavior());
            System.out.println("Isolation: " + template.getIsolationLevel());
            System.out.println("Timeout: " + template.getTimeout());
            System.out.println("ReadOnly: " + template.isReadOnly());
        }
    }
}
```

### 2. Логирование транзакций

```java
@Service
public class LoggingTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void executeWithLogging() {
        transactionTemplate.execute(status -> {
            System.out.println("Transaction started");
            System.out.println("Is new transaction: " + status.isNewTransaction());
            System.out.println("Is rollback only: " + status.isRollbackOnly());
            
            // Бизнес-логика
            performOperation();
            
            System.out.println("Transaction completed");
            return null;
        });
    }
}
```

## Лучшие практики

### 1. Создавайте специализированные шаблоны

```java
@Configuration
public class SpecializedTemplatesConfig {
    
    @Bean("fastReadTemplate")
    public TransactionTemplate fastReadTemplate(PlatformTransactionManager tm) {
        TransactionTemplate template = new TransactionTemplate(tm);
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_SUPPORTS);
        template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
        template.setTimeout(5);
        template.setReadOnly(true);
        return template;
    }
    
    @Bean("criticalWriteTemplate")
    public TransactionTemplate criticalWriteTemplate(PlatformTransactionManager tm) {
        TransactionTemplate template = new TransactionTemplate(tm);
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
        template.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
        template.setTimeout(60);
        template.setReadOnly(false);
        return template;
    }
}
```

### 2. Документируйте назначение шаблонов

```java
/**
 * Шаблон для быстрых операций чтения.
 * Использует SUPPORTS propagation и READ_COMMITTED isolation.
 */
@Bean("fastReadTemplate")
public TransactionTemplate fastReadTemplate(PlatformTransactionManager tm) {
    // Реализация
}

/**
 * Шаблон для критических операций записи.
 * Использует REQUIRES_NEW propagation и SERIALIZABLE isolation.
 */
@Bean("criticalWriteTemplate")
public TransactionTemplate criticalWriteTemplate(PlatformTransactionManager tm) {
    // Реализация
}
```

### 3. Используйте константы для читаемости

```java
@Bean
public TransactionTemplate transactionTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    
    // Используем константы для лучшей читаемости
    template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    template.setTimeout(TransactionTemplate.TIMEOUT_DEFAULT);
    template.setReadOnly(false);
    
    return template;
}
``` 