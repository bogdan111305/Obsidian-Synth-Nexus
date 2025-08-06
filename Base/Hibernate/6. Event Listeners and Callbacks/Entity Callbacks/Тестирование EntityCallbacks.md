# Тестирование EntityCallbacks

**Тестирование EntityCallbacks** — это важная часть разработки, которая обеспечивает надежность и корректность работы обратных вызовов. В этой статье рассмотрим различные подходы к тестированию EntityCallbacks, включая unit тесты, integration тесты и практические примеры.

## 1. Подходы к тестированию

### 1.1. Unit тестирование

Unit тесты фокусируются на тестировании отдельной логики EntityCallback без зависимости от базы данных или других компонентов.

### 1.2. Integration тестирование

Integration тесты проверяют работу EntityCallback в реальной среде с базой данных и Spring контекстом.

### 1.3. End-to-End тестирование

E2E тесты проверяют полный цикл работы приложения с EntityCallback.

## 2. Unit тестирование EntityCallbacks

### 2.1. Базовый пример

```java
@ExtendWith(MockitoExtension.class)
class UserBeforeSaveCallbackTest {
    
    @InjectMocks
    private UserBeforeSaveCallback callback;
    
    @Test
    void onBeforeSave_ShouldSetTimestamps() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        
        // Act
        User result = callback.onBeforeSave(user, "users");
        
        // Assert
        assertNotNull(result.getCreatedAt());
        assertNotNull(result.getUpdatedAt());
        assertEquals("testuser", result.getUsername());
        assertEquals("test@example.com", result.getEmail());
    }
    
    @Test
    void onBeforeSave_ShouldUpdateExistingTimestamps() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        user.setCreatedAt(LocalDateTime.now().minusDays(1));
        
        // Act
        User result = callback.onBeforeSave(user, "users");
        
        // Assert
        assertNotNull(result.getCreatedAt());
        assertNotNull(result.getUpdatedAt());
        assertTrue(result.getUpdatedAt().isAfter(result.getCreatedAt()));
    }
    
    @Test
    void onBeforeSave_ShouldSetCurrentUser() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        
        // Act
        User result = callback.onBeforeSave(user, "users");
        
        // Assert
        assertNotNull(result.getCreatedBy());
        assertNotNull(result.getUpdatedBy());
        assertEquals(result.getCreatedBy(), result.getUpdatedBy());
    }
}
```

### 2.2. Тестирование с моками

```java
@ExtendWith(MockitoExtension.class)
class ProductBeforeSaveCallbackTest {
    
    @InjectMocks
    private ProductBeforeSaveCallback callback;
    
    @Mock
    private ValidationService validationService;
    
    @Test
    void onBeforeSave_ShouldValidateProduct() {
        // Arrange
        Product product = new Product();
        product.setName("Test Product");
        product.setPrice(BigDecimal.valueOf(100));
        
        when(validationService.isValid(product)).thenReturn(true);
        
        // Act
        Product result = callback.onBeforeSave(product, "products");
        
        // Assert
        verify(validationService).isValid(product);
        assertNotNull(result.getUpdatedAt());
    }
    
    @Test
    void onBeforeSave_ShouldThrowException_WhenValidationFails() {
        // Arrange
        Product product = new Product();
        product.setName("Test Product");
        product.setPrice(BigDecimal.valueOf(-100)); // Invalid price
        
        when(validationService.isValid(product)).thenReturn(false);
        
        // Act & Assert
        assertThrows(IllegalArgumentException.class, () -> {
            callback.onBeforeSave(product, "products");
        });
    }
}
```

### 2.3. Тестирование с параметризованными тестами

```java
@ExtendWith(MockitoExtension.class)
class AuditedEntityBeforeSaveCallbackTest {
    
    @InjectMocks
    private AuditedEntityBeforeSaveCallback callback;
    
    @ParameterizedTest
    @ValueSource(strings = {"user1", "user2", "user3"})
    void onBeforeSave_ShouldSetAuditFields(String username) {
        // Arrange
        AuditedEntity entity = new AuditedEntity();
        entity.setName(username);
        
        // Act
        AuditedEntity result = callback.onBeforeSave(entity, "audited_entities");
        
        // Assert
        assertNotNull(result.getCreatedAt());
        assertNotNull(result.getUpdatedAt());
        assertNotNull(result.getCreatedBy());
        assertNotNull(result.getUpdatedBy());
        assertEquals(username, result.getName());
    }
    
    @ParameterizedTest
    @CsvSource({
        "Product1, 100.00",
        "Product2, 200.00",
        "Product3, 300.00"
    })
    void onBeforeSave_ShouldHandleDifferentEntities(String name, BigDecimal price) {
        // Arrange
        AuditedEntity entity = new AuditedEntity();
        entity.setName(name);
        
        // Act
        AuditedEntity result = callback.onBeforeSave(entity, "audited_entities");
        
        // Assert
        assertNotNull(result.getCreatedAt());
        assertNotNull(result.getUpdatedAt());
        assertEquals(name, result.getName());
    }
}
```

