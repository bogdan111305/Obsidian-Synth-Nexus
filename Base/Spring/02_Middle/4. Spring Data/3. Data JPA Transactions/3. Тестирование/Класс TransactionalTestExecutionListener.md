# Класс TransactionalTestExecutionListener

## Обзор

`TransactionalTestExecutionListener` - это ключевой компонент Spring Test Framework, который автоматически управляет транзакциями в тестах. Он является частью инфраструктуры тестирования Spring и обеспечивает изоляцию тестовых данных.

## Основная функциональность

### Автоматическое управление транзакциями

```java
@ExtendWith(SpringExtension.class)
@Transactional
@Rollback
public class UserServiceTest {
    
    @Test
    public void testMethod() {
        // TransactionalTestExecutionListener автоматически:
        // 1. Начинает транзакцию перед тестом
        // 2. Выполняет тест в транзакции
        // 3. Откатывает транзакцию после теста
    }
}
```

## Жизненный цикл

### 1. Инициализация

```java
public class TransactionalTestExecutionListener implements TestExecutionListener {
    
    @Override
    public void beforeTestClass(TestContext testContext) {
        // Инициализация перед классом тестов
    }
    
    @Override
    public void beforeTestMethod(TestContext testContext) {
        // Начало транзакции перед каждым тестом
        TransactionContext transactionContext = getTransactionContext(testContext);
        transactionContext.beginTransaction();
    }
    
    @Override
    public void afterTestMethod(TestContext testContext) {
        // Завершение транзакции после каждого теста
        TransactionContext transactionContext = getTransactionContext(testContext);
        transactionContext.endTransaction();
    }
}
```

### 2. Обработка аннотаций

```java
public class TransactionalTestExecutionListener {
    
    private void processTransactionalAnnotation(TestContext testContext) {
        Method testMethod = testContext.getTestMethod();
        Class<?> testClass = testContext.getTestClass();
        
        // Проверяем аннотации на методе
        Transactional methodAnnotation = testMethod.getAnnotation(Transactional.class);
        if (methodAnnotation != null) {
            applyTransactionSettings(methodAnnotation);
        }
        
        // Проверяем аннотации на классе
        Transactional classAnnotation = testClass.getAnnotation(Transactional.class);
        if (classAnnotation != null) {
            applyTransactionSettings(classAnnotation);
        }
    }
}
```

## Конфигурация транзакций

### Настройка TransactionManager

```java
@Configuration
@EnableTransactionManagement
public class TestTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager() {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource());
        return tm;
    }
    
    @Bean
    public TransactionTemplate transactionTemplate() {
        return new TransactionTemplate(transactionManager());
    }
}
```

### Автоматическое определение

```java
public class TransactionalTestExecutionListener {
    
    private PlatformTransactionManager getTransactionManager(TestContext testContext) {
        ApplicationContext context = testContext.getApplicationContext();
        
        // Ищем TransactionManager в контексте
        Map<String, PlatformTransactionManager> managers = 
            context.getBeansOfType(PlatformTransactionManager.class);
        
        if (managers.size() == 1) {
            return managers.values().iterator().next();
        }
        
        // Если несколько менеджеров, используем по умолчанию
        return context.getBean("transactionManager", PlatformTransactionManager.class);
    }
}
```

## Обработка различных сценариев

### 1. Транзакции с откатом

```java
@Test
@Transactional
@Rollback
public void testWithRollback() {
    // TransactionalTestExecutionListener автоматически:
    // 1. Начинает транзакцию
    // 2. Выполняет тест
    // 3. Откатывает транзакцию
}
```

### 2. Транзакции с коммитом

```java
@Test
@Transactional
@Commit
public void testWithCommit() {
    // TransactionalTestExecutionListener автоматически:
    // 1. Начинает транзакцию
    // 2. Выполняет тест
    // 3. Коммитит транзакцию
}
```

### 3. Без транзакций

```java
@Test
public void testWithoutTransaction() {
    // TransactionalTestExecutionListener не вмешивается
    // Тест выполняется без транзакции
}
```

## Интеграция с различными фреймворками

### JUnit 5

```java
@ExtendWith(SpringExtension.class)
@Transactional
@Rollback
class JUnit5Test {
    
    @Test
    void testMethod() {
        // TransactionalTestExecutionListener работает автоматически
    }
}
```

### JUnit 4

