# Функционал TransactionTemplate

## Обзор

`TransactionTemplate` предоставляет программный способ управления транзакциями в Spring. Он позволяет выполнять код в транзакционном контексте с полным контролем над настройками транзакций.

## Основные методы

### 1. execute(TransactionCallback)

```java
@Service
public class UserService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void createUser(User user) {
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                userRepository.save(user);
            }
        });
    }
}
```

### 2. execute(TransactionCallback) с возвращаемым значением

```java
@Service
public class UserService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public User createUser(User user) {
        return transactionTemplate.execute(new TransactionCallback<User>() {
            @Override
            public User doInTransaction(TransactionStatus status) {
                User savedUser = userRepository.save(user);
                return savedUser;
            }
        });
    }
}
```

### 3. execute с лямбда-выражениями

```java
@Service
public class UserService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void createUser(User user) {
        transactionTemplate.execute(status -> {
            userRepository.save(user);
            return null;
        });
    }
    
    public User createUserWithReturn(User user) {
        return transactionTemplate.execute(status -> {
            return userRepository.save(user);
        });
    }
}
```

## Сценарии использования

### 1. Простые операции

```java
@Service
public class SimpleTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void saveUser(User user) {
        transactionTemplate.execute(status -> {
            userRepository.save(user);
            return null;
        });
    }
    
    public List<User> findAllUsers() {
        return transactionTemplate.execute(status -> {
            return userRepository.findAll();
        });
    }
}
```

### 2. Сложные операции с множественными шагами

```java
@Service
public class ComplexTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void createOrderWithUser(Order order, User user) {
        transactionTemplate.execute(status -> {
            // Шаг 1: Сохраняем пользователя
            User savedUser = userRepository.save(user);
            
            // Шаг 2: Устанавливаем связь
            order.setUser(savedUser);
            
            // Шаг 3: Сохраняем заказ
            Order savedOrder = orderRepository.save(order);
            
            // Шаг 4: Обновляем баланс пользователя
            savedUser.setBalance(savedUser.getBalance().subtract(order.getAmount()));
            userRepository.save(savedUser);
            
            return savedOrder;
        });
    }
}
```

### 3. Операции с проверкой статуса транзакции

```java
@Service
public class StatusAwareTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void performOperation() {
        transactionTemplate.execute(status -> {
            // Проверяем, новая ли это транзакция
            if (status.isNewTransaction()) {
                System.out.println("Создана новая транзакция");
            } else {
                System.out.println("Участвуем в существующей транзакции");
            }
            
            // Проверяем, помечена ли транзакция для отката
            if (status.isRollbackOnly()) {
                System.out.println("Транзакция помечена для отката");
            }
            
            // Выполняем бизнес-логику
            performBusinessLogic();
            
            return null;
        });
    }
}
```

### 4. Операции с обработкой исключений

```java
@Service
public class ExceptionHandlingTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void performOperationWithExceptionHandling() {
        try {
            transactionTemplate.execute(status -> {
                try {
                    // Бизнес-логика
                    performBusinessLogic();
                    
                    // Если что-то пошло не так, помечаем для отката
                    if (someCondition) {
                        status.setRollbackOnly();
                    }
                    
                } catch (BusinessException e) {
                    // Помечаем транзакцию для отката
                    status.setRollbackOnly();
                    throw e;
                }
                
                return null;
            });
        } catch (TransactionException e) {
            // Обработка транзакционных исключений
            handleTransactionException(e);
        } catch (BusinessException e) {
            // Обработка бизнес-исключений
            handleBusinessException(e);
        }
    }
}
```

## Интеграция с различными компонентами

### 1. Интеграция с Repository

```java
@Service
public class RepositoryTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    @Autowired
    private UserRepository userRepository;
    @Autowired
    private OrderRepository orderRepository;
    
    public void createUserWithOrders(User user, List<Order> orders) {
        transactionTemplate.execute(status -> {
            // Сохраняем пользователя
            User savedUser = userRepository.save(user);
            
            // Сохраняем заказы
            for (Order order : orders) {
                order.setUser(savedUser);
                orderRepository.save(order);
            }
            
            return savedUser;
        });
    }
}
```

### 2. Интеграция с JdbcTemplate

```java
@Service
public class JdbcTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void performBatchOperation(List<String> queries) {
        transactionTemplate.execute(status -> {
            for (String query : queries) {
                jdbcTemplate.execute(query);
            }
            return null;
        });
    }
    
    public List<Map<String, Object>> executeQuery(String sql) {
        return transactionTemplate.execute(status -> {
            return jdbcTemplate.queryForList(sql);
        });
    }
}
```

### 3. Интеграция с EntityManager

```java
@Service
public class EntityManagerTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    @Autowired
    private EntityManager entityManager;
    
    public void performEntityManagerOperation() {
        transactionTemplate.execute(status -> {
            // Используем EntityManager напрямую
            TypedQuery<User> query = entityManager.createQuery(
                "SELECT u FROM User u WHERE u.active = true", User.class);
            
            List<User> users = query.getResultList();
            
            // Модифицируем пользователей
            for (User user : users) {
                user.setLastLogin(LocalDateTime.now());
                entityManager.merge(user);
            }
            
            return users;
        });
    }
}
```

## Продвинутые сценарии

### 1. Условные транзакции