## 3. Integration тестирование

### 3.1. Тестирование с реальной базой данных

```java
@SpringBootTest
@Transactional
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class EntityCallbackIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    void shouldExecuteBeforeSaveCallback_WhenSavingUser() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        
        // Act
        User savedUser = userRepository.save(user);
        
        // Assert
        assertNotNull(savedUser.getId());
        assertNotNull(savedUser.getCreatedAt());
        assertNotNull(savedUser.getUpdatedAt());
        assertNotNull(savedUser.getCreatedBy());
        assertNotNull(savedUser.getUpdatedBy());
        assertEquals("testuser", savedUser.getUsername());
    }
    
    @Test
    void shouldExecuteAfterSaveCallback_WhenSavingOrder() {
        // Arrange
        Order order = new Order();
        order.setOrderNumber("ORD-001");
        order.setTotal(BigDecimal.valueOf(100));
        
        // Act
        Order savedOrder = orderRepository.save(order);
        
        // Assert
        assertNotNull(savedOrder.getId());
        // Проверяем, что после сохранения была отправлена нотификация
        // (это можно проверить через мок или логи)
    }
    
    @Test
    void shouldExecuteBeforeDeleteCallback_WhenDeletingDocument() {
        // Arrange
        Document document = new Document();
        document.setFileName("test.pdf");
        document.setFilePath("/path/to/test.pdf");
        
        Document savedDocument = documentRepository.save(document);
        
        // Act
        documentRepository.delete(savedDocument);
        
        // Assert
        // Проверяем, что была создана резервная копия
        // (это можно проверить через мок или файловую систему)
    }
}
```

### 3.2. Тестирование с тестовой конфигурацией

```java
@TestConfiguration
class TestEntityCallbackConfig {
    
    @Bean
    public BeforeSaveCallback<User> testUserBeforeSaveCallback() {
        return (user, collection) -> {
            user.setCreatedAt(LocalDateTime.now());
            user.setUpdatedAt(LocalDateTime.now());
            user.setCreatedBy("test-user");
            user.setUpdatedBy("test-user");
            return user;
        };
    }
    
    @Bean
    public AfterSaveCallback<Order> testOrderAfterSaveCallback() {
        return (order, collection) -> {
            System.out.println("Test: Order saved: " + order.getOrderNumber());
            return order;
        };
    }
}

@SpringBootTest
@Import(TestEntityCallbackConfig.class)
@Transactional
class EntityCallbackWithTestConfigTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldUseTestCallback() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        
        // Act
        User savedUser = userRepository.save(user);
        
        // Assert
        assertEquals("test-user", savedUser.getCreatedBy());
        assertEquals("test-user", savedUser.getUpdatedBy());
    }
}
```

## 4. Тестирование с моками и шпионами

### 4.1. Использование @SpyBean

```java
@SpringBootTest
@Transactional
class EntityCallbackWithSpyTest {
    
    @SpyBean
    private UserBeforeSaveCallback userBeforeSaveCallback;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldCallBeforeSaveCallback() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        
        // Act
        userRepository.save(user);
        
        // Assert
        verify(userBeforeSaveCallback, times(1))
            .onBeforeSave(any(User.class), eq("users"));
    }
    
    @Test
    void shouldModifyUserInCallback() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        
        doAnswer(invocation -> {
            User u = invocation.getArgument(0);
            u.setCreatedAt(LocalDateTime.now());
            u.setUpdatedAt(LocalDateTime.now());
            return u;
        }).when(userBeforeSaveCallback).onBeforeSave(any(User.class), anyString());
        
        // Act
        User savedUser = userRepository.save(user);
        
        // Assert
        assertNotNull(savedUser.getCreatedAt());
        assertNotNull(savedUser.getUpdatedAt());
    }
}
```

