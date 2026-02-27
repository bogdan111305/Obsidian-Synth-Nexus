# Аннотации @Transactional из Jakarta EE и Spring

## Обзор

Аннотация `@Transactional` существует как в Jakarta EE (ранее Java EE), так и в Spring Framework. Хотя они имеют схожие названия и назначение, между ними есть существенные различия в функциональности и использовании.

## Jakarta EE @Transactional

### Основные характеристики

```java
import jakarta.transaction.Transactional;

@Service
public class UserService {
    
    @Transactional
    public void createUser(User user) {
        // Метод выполняется в транзакции
    }
}
```

### Доступные атрибуты

```java
@Transactional(
    value = TxType.REQUIRED,           // Тип распространения
    rollbackOn = Exception.class,      // Исключения для отката
    dontRollbackOn = RuntimeException.class, // Исключения без отката
    timeout = 30                       // Таймаут в секундах
)
public void method() {
    // Реализация
}
```

### Типы распространения (TxType)

```java
public enum TxType {
    REQUIRED,      // Требует транзакцию (создает новую, если нет)
    REQUIRES_NEW,  // Всегда создает новую транзакцию
    MANDATORY,     // Требует существующую транзакцию
    SUPPORTS,      // Поддерживает транзакцию (работает без нее)
    NOT_SUPPORTED, // Не поддерживает транзакцию
    NEVER          // Никогда не выполняется в транзакции
}
```

### Примеры использования

```java
@Service
public class JakartaEEService {
    
    @Transactional(TxType.REQUIRED)
    public void method1() {
        // Выполняется в существующей транзакции или создает новую
    }
    
    @Transactional(TxType.REQUIRES_NEW)
    public void method2() {
        // Всегда создает новую транзакцию
    }
    
    @Transactional(TxType.MANDATORY)
    public void method3() {
        // Требует существующую транзакцию, иначе исключение
    }
    
    @Transactional(TxType.SUPPORTS)
    public void method4() {
        // Работает в транзакции, если она есть
    }
    
    @Transactional(TxType.NOT_SUPPORTED)
    public void method5() {
        // Приостанавливает текущую транзакцию
    }
    
    @Transactional(TxType.NEVER)
    public void method6() {
        // Выбрасывает исключение, если есть активная транзакция
    }
}
```

## Spring @Transactional

### Основные характеристики

```java
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserService {
    
    @Transactional
    public void createUser(User user) {
        // Метод выполняется в транзакции
    }
}
```

### Доступные атрибуты

```java
@Transactional(
    value = "transactionManager",       // Имя менеджера транзакций
    propagation = Propagation.REQUIRED, // Тип распространения
    isolation = Isolation.READ_COMMITTED, // Уровень изоляции
    timeout = 30,                      // Таймаут в секундах
    readOnly = false,                  // Только для чтения
    rollbackFor = Exception.class,     // Исключения для отката
    rollbackForClassName = "Exception", // Имя класса исключения
    noRollbackFor = RuntimeException.class, // Исключения без отката
    noRollbackForClassName = "RuntimeException" // Имя класса исключения
)
public void method() {
    // Реализация
}
```

### Типы распространения (Propagation)

```java
public enum Propagation {
    REQUIRED,      // Создает новую транзакцию или участвует в существующей
    REQUIRES_NEW,  // Всегда создает новую транзакцию
    MANDATORY,     // Требует существующую транзакцию
    SUPPORTS,      // Работает в транзакции или без неё
    NOT_SUPPORTED, // Приостанавливает транзакцию
    NEVER,         // Выбрасывает исключение при наличии транзакции
    NESTED         // Создает вложенную транзакцию
}
```

### Уровни изоляции (Isolation)

```java
public enum Isolation {
    DEFAULT,           // Уровень по умолчанию СУБД
    READ_UNCOMMITTED,  // Самый низкий уровень изоляции
    READ_COMMITTED,    // Стандартный уровень
    REPEATABLE_READ,   // Повторяемое чтение
    SERIALIZABLE       // Самый высокий уровень изоляции
}
```

## Ключевые различия

### 1. Функциональность

| Аспект | Jakarta EE | Spring |
|--------|------------|--------|
| Уровни изоляции | ❌ Не поддерживает | ✅ Полная поддержка |
| Таймауты | ✅ Поддерживает | ✅ Поддерживает |
| ReadOnly | ❌ Не поддерживает | ✅ Поддерживает |
| Множественные менеджеры | ❌ Ограниченная поддержка | ✅ Полная поддержка |
| Программное управление | ❌ Нет | ✅ TransactionTemplate |

### 2. Синтаксис

```java
// Jakarta EE
@Transactional(
    value = TxType.REQUIRED,
    rollbackOn = Exception.class,
    dontRollbackOn = RuntimeException.class,
    timeout = 30
)

// Spring
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED,
    timeout = 30,
    readOnly = false,
    rollbackFor = Exception.class,
    noRollbackFor = RuntimeException.class
)
```