```java
@Service
public class ConditionalTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void performConditionalOperation(boolean useTransaction) {
        if (useTransaction) {
            transactionTemplate.execute(status -> {
                performOperation();
                return null;
            });
        } else {
            performOperation();
        }
    }
    
    public void performOperationWithRetry() {
        int maxRetries = 3;
        int retryCount = 0;
        
        while (retryCount < maxRetries) {
            try {
                transactionTemplate.execute(status -> {
                    performOperation();
                    return null;
                });
                break; // Успешно выполнили
            } catch (TransactionException e) {
                retryCount++;
                if (retryCount >= maxRetries) {
                    throw e;
                }
                // Ждем перед повторной попыткой
                try {
                    Thread.sleep(1000 * retryCount);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(ie);
                }
            }
        }
    }
}
```

### 2. Вложенные транзакции

```java
@Service
public class NestedTransactionService {
    
    @Autowired
    private TransactionTemplate outerTemplate;
    @Autowired
    private TransactionTemplate innerTemplate;
    
    public void performNestedOperation() {
        outerTemplate.execute(outerStatus -> {
            // Внешняя транзакция
            performOuterOperation();
            
            try {
                innerTemplate.execute(innerStatus -> {
                    // Внутренняя транзакция
                    performInnerOperation();
                    return null;
                });
            } catch (Exception e) {
                // Обработка исключений внутренней транзакции
                handleInnerException(e);
            }
            
            return null;
        });
    }
}
```

### 3. Асинхронные операции

```java
@Service
public class AsyncTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    @Async
    public CompletableFuture<Void> performAsyncOperation() {
        return CompletableFuture.runAsync(() -> {
            transactionTemplate.execute(status -> {
                performOperation();
                return null;
            });
        });
    }
    
    public void performOperationWithAsyncCallback() {
        transactionTemplate.execute(status -> {
            // Синхронная операция
            performSyncOperation();
            
            // Асинхронная операция после коммита
            CompletableFuture.runAsync(() -> {
                performAsyncOperation();
            });
            
            return null;
        });
    }
}
```

## Мониторинг и профилирование

### 1. Логирование транзакций

```java
@Service
public class LoggingTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void performOperationWithLogging() {
        long startTime = System.currentTimeMillis();
        
        transactionTemplate.execute(status -> {
            System.out.println("Начало транзакции: " + System.currentTimeMillis());
            System.out.println("Новая транзакция: " + status.isNewTransaction());
            
            performOperation();
            
            System.out.println("Завершение транзакции: " + System.currentTimeMillis());
            return null;
        });
        
        long endTime = System.currentTimeMillis();
        System.out.println("Общее время выполнения: " + (endTime - startTime) + "ms");
    }
}
```

### 2. Метрики производительности

```java
@Component
public class TransactionMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public TransactionMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void recordTransactionMetrics(String operation, long duration) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        meterRegistry.timer("transaction.duration", "operation", operation)
            .record(duration, TimeUnit.MILLISECONDS);
        
        meterRegistry.counter("transaction.count", "operation", operation)
            .increment();
    }
}

@Service
public class MetricsTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    @Autowired
    private TransactionMetrics metrics;
    
    public void performOperationWithMetrics(String operation) {
        long startTime = System.currentTimeMillis();
        
        try {
            transactionTemplate.execute(status -> {
                performOperation();
                return null;
            });
            
            long duration = System.currentTimeMillis() - startTime;
            metrics.recordTransactionMetrics(operation, duration);
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            metrics.recordTransactionMetrics(operation + ".error", duration);
            throw e;
        }
    }
}
```

## Лучшие практики

### 1. Создавайте специализированные шаблоны

```java
@Configuration
public class SpecializedTemplatesConfig {
    
    @Bean("readOnlyTemplate")
    public TransactionTemplate readOnlyTemplate(PlatformTransactionManager tm) {
        TransactionTemplate template = new TransactionTemplate(tm);
        template.setReadOnly(true);
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_SUPPORTS);
        return template;
    }
    
    @Bean("writeTemplate")
    public TransactionTemplate writeTemplate(PlatformTransactionManager tm) {
        TransactionTemplate template = new TransactionTemplate(tm);
        template.setReadOnly(false);
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        return template;
    }
}
```

### 2. Используйте лямбда-выражения для читаемости

```java
@Service
public class CleanTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void performOperation() {
        transactionTemplate.execute(status -> {
            // Чистый и читаемый код
            performBusinessLogic();
            return null;
        });
    }
}
```

### 3. Обрабатывайте исключения правильно

```java
@Service
public class ExceptionHandlingService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void performOperationWithProperExceptionHandling() {
        try {
            transactionTemplate.execute(status -> {
                performOperation();
                return null;
            });
        } catch (TransactionException e) {
            // Логируем транзакционные ошибки
            log.error("Transaction failed", e);
            throw new BusinessException("Operation failed", e);
        } catch (RuntimeException e) {
            // Логируем бизнес-ошибки
            log.error("Business operation failed", e);
            throw e;
        }
    }
}
```

### 4. Документируйте сложные операции

```java
/**
 * Выполняет сложную операцию с множественными шагами в транзакции.
 * Все шаги выполняются атомарно - либо все успешно, либо все откатываются.
 */
public void performComplexOperation() {
    transactionTemplate.execute(status -> {
        // Шаг 1: Валидация
        validateData();
        
        // Шаг 2: Сохранение основных данных
        saveMainData();
        
        // Шаг 3: Обновление связанных данных
        updateRelatedData();
        
        // Шаг 4: Отправка уведомлений
        sendNotifications();
        
        return null;
    });
}
``` 