# Transaction propagation

## Обзор

Transaction propagation (распространение транзакций) определяет, как транзакции взаимодействуют друг с другом при вложенных вызовах методов. Spring предоставляет семь типов распространения транзакций, каждый из которых имеет свое специфическое поведение.

## Типы распространения

### 1. REQUIRED (по умолчанию)

```java
@Service
@Transactional(propagation = Propagation.REQUIRED)
public class UserService {
    
    public void createUser(User user) {
        // Если транзакция уже существует, использует её
        // Если нет, создает новую транзакцию
        userRepository.save(user);
    }
}
```

**Поведение:**
- Если транзакция уже существует, участвует в ней
- Если транзакции нет, создает новую

### 2. REQUIRES_NEW

```java
@Service
@Transactional(propagation = Propagation.REQUIRES_NEW)
public class LogService {
    
    public void logActivity(String activity) {
        // Всегда создает новую транзакцию
        // Приостанавливает существующую транзакцию
        logRepository.save(new Log(activity));
    }
}
```

**Поведение:**
- Всегда создает новую транзакцию
- Приостанавливает существующую транзакцию
- Независимо от родительской транзакции

### 3. MANDATORY

```java
@Service
@Transactional(propagation = Propagation.MANDATORY)
public class AuditService {
    
    public void auditOperation(String operation) {
        // Требует существующую транзакцию
        // Выбрасывает исключение, если транзакции нет
        auditRepository.save(new Audit(operation));
    }
}
```

**Поведение:**
- Требует существующую транзакцию
- Выбрасывает `IllegalTransactionStateException`, если транзакции нет

### 4. SUPPORTS

```java
@Service
@Transactional(propagation = Propagation.SUPPORTS)
public class ReadOnlyService {
    
    public List<User> findAllUsers() {
        // Работает в транзакции, если она есть
        // Работает без транзакции, если её нет
        return userRepository.findAll();
    }
}
```

**Поведение:**
- Работает в существующей транзакции
- Работает без транзакции, если её нет

### 5. NOT_SUPPORTED

```java
@Service
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public class ExternalService {
    
    public void callExternalAPI(String data) {
        // Приостанавливает существующую транзакцию
        // Выполняется без транзакции
        externalApiClient.send(data);
    }
}
```

**Поведение:**
- Приостанавливает существующую транзакцию
- Выполняется без транзакции

### 6. NEVER

```java
@Service
@Transactional(propagation = Propagation.NEVER)
public class NonTransactionalService {
    
    public void performOperation() {
        // Выбрасывает исключение, если есть активная транзакция
        // Выполняется без транзакции, если её нет
        performNonTransactionalOperation();
    }
}
```

**Поведение:**
- Выбрасывает исключение, если есть активная транзакция
- Выполняется без транзакции, если её нет

### 7. NESTED

```java
@Service
@Transactional(propagation = Propagation.NESTED)
public class NestedService {
    
    public void nestedOperation() {
        // Создает вложенную транзакцию (если поддерживается)
        // Иначе ведет себя как REQUIRED
        performNestedOperation();
    }
}
```

**Поведение:**
- Создает вложенную транзакцию (если поддерживается СУБД)
- Иначе ведет себя как REQUIRED

## Практические примеры

### 1. Сценарий с REQUIRED

```java
@Service
@Transactional(propagation = Propagation.REQUIRED)
public class OrderService {
    
    @Autowired
    private UserService userService;
    
    public void createOrder(Order order, User user) {
        // Начинается транзакция
        orderRepository.save(order);
        
        // Вызывается в той же транзакции
        userService.updateUserBalance(user, order.getAmount());
        
        // Если любой метод выбросит исключение, вся транзакция откатится
    }
}

@Service
@Transactional(propagation = Propagation.REQUIRED)
public class UserService {
    
    public void updateUserBalance(User user, BigDecimal amount) {
        // Участвует в транзакции OrderService
        user.setBalance(user.getBalance().subtract(amount));
        userRepository.save(user);
    }
}
```

### 2. Сценарий с REQUIRES_NEW

```java
@Service
@Transactional(propagation = Propagation.REQUIRES_NEW)
public class LogService {
    
    public void logActivity(String activity) {
        // Создается новая транзакция
        // Приостанавливается транзакция OrderService
        logRepository.save(new Log(activity));
        
        // Если здесь произойдет исключение, откатится только эта транзакция
        // Транзакция OrderService не пострадает
    }
}

@Service
@Transactional(propagation = Propagation.REQUIRED)
public class OrderService {
    
    @Autowired
    private LogService logService;
    
    public void createOrder(Order order) {
        try {
            orderRepository.save(order);
            
            // Вызывается в новой транзакции
            logService.logActivity("Order created: " + order.getId());
            
        } catch (Exception e) {
            // Если logService выбросит исключение, order все равно сохранится
            // Потому что они в разных транзакциях
        }
    }
}
```

### 3. Сценарий с MANDATORY

```java
@Service
@Transactional(propagation = Propagation.MANDATORY)
public class AuditService {
    
    public void auditOperation(String operation) {
        // Требует существующую транзакцию
        auditRepository.save(new Audit(operation));
    }
}

@Service
@Transactional(propagation = Propagation.REQUIRED)
public class UserService {
    
    @Autowired
    private AuditService auditService;
    
    public void createUser(User user) {
        // Начинается транзакция
        userRepository.save(user);
        
        // Вызывается в той же транзакции
        auditService.auditOperation("User created: " + user.getId());
    }
    
    public void findUser(Long id) {
        // ❌ Ошибка: нет транзакции
        // auditService.auditOperation("User found: " + id);
    }
}
```

### 4. Сценарий с SUPPORTS

