# TestContext - Общая структура работы с TransactionManager

## Обзор

`TestContext` в Spring предоставляет инфраструктуру для управления транзакциями в тестах. Это позволяет изолировать тесты друг от друга и обеспечивать консистентность данных между тестовыми запусками.

## Основные компоненты

### 1. TestContext Framework

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = {TestConfig.class})
@Transactional
public class TransactionalTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    public void testMethod() {
        // Тест выполняется в транзакции
    }
}
```

### 2. TransactionalTestExecutionListener

Этот listener автоматически управляет транзакциями в тестах:

```java
@Transactional
@Rollback
public class UserServiceTest {
    
    @Test
    public void testCreateUser() {
        // Метод выполняется в транзакции
        // После завершения транзакция откатывается
    }
}
```

## Конфигурация TransactionManager

### Автоматическая конфигурация

Spring Boot автоматически настраивает `TransactionManager`:

```java
@Configuration
@EnableTransactionManagement
public class TestConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

### Ручная конфигурация

```java
@Configuration
public class ManualTransactionConfig {
    
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource);
        return tm;
    }
}
```

## Типы TransactionManager

### 1. DataSourceTransactionManager
Для простых JDBC транзакций:

```java
@Bean
public PlatformTransactionManager dataSourceTransactionManager() {
    return new DataSourceTransactionManager(dataSource());
}
```

### 2. JpaTransactionManager
Для JPA/Hibernate транзакций:

```java
@Bean
public PlatformTransactionManager jpaTransactionManager() {
    JpaTransactionManager tm = new JpaTransactionManager();
    tm.setEntityManagerFactory(entityManagerFactory());
    return tm;
}
```

### 3. HibernateTransactionManager
Для Hibernate транзакций:

```java
@Bean
public PlatformTransactionManager hibernateTransactionManager() {
    HibernateTransactionManager tm = new HibernateTransactionManager();
    tm.setSessionFactory(sessionFactory());
    return tm;
}
```

## Жизненный цикл транзакций в тестах

### 1. Начало транзакции

```java
@Transactional
public class TransactionLifecycleTest {
    
    @Test
    public void testTransactionLifecycle() {
        // TransactionalTestExecutionListener автоматически:
        // 1. Создает транзакцию перед тестом
        // 2. Настраивает TransactionManager
        // 3. Подготавливает контекст
    }
}
```

### 2. Выполнение теста

```java
@Test
@Transactional
public void testInTransaction() {
    // Тест выполняется в транзакции
    User user = new User("John", "Doe");
    userRepository.save(user);
    
    // Изменения видны в рамках транзакции
    assertThat(userRepository.findById(user.getId())).isPresent();
}
```

### 3. Завершение транзакции

```java
@Test
@Transactional
@Rollback
public void testWithRollback() {
    // После теста транзакция автоматически откатывается
    userRepository.save(new User("John", "Doe"));
    
    // Изменения не сохранятся в базе данных
}
```

## Конфигурация для разных сценариев

### 1. Интеграционные тесты

```java
@ExtendWith(SpringExtension.class)
@DataJpaTest
@Transactional
@Rollback
class UserRepositoryIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private TestEntityManager testEntityManager;
    
    @Test
    void testCreateUser() {
        // Given
        User user = new User("John", "Doe");
        
        // When
        User savedUser = userRepository.save(user);
        
        // Then
        assertThat(savedUser.getId()).isNotNull();
        assertThat(savedUser.getName()).isEqualTo("John");
    }
}
```

### 2. Слойные тесты

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = {ServiceConfig.class})
@Transactional
@Rollback
class UserServiceLayerTest {
    
    @Autowired
    private UserService userService;
    
    @MockBean
    private EmailService emailService;
    
    @Test
    void testCreateUserWithEmail() {
        // Given
        User user = new User("John", "Doe");
        
        // When
        User createdUser = userService.createUser(user);
        
        // Then
        verify(emailService).sendWelcomeEmail(createdUser.getEmail());
    }
}
```

### 3. End-to-End тесты

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Transactional
@Rollback
class UserControllerE2ETest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void testCreateUserViaAPI() {
        // Given
        User user = new User("John", "Doe");
        
        // When
        ResponseEntity<User> response = restTemplate.postForEntity(
            "/api/users", user, User.class);
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getId()).isNotNull();
    }
}
```

## Управление транзакциями в тестах

### 1. @Transactional

```java
@Transactional
public class TransactionalTest {
    
    @Test
    public void testInTransaction() {
        // Тест выполняется в транзакции
    }
}
```

### 2. @Rollback

```java
@Transactional
@Rollback
public class RollbackTest {
    
    @Test
    public void testWithRollback() {
        // Транзакция откатится после теста
    }
}
```

### 3. @Commit

```java
@Transactional
@Commit
public class CommitTest {
    
    @Test
    public void testWithCommit() {
        // Транзакция зафиксируется после теста
    }
}
```

### 4. @DirtiesContext

```java
@DirtiesContext
public class DirtiesContextTest {
    
    @Test
    public void testThatModifiesContext() {
        // Контекст будет пересоздан после теста
    }
}
```

## Конфигурация для разных фреймворков