### 3. Обработка исключений

```java
// Jakarta EE
@Transactional(
    rollbackOn = {SQLException.class, IOException.class},
    dontRollbackOn = {RuntimeException.class}
)
public void jakartaMethod() {
    // Исключения для отката и исключения из отката
}

// Spring
@Transactional(
    rollbackFor = {SQLException.class, IOException.class},
    noRollbackFor = {RuntimeException.class}
)
public void springMethod() {
    // Аналогичная функциональность
}
```

## Практические примеры

### 1. Сравнение базового использования

```java
// Jakarta EE
@Service
public class JakartaUserService {
    
    @Transactional(TxType.REQUIRED)
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    @Transactional(TxType.REQUIRES_NEW)
    public void logActivity(String activity) {
        logRepository.save(new Log(activity));
    }
}

// Spring
@Service
public class SpringUserService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logActivity(String activity) {
        logRepository.save(new Log(activity));
    }
}
```

### 2. Работа с уровнями изоляции

```java
// Только в Spring
@Service
public class SpringUserService {
    
    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.READ_COMMITTED
    )
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.SERIALIZABLE
    )
    public void criticalOperation() {
        // Критическая операция с максимальной изоляцией
    }
}
```

### 3. Работа с readOnly

```java
// Только в Spring
@Service
public class SpringUserService {
    
    @Transactional(
        propagation = Propagation.SUPPORTS,
        readOnly = true
    )
    public List<User> findAllUsers() {
        return userRepository.findAll();
    }
    
    @Transactional(
        propagation = Propagation.REQUIRED,
        readOnly = false
    )
    public User createUser(User user) {
        return userRepository.save(user);
    }
}
```

### 4. Работа с множественными менеджерами

```java
// Только в Spring
@Service
public class MultiDataSourceService {
    
    @Transactional(transactionManager = "jpaTransactionManager")
    public void jpaOperation() {
        // Операция с JPA
    }
    
    @Transactional(transactionManager = "jdbcTransactionManager")
    public void jdbcOperation() {
        // Операция с JDBC
    }
}
```

## Миграция между подходами

### Из Jakarta EE в Spring

```java
// До (Jakarta EE)
@Transactional(
    value = TxType.REQUIRED,
    rollbackOn = Exception.class,
    timeout = 30
)
public void method() {
    // Реализация
}

// После (Spring)
@Transactional(
    propagation = Propagation.REQUIRED,
    rollbackFor = Exception.class,
    timeout = 30
)
public void method() {
    // Реализация
}
```

### Из Spring в Jakarta EE

```java
// До (Spring)
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED,
    readOnly = true,
    rollbackFor = Exception.class
)
public void method() {
    // Реализация
}

// После (Jakarta EE) - с потерей функциональности
@Transactional(
    value = TxType.REQUIRED,
    rollbackOn = Exception.class
    // isolation и readOnly недоступны
)
public void method() {
    // Реализация
}
```

## Рекомендации по выбору

### Когда использовать Jakarta EE @Transactional

1. **Проекты на чистом Jakarta EE** - без Spring Framework
2. **Простая функциональность** - когда не нужны уровни изоляции
3. **Стандартизация** - когда важна совместимость со стандартами
4. **Минимальные требования** - базовое управление транзакциями

### Когда использовать Spring @Transactional

1. **Spring Boot приложения** - естественный выбор
2. **Сложные требования** - нужны уровни изоляции, readOnly
3. **Множественные источники данных** - разные менеджеры транзакций
4. **Программное управление** - нужен TransactionTemplate
5. **Гибкость** - больше возможностей настройки

## Лучшие практики

### 1. Явное указание настроек

```java
// ✅ Хорошо
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED,
    timeout = 30,
    readOnly = false
)
public void method() {
    // Реализация
}

// ❌ Плохо
@Transactional
public void method() {
    // Неясные настройки по умолчанию
}
```

### 2. Документирование намерений

```java
/**
 * Создает пользователя в транзакции.
 * Использует REQUIRED propagation для участия в существующей транзакции
 * или создания новой.
 */
@Transactional(propagation = Propagation.REQUIRED)
public User createUser(User user) {
    return userRepository.save(user);
}
```

### 3. Обработка исключений

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    rollbackFor = {SQLException.class, DataAccessException.class},
    noRollbackFor = {BusinessException.class}
)
public void businessOperation() {
    // Бизнес-логика с правильной обработкой исключений
}
```

### 4. Тестирование

```java
@ExtendWith(SpringExtension.class)
@Transactional
@Rollback
class UserServiceTest {
    
    @Test
    void testCreateUser() {
        // Тест с автоматическим откатом транзакции
    }
}
```

## Отладка и мониторинг

### Включение логирования

```properties
# application.properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.orm.jpa=DEBUG
```

### Проверка настроек

```java
@Component
public class TransactionChecker {
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkTransactionSettings(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        // Проверяем настройки @Transactional
        // Анализируем propagation, isolation и другие параметры
    }
} 