```java
@Service
@Transactional(propagation = Propagation.SUPPORTS)
public class ReadOnlyService {
    
    public List<User> findAllUsers() {
        // Работает в транзакции, если она есть
        // Работает без транзакции, если её нет
        return userRepository.findAll();
    }
}

@Service
@Transactional(propagation = Propagation.REQUIRED)
public class UserService {
    
    @Autowired
    private ReadOnlyService readOnlyService;
    
    public void processUsers() {
        // Вызывается в транзакции
        List<User> users = readOnlyService.findAllUsers();
        // Обработка пользователей
    }
}

// Вызов без транзакции
public void standaloneMethod() {
    // Вызывается без транзакции
    List<User> users = readOnlyService.findAllUsers();
}
```

### 5. Сценарий с NOT_SUPPORTED

```java
@Service
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public class ExternalService {
    
    public void callExternalAPI(String data) {
        // Приостанавливает существующую транзакцию
        externalApiClient.send(data);
    }
}

@Service
@Transactional(propagation = Propagation.REQUIRED)
public class OrderService {
    
    @Autowired
    private ExternalService externalService;
    
    public void createOrder(Order order) {
        // Начинается транзакция
        orderRepository.save(order);
        
        // Приостанавливается транзакция, вызывается внешний API
        externalService.callExternalAPI("Order created: " + order.getId());
        
        // Транзакция возобновляется
        order.setStatus("PROCESSED");
        orderRepository.save(order);
    }
}
```

### 6. Сценарий с NEVER

```java
@Service
@Transactional(propagation = Propagation.NEVER)
public class NonTransactionalService {
    
    public void performOperation() {
        // Выполняется только без транзакции
        performNonTransactionalOperation();
    }
}

@Service
@Transactional(propagation = Propagation.REQUIRED)
public class UserService {
    
    @Autowired
    private NonTransactionalService nonTransactionalService;
    
    public void createUser(User user) {
        // ❌ Ошибка: есть активная транзакция
        // nonTransactionalService.performOperation();
        
        userRepository.save(user);
    }
    
    public void standaloneOperation() {
        // ✅ Работает: нет активной транзакции
        nonTransactionalService.performOperation();
    }
}
```

### 7. Сценарий с NESTED

```java
@Service
@Transactional(propagation = Propagation.NESTED)
public class NestedService {
    
    public void nestedOperation() {
        // Создает вложенную транзакцию (если поддерживается)
        performNestedOperation();
    }
}

@Service
@Transactional(propagation = Propagation.REQUIRED)
public class OrderService {
    
    @Autowired
    private NestedService nestedService;
    
    public void createOrder(Order order) {
        // Начинается транзакция
        orderRepository.save(order);
        
        try {
            // Создается вложенная транзакция
            nestedService.nestedOperation();
        } catch (Exception e) {
            // Если вложенная транзакция откатится, основная транзакция продолжит работу
            // (если поддерживается СУБД)
        }
    }
}
```

## Комбинирование типов распространения

### Сложный сценарий

```java
@Service
@Transactional(propagation = Propagation.REQUIRED)
public class OrderService {
    
    @Autowired
    private UserService userService;
    @Autowired
    private LogService logService;
    @Autowired
    private ExternalService externalService;
    
    public void createOrder(Order order, User user) {
        // REQUIRED: создается транзакция
        orderRepository.save(order);
        
        // REQUIRED: участвует в той же транзакции
        userService.updateUserBalance(user, order.getAmount());
        
        try {
            // REQUIRES_NEW: создается новая транзакция
            logService.logActivity("Order created: " + order.getId());
        } catch (Exception e) {
            // Логирование не повлияет на основную транзакцию
        }
        
        try {
            // NOT_SUPPORTED: приостанавливает транзакцию
            externalService.callExternalAPI("Order notification");
        } catch (Exception e) {
            // Внешний вызов не повлияет на транзакцию
        }
        
        // REQUIRED: продолжается основная транзакция
        order.setStatus("COMPLETED");
        orderRepository.save(order);
    }
}
```

## Лучшие практики

### 1. Выбирайте правильный тип распространения

```java
// Для операций записи
@Transactional(propagation = Propagation.REQUIRED)
public void createUser(User user) {
    // Создает или участвует в транзакции
}

// Для операций логирования
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logActivity(String activity) {
    // Всегда в новой транзакции
}

// Для операций чтения
@Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
public List<User> findAllUsers() {
    // Работает с транзакцией или без неё
}
```

### 2. Избегайте сложных вложений

```java
// ❌ Плохо: сложные вложения
@Service
public class ComplexService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void method1() {
        method2();
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void method2() {
        method3();
    }
    
    @Transactional(propagation = Propagation.NESTED)
    public void method3() {
        // Сложно отследить поведение
    }
}

// ✅ Хорошо: простые и понятные сценарии
@Service
public class SimpleService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void businessOperation() {
        // Основная бизнес-логика
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditOperation() {
        // Аудит в отдельной транзакции
    }
}
```

### 3. Документируйте намерения

```java
/**
 * Создает пользователя в транзакции.
 * Если транзакция уже существует, участвует в ней.
 * Если нет, создает новую транзакцию.
 */
@Transactional(propagation = Propagation.REQUIRED)
public void createUser(User user) {
    // Реализация
}

/**
 * Логирует активность в новой транзакции.
 * Независимо от существующих транзакций.
 */
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logActivity(String activity) {
    // Реализация
}
```

## Отладка и мониторинг

### Включение логирования

```properties
# application.properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.orm.jpa=DEBUG
```

### Проверка типа распространения

```java
@Component
public class PropagationChecker {
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkPropagation(ApplicationReadyEvent event) {
        // Проверяем настройки распространения
        ApplicationContext context = event.getApplicationContext();
        
        // Анализируем аннотации @Transactional
        // и их настройки propagation
    }
}
``` 