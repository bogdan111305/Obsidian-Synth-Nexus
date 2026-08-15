# Аннотации @Commit и @Rollback

## Обзор

Аннотации `@Commit` и `@Rollback` в Spring Test Framework позволяют контролировать поведение транзакций в тестах. Они определяют, будет ли транзакция зафиксирована или откачена после выполнения теста.

## @Rollback

### Основное назначение

Аннотация `@Rollback` указывает, что транзакция должна быть откачена после завершения теста. Это поведение по умолчанию для тестов с аннотацией `@Transactional`.

### Базовое использование

```java
@Test
@Transactional
@Rollback
public void testWithRollback() {
    // Тест выполняется в транзакции
    // После завершения транзакция откатывается
    User user = new User("John", "Doe");
    userRepository.save(user);
    
    // Изменения не сохранятся в базе данных
}
```

### Явное указание отката

```java
@Test
@Transactional
@Rollback(true) // Явно указываем откат
public void testExplicitRollback() {
    // Транзакция будет откачена
}
```

### Откат по умолчанию

```java
@Transactional
@Rollback // По умолчанию true
public class TestClass {
    
    @Test
    public void testMethod() {
        // Транзакция откатится автоматически
    }
}
```

## @Commit

### Основное назначение

Аннотация `@Commit` указывает, что транзакция должна быть зафиксирована после завершения теста. Это полезно для тестов, которые должны проверить реальное сохранение данных.

### Базовое использование

```java
@Test
@Transactional
@Commit
public void testWithCommit() {
    // Тест выполняется в транзакции
    // После завершения транзакция коммитится
    User user = new User("John", "Doe");
    userRepository.save(user);
    
    // Изменения сохранятся в базе данных
}
```

### Явное указание коммита

```java
@Test
@Transactional
@Commit(true) // Явно указываем коммит
public void testExplicitCommit() {
    // Транзакция будет зафиксирована
}
```

## Приоритет аннотаций

### Иерархия приоритетов

```java
@Test
@Transactional
@Rollback(false) // Эквивалентно @Commit
public void testWithRollbackFalse() {
    // Транзакция будет зафиксирована
}

@Test
@Transactional
@Commit
@Rollback(true) // @Rollback имеет приоритет над @Commit
public void testWithConflictingAnnotations() {
    // Транзакция будет откачена (приоритет у @Rollback)
}
```

### Настройка на уровне класса

```java
@Transactional
@Rollback // Применяется ко всем тестам в классе
public class TestClass {
    
    @Test
    public void test1() {
        // Транзакция откатится
    }
    
    @Test
    @Commit // Переопределяет настройку класса
    public void test2() {
        // Транзакция зафиксируется
    }
}
```

## Практические сценарии

### 1. Тестирование с откатом (рекомендуется)

```java
@Test
@Transactional
@Rollback
public void testUserCreation() {
    // Given
    User user = new User("John", "Doe");
    
    // When
    User savedUser = userRepository.save(user);
    
    // Then
    assertThat(savedUser.getId()).isNotNull();
    assertThat(savedUser.getName()).isEqualTo("John");
    
    // Транзакция откатится, данные не сохранятся в БД
}
```

### 2. Тестирование с коммитом

```java
@Test
@Transactional
@Commit
public void testUserPersistence() {
    // Given
    User user = new User("John", "Doe");
    
    // When
    User savedUser = userRepository.save(user);
    
    // Then
    assertThat(savedUser.getId()).isNotNull();
    
    // Проверяем, что данные действительно сохранены
    User foundUser = userRepository.findById(savedUser.getId()).orElse(null);
    assertThat(foundUser).isNotNull();
    assertThat(foundUser.getName()).isEqualTo("John");
    
    // Транзакция зафиксируется, данные сохранятся в БД
}
```

### 3. Тестирование интеграции

```java
@Test
@Transactional
@Commit
public void testIntegrationFlow() {
    // Given
    User user = new User("John", "Doe");
    userRepository.save(user);
    
    // When
    Order order = new Order(user, BigDecimal.valueOf(100));
    orderRepository.save(order);
    
    // Then
    assertThat(order.getId()).isNotNull();
    assertThat(order.getUser()).isEqualTo(user);
    
    // Проверяем связь между сущностями
    User foundUser = userRepository.findById(user.getId()).orElse(null);
    assertThat(foundUser.getOrders()).hasSize(1);
    
    // Данные сохранятся для проверки связей
}
```