```java
@RunWith(SpringJUnit4ClassRunner.class)
@Transactional
@Rollback
public class JUnit4Test {
    
    @Test
    public void testMethod() {
        // TransactionalTestExecutionListener работает автоматически
    }
}
```

### TestNG

```java
@ContextConfiguration(classes = TestConfig.class)
@Transactional
@Rollback
public class TestNGTest {
    
    @Test
    public void testMethod() {
        // TransactionalTestExecutionListener работает автоматически
    }
}
```

## Кастомизация поведения

### Создание кастомного listener

```java
public class CustomTransactionalTestExecutionListener extends TransactionalTestExecutionListener {
    
    @Override
    public void beforeTestMethod(TestContext testContext) {
        // Дополнительная логика перед тестом
        log.info("Starting custom transaction for test: " + testContext.getTestMethod().getName());
        
        // Вызываем родительский метод
        super.beforeTestMethod(testContext);
    }
    
    @Override
    public void afterTestMethod(TestContext testContext) {
        // Дополнительная логика после теста
        log.info("Ending custom transaction for test: " + testContext.getTestMethod().getName());
        
        // Вызываем родительский метод
        super.afterTestMethod(testContext);
    }
}
```

### Регистрация кастомного listener

```java
@TestExecutionListeners({
    CustomTransactionalTestExecutionListener.class,
    DependencyInjectionTestExecutionListener.class
})
@Transactional
@Rollback
public class CustomTest {
    
    @Test
    public void testMethod() {
        // Использует кастомный listener
    }
}
```

## Обработка исключений

### Автоматический откат при исключениях

```java
@Test
@Transactional
@Rollback
public void testWithException() {
    // TransactionalTestExecutionListener автоматически откатит транзакцию
    // при возникновении исключения
    throw new RuntimeException("Test exception");
}
```

### Настройка исключений для отката

```java
@Test
@Transactional(rollbackFor = Exception.class)
@Rollback
public void testWithCustomRollback() {
    // Транзакция откатится при любом исключении
    throw new Exception("Custom exception");
}
```

## Отладка и мониторинг

### Включение логирования

```properties
# application.properties
logging.level.org.springframework.test.context.transaction=DEBUG
logging.level.org.springframework.transaction=DEBUG
```

### Проверка статуса транзакции

```java
@Test
@Transactional
@Rollback
public void testTransactionStatus() {
    // Получаем информацию о текущей транзакции
    TransactionStatus status = TransactionSynchronizationManager.getCurrentTransactionStatus();
    
    assertThat(status.isNewTransaction()).isTrue();
    assertThat(status.isRollbackOnly()).isFalse();
}
```

## Лучшие практики

### 1. Использование @Rollback по умолчанию

```java
@Transactional
@Rollback
public class TestClass {
    // Все тесты будут откатываться автоматически
}
```

### 2. Изоляция тестов

```java
@Test
@Transactional
@Rollback
public void testIsolated() {
    // Каждый тест должен быть независимым
    // TransactionalTestExecutionListener обеспечивает изоляцию
}
```

### 3. Правильное использование @DirtiesContext

```java
@Test
@DirtiesContext
public void testThatModifiesContext() {
    // Используйте только когда тест модифицирует контекст Spring
    // TransactionalTestExecutionListener не будет управлять транзакциями
}
```

### 4. Оптимизация производительности

```java
@Test
@Transactional
@Rollback
public void testOptimized() {
    // Используйте batch операции для больших объемов данных
    // TransactionalTestExecutionListener эффективно управляет транзакциями
}
```

## Взаимодействие с другими компонентами

### EntityManager

```java
@Test
@Transactional
@Rollback
public void testWithEntityManager() {
    @Autowired
    private EntityManager entityManager;
    
    // EntityManager автоматически участвует в транзакции
    // управляемой TransactionalTestExecutionListener
}
```

### JdbcTemplate

```java
@Test
@Transactional
@Rollback
public void testWithJdbcTemplate() {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // JdbcTemplate автоматически участвует в транзакции
    // управляемой TransactionalTestExecutionListener
}
```

### TransactionTemplate

```java
@Test
@Transactional
@Rollback
public void testWithTransactionTemplate() {
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    // TransactionTemplate может создавать вложенные транзакции
    // в рамках транзакции, управляемой TransactionalTestExecutionListener
}
``` 