### 4.2. Использование @MockBean

```java
@SpringBootTest
@Transactional
class EntityCallbackWithMockTest {
    
    @MockBean
    private NotificationService notificationService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void shouldSendNotification_WhenOrderSaved() {
        // Arrange
        Order order = new Order();
        order.setOrderNumber("ORD-001");
        order.setTotal(BigDecimal.valueOf(100));
        
        // Act
        orderRepository.save(order);
        
        // Assert
        verify(notificationService, times(1))
            .sendOrderNotification(any(Order.class));
    }
    
    @Test
    void shouldNotSendNotification_WhenOrderUpdateFails() {
        // Arrange
        Order order = new Order();
        order.setOrderNumber("ORD-001");
        order.setTotal(BigDecimal.valueOf(100));
        
        doThrow(new RuntimeException("Notification failed"))
            .when(notificationService).sendOrderNotification(any(Order.class));
        
        // Act & Assert
        assertThrows(RuntimeException.class, () -> {
            orderRepository.save(order);
        });
    }
}
```

## 5. Тестирование исключений

### 5.1. Тестирование валидации

```java
@ExtendWith(MockitoExtension.class)
class ValidationBeforeSaveCallbackTest {
    
    @InjectMocks
    private ValidationBeforeSaveCallback callback;
    
    @Test
    void shouldThrowException_WhenNameIsEmpty() {
        // Arrange
        ValidatedEntity entity = new ValidatedEntity();
        entity.setName("");
        entity.setPrice(BigDecimal.valueOf(100));
        
        // Act & Assert
        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> callback.onBeforeSave(entity, "validated_entities")
        );
        
        assertEquals("Entity name cannot be empty", exception.getMessage());
    }
    
    @Test
    void shouldThrowException_WhenPriceIsNegative() {
        // Arrange
        ValidatedEntity entity = new ValidatedEntity();
        entity.setName("Test Product");
        entity.setPrice(BigDecimal.valueOf(-100));
        
        // Act & Assert
        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> callback.onBeforeSave(entity, "validated_entities")
        );
        
        assertEquals("Entity price must be positive", exception.getMessage());
    }
    
    @Test
    void shouldNotThrowException_WhenEntityIsValid() {
        // Arrange
        ValidatedEntity entity = new ValidatedEntity();
        entity.setName("Test Product");
        entity.setPrice(BigDecimal.valueOf(100));
        
        // Act & Assert
        assertDoesNotThrow(() -> {
            callback.onBeforeSave(entity, "validated_entities");
        });
    }
}
```

### 5.2. Тестирование обработки ошибок

```java
@SpringBootTest
@Transactional
class EntityCallbackErrorHandlingTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldHandleCallbackException_Gracefully() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        
        // Act & Assert
        // Если callback выбрасывает исключение, транзакция должна откатиться
        assertThrows(Exception.class, () -> {
            userRepository.save(user);
        });
    }
    
    @Test
    void shouldContinueExecution_WhenCallbackThrowsNonCriticalException() {
        // Arrange
        Product product = new Product();
        product.setName("Test Product");
        product.setPrice(BigDecimal.valueOf(100));
        
        // Act
        Product savedProduct = productRepository.save(product);
        
        // Assert
        assertNotNull(savedProduct.getId());
        // Проверяем, что продукт сохранился, несмотря на ошибку в callback
    }
}
```

## 6. Тестирование производительности

### 6.1. Тестирование времени выполнения

```java
@SpringBootTest
@Transactional
class EntityCallbackPerformanceTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldExecuteCallback_WithinReasonableTime() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        
        // Act
        long startTime = System.currentTimeMillis();
        User savedUser = userRepository.save(user);
        long endTime = System.currentTimeMillis();
        
        // Assert
        long executionTime = endTime - startTime;
        assertTrue(executionTime < 1000, "Callback execution took too long: " + executionTime + "ms");
        assertNotNull(savedUser.getId());
    }
    
    @Test
    void shouldHandleBulkOperations_Efficiently() {
        // Arrange
        List<User> users = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            User user = new User();
            user.setUsername("user" + i);
            user.setEmail("user" + i + "@example.com");
            users.add(user);
        }
        
        // Act
        long startTime = System.currentTimeMillis();
        List<User> savedUsers = userRepository.saveAll(users);
        long endTime = System.currentTimeMillis();
        
        // Assert
        long executionTime = endTime - startTime;
        assertTrue(executionTime < 5000, "Bulk operation took too long: " + executionTime + "ms");
        assertEquals(100, savedUsers.size());
    }
}
```