### JUnit 5
```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = TestConfig.class)
@Transactional
@Rollback
class JUnit5Test {
    
    @Test
    void testMethod() {
        // Тест
    }
}
```

### JUnit 4
```java
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = TestConfig.class)
@Transactional
@Rollback
public class JUnit4Test {
    
    @Test
    public void testMethod() {
        // Тест
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
        // Тест
    }
}
```

## Специализированные аннотации

### 1. @DataJpaTest

```java
@DataJpaTest
@Transactional
@Rollback
class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private TestEntityManager testEntityManager;
    
    @Test
    void testFindByEmail() {
        // Given
        User user = new User("john@example.com", "John");
        testEntityManager.persistAndFlush(user);
        
        // When
        Optional<User> found = userRepository.findByEmail("john@example.com");
        
        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John");
    }
}
```

### 2. @WebMvcTest

```java
@WebMvcTest(UserController.class)
@Transactional
@Rollback
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void testCreateUser() throws Exception {
        // Given
        User user = new User("John", "Doe");
        when(userService.createUser(any(User.class)))
            .thenReturn(user);
        
        // When & Then
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"John\",\"email\":\"john@example.com\"}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

### 3. @SpringBootTest

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Transactional
@Rollback
class UserServiceIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testCreateUserIntegration() {
        // Given
        User user = new User("John", "Doe");
        
        // When
        User createdUser = userService.createUser(user);
        
        // Then
        assertThat(createdUser.getId()).isNotNull();
        assertThat(userRepository.findById(createdUser.getId())).isPresent();
    }
}
```

## Лучшие практики

### 1. Используйте @Rollback по умолчанию
```java
@Transactional
@Rollback
public class TestClass {
    // Все тесты будут откатываться
}
```

### 2. Изолируйте тесты
```java
@Test
@Transactional
@Rollback
public void testMethod() {
    // Каждый тест должен быть независимым
}
```

### 3. Используйте @DirtiesContext при необходимости
```java
@Test
@DirtiesContext
public void testThatModifiesContext() {
    // Тест, который модифицирует контекст
}
```

### 4. Настройте отдельную базу данных для тестов
```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
```

### 5. Используйте TestEntityManager для прямого доступа

```java
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager testEntityManager;
    
    @Test
    void testDirectEntityAccess() {
        // Given
        User user = new User("John", "Doe");
        testEntityManager.persistAndFlush(user);
        
        // When
        User found = testEntityManager.find(User.class, user.getId());
        
        // Then
        assertThat(found).isNotNull();
        assertThat(found.getName()).isEqualTo("John");
    }
}
```

### 6. Настройте профили для тестов

```java
@ActiveProfiles("test")
@DataJpaTest
class ProfileTest {
    
    @Test
    void testWithTestProfile() {
        // Тест с профилем "test"
    }
}
```

## Отладка транзакций

### Включение логирования
```properties
# application.properties
logging.level.org.springframework.transaction=DEBUG
logging.level.org.springframework.orm.jpa=DEBUG
logging.level.org.springframework.test=DEBUG
```

### Проверка статуса транзакции
```java
@Autowired
private TransactionTemplate transactionTemplate;

@Test
public void testTransactionStatus() {
    transactionTemplate.execute(status -> {
        System.out.println("Is new transaction: " + status.isNewTransaction());
        System.out.println("Is rollback only: " + status.isRollbackOnly());
        System.out.println("Transaction name: " + status.getTransactionName());
        return null;
    });
}
```

### Мониторинг транзакций в тестах

```java
@Component
public class TransactionMonitor {
    
    @EventListener(ApplicationReadyEvent.class)
    public void setupTransactionMonitoring(ApplicationReadyEvent event) {
        // Настройка мониторинга транзакций в тестах
    }
}
```

## Обработка исключений

### Настройка rollback
```java
@Test
@Transactional(rollbackFor = Exception.class)
public void testWithRollback() {
    // Транзакция откатится при любом исключении
}
```

### Исключения из rollback
```java
@Test
@Transactional(noRollbackFor = RuntimeException.class)
public void testWithoutRollback() {
    // RuntimeException не вызовет откат
}
```

### Тестирование исключений

```java
@Test
@Transactional
@Rollback
public void testExceptionHandling() {
    // Given
    User user = new User("John", "Doe");
    
    // When & Then
    assertThatThrownBy(() -> userService.createUserWithValidation(user))
        .isInstanceOf(ValidationException.class)
        .hasMessage("User validation failed");
}
```

## Производительность тестов

### 1. Используйте in-memory базы данных

```java
@Configuration
@TestConfiguration
public class TestDatabaseConfig {
    
    @Bean
    public DataSource testDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("schema.sql")
            .addScript("test-data.sql")
            .build();
    }
}
```

### 2. Оптимизируйте конфигурацию

```java
@Configuration
@TestConfiguration
public class OptimizedTestConfig {
    
    @Bean
    public PlatformTransactionManager testTransactionManager() {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(testDataSource());
        tm.setDefaultTimeout(5); // Короткий таймаут для тестов
        return tm;
    }
}
```

### 3. Используйте @Sql для подготовки данных

```java
@Test
@Sql("/test-data.sql")
@Sql(statements = "DELETE FROM users", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
public void testWithSqlScript() {
    // Тест с подготовленными данными
}
``` 