## Взаимодействие с другими аннотациями

### @DirtiesContext

```java
@Test
@Transactional
@Commit
@DirtiesContext // Пересоздает контекст после теста
public void testWithDirtiesContext() {
    // Тест с коммитом и пересозданием контекста
}
```

### @TestExecutionListeners

```java
@TestExecutionListeners({
    TransactionalTestExecutionListener.class,
    DependencyInjectionTestExecutionListener.class
})
@Transactional
@Rollback
public class CustomTest {
    
    @Test
    public void testMethod() {
        // Использует кастомные listeners с откатом
    }
}
```

## Настройка глобального поведения

### Конфигурация по умолчанию

```java
@Configuration
@EnableTransactionManagement
public class TestConfig {
    
    @Bean
    public TransactionTemplate transactionTemplate() {
        TransactionTemplate template = new TransactionTemplate(transactionManager());
        template.setRollbackOnCommitFailure(true);
        return template;
    }
}
```

### Настройка в application.properties

```properties
# application-test.properties
spring.test.transaction.rollback.enabled=true
spring.test.transaction.rollback.default=true
```

## Обработка исключений

### Автоматический откат при исключениях

```java
@Test
@Transactional
@Rollback
public void testWithException() {
    // Даже с @Rollback, исключение вызовет откат
    throw new RuntimeException("Test exception");
}
```

### Настройка исключений для отката

```java
@Test
@Transactional(rollbackFor = Exception.class)
@Commit
public void testWithCustomRollback() {
    // @Commit будет проигнорирован при исключении
    throw new Exception("Custom exception");
}
```

## Отладка и мониторинг

### Логирование транзакций

```properties
# application.properties
logging.level.org.springframework.test.context.transaction=DEBUG
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.orm.jpa=DEBUG
```

### Проверка статуса транзакции

```java
@Test
@Transactional
@Rollback
public void testTransactionStatus() {
    TransactionStatus status = TransactionSynchronizationManager.getCurrentTransactionStatus();
    
    assertThat(status.isNewTransaction()).isTrue();
    assertThat(status.isRollbackOnly()).isFalse();
    
    // После теста транзакция будет откачена
}
```

## Лучшие практики

### 1. Используйте @Rollback по умолчанию

```java
@Transactional
@Rollback // Рекомендуется для большинства тестов
public class TestClass {
    
    @Test
    public void testMethod() {
        // Изоляция тестов, быстрая очистка данных
    }
}
```

### 2. Используйте @Commit только при необходимости

```java
@Test
@Transactional
@Commit // Только для тестов, требующих реального сохранения
public void testPersistence() {
    // Тесты, которые должны проверить реальное сохранение
}
```

### 3. Избегайте смешивания подходов

```java
// ❌ Плохо - смешивание подходов
@Test
@Transactional
@Commit
public void test1() { /* коммит */ }

@Test
@Transactional
@Rollback
public void test2() { /* откат */ }

// ✅ Хорошо - консистентный подход
@Transactional
@Rollback
public class TestClass {
    
    @Test
    public void test1() { /* откат */ }
    
    @Test
    public void test2() { /* откат */ }
}
```

### 4. Документирование намерений

```java
/**
 * Тест проверяет реальное сохранение данных в БД.
 * Использует @Commit для фиксации транзакции.
 */
@Test
@Transactional
@Commit
public void testRealPersistence() {
    // Реализация теста
}
```

## Производительность

### Влияние на скорость тестов

```java
// Быстрые тесты с откатом
@Test
@Transactional
@Rollback
public void fastTest() {
    // Быстрое выполнение, быстрая очистка
}

// Медленные тесты с коммитом
@Test
@Transactional
@Commit
public void slowTest() {
    // Медленное выполнение, медленная очистка
}
```

### Оптимизация для больших объемов данных

```java
@Test
@Transactional
@Rollback
public void testBulkOperations() {
    // Для больших объемов данных используйте откат
    List<User> users = createLargeUserList();
    userRepository.saveAll(users);
    
    // Откат будет быстрее, чем очистка коммиченных данных
}
``` 