## 7. Тестирование с различными сценариями

### 7.1. Тестирование с разными типами сущностей

```java
@SpringBootTest
@Transactional
class MultiEntityCallbackTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void shouldExecuteCallbacks_ForDifferentEntityTypes() {
        // Test User callback
        User user = new User();
        user.setUsername("testuser");
        User savedUser = userRepository.save(user);
        assertNotNull(savedUser.getCreatedAt());
        
        // Test Product callback
        Product product = new Product();
        product.setName("Test Product");
        product.setPrice(BigDecimal.valueOf(100));
        Product savedProduct = productRepository.save(product);
        assertNotNull(savedProduct.getUpdatedAt());
        
        // Test Order callback
        Order order = new Order();
        order.setOrderNumber("ORD-001");
        order.setTotal(BigDecimal.valueOf(100));
        Order savedOrder = orderRepository.save(order);
        assertNotNull(savedOrder.getId());
    }
}
```

### 7.2. Тестирование с транзакциями

```java
@SpringBootTest
@Transactional
class EntityCallbackTransactionTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldRollbackTransaction_WhenCallbackThrowsException() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        
        // Act & Assert
        assertThrows(Exception.class, () -> {
            userRepository.save(user);
        });
        
        // Проверяем, что пользователь не сохранился
        assertFalse(userRepository.findByUsername("testuser").isPresent());
    }
    
    @Test
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void shouldCommitTransaction_WhenCallbackSucceeds() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        
        // Act
        User savedUser = userRepository.save(user);
        
        // Assert
        assertNotNull(savedUser.getId());
        assertTrue(userRepository.findByUsername("testuser").isPresent());
    }
}
```

## 8. Вопросы для собеседования

### Базовые вопросы

1. **Как тестировать EntityCallback?**
   - Unit тесты с моками
   - Integration тесты с реальной базой данных
   - Проверка выполнения логики и результатов

2. **Какие инструменты используются для тестирования EntityCallback?**
   - JUnit 5
   - Mockito для моков
   - Spring Boot Test для integration тестов
   - @Transactional для управления транзакциями

3. **Как тестировать исключения в EntityCallback?**
   - Использование assertThrows
   - Проверка сообщений об ошибках
   - Тестирование различных сценариев валидации

### Продвинутые вопросы

4. **Как тестировать производительность EntityCallback?**
   - Измерение времени выполнения
   - Тестирование с большими объемами данных
   - Профилирование и оптимизация

5. **Как тестировать EntityCallback с транзакциями?**
   - Использование @Transactional
   - Тестирование отката транзакций
   - Проверка изоляции транзакций

6. **Как тестировать EntityCallback с внешними зависимостями?**
   - Использование @MockBean
   - Мокирование внешних сервисов
   - Тестирование интеграции

### Практические вопросы

7. **Напишите unit тест для EntityCallback**
```java
@ExtendWith(MockitoExtension.class)
class CustomEntityCallbackTest {
    @InjectMocks
    private CustomEntityCallback callback;
    
    @Test
    void shouldProcessEntity() {
        // Arrange
        TestEntity entity = new TestEntity();
        entity.setName("test");
        
        // Act
        TestEntity result = callback.onBeforeSave(entity, "test_entities");
        
        // Assert
        assertNotNull(result.getProcessedAt());
        assertEquals("test", result.getName());
    }
}
```

8. **Создайте integration тест для EntityCallback**
```java
@SpringBootTest
@Transactional
class EntityCallbackIntegrationTest {
    @Autowired
    private TestRepository repository;
    
    @Test
    void shouldExecuteCallback_WhenSavingEntity() {
        // Arrange
        TestEntity entity = new TestEntity();
        entity.setName("test");
        
        // Act
        TestEntity saved = repository.save(entity);
        
        // Assert
        assertNotNull(saved.getId());
        assertNotNull(saved.getProcessedAt());
    }
}
```

9. **Реализуйте тест для валидации в EntityCallback**
```java
@Test
void shouldValidateEntity_BeforeSave() {
    // Arrange
    ValidatedEntity entity = new ValidatedEntity();
    entity.setName(""); // Invalid
    
    // Act & Assert
    assertThrows(IllegalArgumentException.class, () -> {
        callback.onBeforeSave(entity, "validated_entities");
    });
}
``` 