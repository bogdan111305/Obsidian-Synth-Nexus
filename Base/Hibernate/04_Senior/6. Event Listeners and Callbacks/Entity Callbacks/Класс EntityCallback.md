# Класс EntityCallback в JPA

**EntityCallback** — это функциональный интерфейс в JPA, который предоставляет более гибкий способ обработки событий жизненного цикла сущностей по сравнению с аннотациями. Он был введен в JPA 2.2 и предоставляет типобезопасный подход к обработке событий.

## 1. Обзор EntityCallback

### 1.1. Что такое EntityCallback?

`EntityCallback` — это функциональный интерфейс, который позволяет обрабатывать события жизненного цикла сущностей с использованием лямбда-выражений или ссылок на методы. Это более современный и гибкий подход по сравнению с аннотациями.

### 1.2. Преимущества EntityCallback

- **Типобезопасность**: Компилятор проверяет типы на этапе компиляции
- **Гибкость**: Можно использовать лямбда-выражения и ссылки на методы
- **Переиспользование**: Callbacks можно переиспользовать для разных сущностей
- **Тестируемость**: Легче тестировать и мокать
- **Инъекция зависимостей**: Можно инжектировать Spring бины

## 2. Типы EntityCallback

JPA предоставляет несколько типов EntityCallback для различных событий жизненного цикла:

### 2.1. BeforeConvertCallback

**Описание**: Выполняется перед конвертацией сущности в документ (для NoSQL) или перед сохранением.

**Сигнатура**:
```java
@FunctionalInterface
public interface BeforeConvertCallback<T> {
    T onBeforeConvert(T entity, String collection);
}
```

**Пример использования**:
```java
@Component
public class UserBeforeConvertCallback implements BeforeConvertCallback<User> {
    
    @Override
    public User onBeforeConvert(User user, String collection) {
        if (user.getCreatedAt() == null) {
            user.setCreatedAt(LocalDateTime.now());
        }
        user.setUpdatedAt(LocalDateTime.now());
        System.out.println("BeforeConvert: User will be converted: " + user.getUsername());
        return user;
    }
}
```

### 2.2. BeforeSaveCallback

**Описание**: Выполняется перед сохранением сущности в базу данных.

**Сигнатура**:
```java
@FunctionalInterface
public interface BeforeSaveCallback<T> {
    T onBeforeSave(T entity, String collection);
}
```

**Пример использования**:
```java
@Component
public class ProductBeforeSaveCallback implements BeforeSaveCallback<Product> {
    
    @Override
    public Product onBeforeSave(Product product, String collection) {
        if (product.getCreatedAt() == null) {
            product.setCreatedAt(LocalDateTime.now());
        }
        product.setUpdatedAt(LocalDateTime.now());
        
        // Валидация
        if (product.getPrice().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Product price must be positive");
        }
        
        System.out.println("BeforeSave: Product will be saved: " + product.getName());
        return product;
    }
}
```

### 2.3. AfterSaveCallback

**Описание**: Выполняется после сохранения сущности в базу данных.

**Сигнатура**:
```java
@FunctionalInterface
public interface AfterSaveCallback<T> {
    T onAfterSave(T entity, String collection);
}
```

**Пример использования**:
```java
@Component
public class OrderAfterSaveCallback implements AfterSaveCallback<Order> {
    
    @Override
    public Order onAfterSave(Order order, String collection) {
        System.out.println("AfterSave: Order saved with ID: " + order.getId());
        
        // Можно отправить уведомление
        sendOrderNotification(order);
        
        return order;
    }
    
    private void sendOrderNotification(Order order) {
        // Логика отправки уведомления
        System.out.println("Sending notification for order: " + order.getOrderNumber());
    }
}
```

### 2.4. BeforeDeleteCallback

**Описание**: Выполняется перед удалением сущности из базы данных.

**Сигнатура**:
```java
@FunctionalInterface
public interface BeforeDeleteCallback<T> {
    T onBeforeDelete(T entity, String collection);
}
```

**Пример использования**:
```java
@Component
public class DocumentBeforeDeleteCallback implements BeforeDeleteCallback<Document> {
    
    @Override
    public Document onBeforeDelete(Document document, String collection) {
        System.out.println("BeforeDelete: Document will be deleted: " + document.getFileName());
        
        // Создание резервной копии
        createBackup(document);
        
        return document;
    }
    
    private void createBackup(Document document) {
        // Логика создания резервной копии
        System.out.println("Creating backup for document: " + document.getFileName());
    }
}
```

### 2.5. AfterDeleteCallback

**Описание**: Выполняется после удаления сущности из базы данных.

**Сигнатура**:
```java
@FunctionalInterface
public interface AfterDeleteCallback<T> {
    T onAfterDelete(T entity, String collection);
}
```

**Пример использования**:
```java
@Component
public class FileAfterDeleteCallback implements AfterDeleteCallback<File> {
    
    @Override
    public File onAfterDelete(File file, String collection) {
        System.out.println("AfterDelete: File deleted: " + file.getFileName());
        
        // Удаление физического файла
        deletePhysicalFile(file);
        
        return file;
    }
    
    private void deletePhysicalFile(File file) {
        try {
            java.nio.file.Files.deleteIfExists(
                java.nio.file.Paths.get(file.getFilePath())
            );
            System.out.println("Physical file deleted: " + file.getFilePath());
        } catch (IOException e) {
            System.err.println("Error deleting physical file: " + e.getMessage());
        }
    }
}
```

### 2.6. AfterLoadCallback

**Описание**: Выполняется после загрузки сущности из базы данных.

**Сигнатура**:
```java
@FunctionalInterface
public interface AfterLoadCallback<T> {
    T onAfterLoad(T entity, String collection);
}
```

**Пример использования**:
```java
@Component
public class UserAfterLoadCallback implements AfterLoadCallback<User> {
    
    @Override
    public User onAfterLoad(User user, String collection) {
        System.out.println("AfterLoad: User loaded: " + user.getUsername());
        
        // Вычисление производных полей
        user.setFullName(user.getFirstName() + " " + user.getLastName());
        
        // Обновление статистики
        updateUserStatistics(user);
        
        return user;
    }
    
    private void updateUserStatistics(User user) {
        // Логика обновления статистики
        System.out.println("Updating statistics for user: " + user.getUsername());
    }
}
```

## 3. Регистрация EntityCallback

### 3.1. Автоматическая регистрация

Spring Data автоматически обнаруживает и регистрирует бины, реализующие EntityCallback интерфейсы:

```java
@Configuration
public class EntityCallbackConfig {
    
    @Bean
    public BeforeSaveCallback<User> userBeforeSaveCallback() {
        return (user, collection) -> {
            if (user.getCreatedAt() == null) {
                user.setCreatedAt(LocalDateTime.now());
            }
            user.setUpdatedAt(LocalDateTime.now());
            return user;
        };
    }
    
    @Bean
    public AfterSaveCallback<Order> orderAfterSaveCallback() {
        return (order, collection) -> {
            System.out.println("Order saved: " + order.getOrderNumber());
            return order;
        };
    }
}
```

### 3.2. Программная регистрация

Можно регистрировать callbacks программно:

```java
@Configuration
public class CustomEntityCallbackConfig {
    
    @Bean
    public MongoTemplate mongoTemplate(MongoDatabaseFactory mongoDatabaseFactory) {
        MongoTemplate template = new MongoTemplate(mongoDatabaseFactory);
        
        // Регистрация BeforeSaveCallback
        template.setEntityCallbacks(new EntityCallbacks() {
            @Override
            public <T> T callback(BeforeSaveCallback<T> callback, T entity, String collection) {
                return callback.onBeforeSave(entity, collection);
            }
        });
        
        return template;
    }
}
```

## 4. Практические примеры

### 4.1. Аудит с EntityCallback

```java
@Entity
@Table(name = "audited_entities")
public class AuditedEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String createdBy;
    private String updatedBy;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
    
    public String getCreatedBy() { return createdBy; }
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
    
    public String getUpdatedBy() { return updatedBy; }
    public void setUpdatedBy(String updatedBy) { this.updatedBy = updatedBy; }
}

@Component
public class AuditedEntityBeforeSaveCallback implements BeforeSaveCallback<AuditedEntity> {
    
    @Override
    public AuditedEntity onBeforeSave(AuditedEntity entity, String collection) {
        if (entity.getCreatedAt() == null) {
            entity.setCreatedAt(LocalDateTime.now());
            entity.setCreatedBy(getCurrentUser());
        }
        entity.setUpdatedAt(LocalDateTime.now());
        entity.setUpdatedBy(getCurrentUser());
        
        System.out.println("BeforeSave: AuditedEntity will be saved: " + entity.getName());
        return entity;
    }
    
    private String getCurrentUser() {
        try {
            return SecurityContextHolder.getContext().getAuthentication().getName();
        } catch (Exception e) {
            return "system";
        }
    }
}
```

### 4.2. Валидация с EntityCallback

```java
@Entity
@Table(name = "validated_entities")
public class ValidatedEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private BigDecimal price;
    
    private int quantity;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
}

@Component
public class ValidatedEntityBeforeSaveCallback implements BeforeSaveCallback<ValidatedEntity> {
    
    @Override
    public ValidatedEntity onBeforeSave(ValidatedEntity entity, String collection) {
        validateEntity(entity);
        System.out.println("BeforeSave: ValidatedEntity validated and ready for save: " + entity.getName());
        return entity;
    }
    
    private void validateEntity(ValidatedEntity entity) {
        if (entity.getName() == null || entity.getName().trim().isEmpty()) {
            throw new IllegalArgumentException("Entity name cannot be empty");
        }
        
        if (entity.getPrice() == null || entity.getPrice().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Entity price must be positive");
        }
        
        if (entity.getQuantity() < 0) {
            throw new IllegalArgumentException("Entity quantity cannot be negative");
        }
    }
}
```

### 4.3. Кэширование с EntityCallback

```java
@Entity
@Table(name = "cached_entities")
public class CachedEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String data;
    
    @Transient
    private String cachedData;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getData() { return data; }
    public void setData(String data) { this.data = data; }
    
    public String getCachedData() { return cachedData; }
    public void setCachedData(String cachedData) { this.cachedData = cachedData; }
}

@Component
public class CachedEntityAfterLoadCallback implements AfterLoadCallback<CachedEntity> {
    
    @Override
    public CachedEntity onAfterLoad(CachedEntity entity, String collection) {
        // Кэширование обработанных данных
        entity.setCachedData(processData(entity.getData()));
        System.out.println("AfterLoad: CachedEntity loaded and cached: " + entity.getName());
        return entity;
    }
    
    private String processData(String data) {
        // Логика обработки данных
        return data != null ? data.toUpperCase() : "";
    }
}
```

## 5. Комбинирование с аннотациями

EntityCallback можно комбинировать с аннотациями для более сложной логики:

```java
@Entity
@Table(name = "combined_entities")
public class CombinedEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // Аннотации для базовой логики
    @PrePersist
    public void prePersist() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
        System.out.println("PrePersist: CombinedEntity will be saved");
    }
    
    @PreUpdate
    public void preUpdate() {
        this.updatedAt = LocalDateTime.now();
        System.out.println("PreUpdate: CombinedEntity will be updated");
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}

@Component
public class CombinedEntityBeforeSaveCallback implements BeforeSaveCallback<CombinedEntity> {
    
    @Override
    public CombinedEntity onBeforeSave(CombinedEntity entity, String collection) {
        // Дополнительная логика поверх аннотаций
        validateEntity(entity);
        System.out.println("BeforeSave: CombinedEntity validated: " + entity.getName());
        return entity;
    }
    
    private void validateEntity(CombinedEntity entity) {
        if (entity.getName() == null || entity.getName().trim().isEmpty()) {
            throw new IllegalArgumentException("Entity name cannot be empty");
        }
    }
}
```

## 6. Тестирование EntityCallback

### 6.1. Unit тестирование

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
}
```

### 6.2. Integration тестирование

```java
@SpringBootTest
@Transactional
class EntityCallbackIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldExecuteBeforeSaveCallback() {
        // Arrange
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        
        // Act
        User savedUser = userRepository.save(user);
        
        // Assert
        assertNotNull(savedUser.getCreatedAt());
        assertNotNull(savedUser.getUpdatedAt());
        assertNotNull(savedUser.getCreatedBy());
        assertNotNull(savedUser.getUpdatedBy());
    }
}
```

## 7. Вопросы для собеседования

### Базовые вопросы

1. **Что такое EntityCallback в JPA?**
   - Функциональный интерфейс для обработки событий жизненного цикла
   - Типобезопасный подход к обработке событий
   - Альтернатива аннотациям для более гибкой логики

2. **Какие типы EntityCallback доступны?**
   - BeforeConvertCallback, BeforeSaveCallback, AfterSaveCallback
   - BeforeDeleteCallback, AfterDeleteCallback, AfterLoadCallback

3. **Как зарегистрировать EntityCallback?**
   - Автоматически через Spring Data
   - Программно через конфигурацию
   - Как Spring бин

### Продвинутые вопросы

4. **В чем разница между EntityCallback и аннотациями?**
   - EntityCallback: типобезопасность, инъекция зависимостей, переиспользование
   - Аннотации: простота, встроенность в сущность

5. **Можно ли комбинировать EntityCallback с аннотациями?**
   - Да, можно использовать оба подхода одновременно
   - Аннотации выполняются первыми, затем EntityCallback

6. **Как тестировать EntityCallback?**
   - Unit тесты с моками
   - Integration тесты с реальной базой данных
   - Проверка выполнения логики и результатов

### Практические вопросы

7. **Реализуйте EntityCallback для аудита**
```java
@Component
public class AuditBeforeSaveCallback<T> implements BeforeSaveCallback<T> {
    @Override
    public T onBeforeSave(T entity, String collection) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            if (auditable.getCreatedAt() == null) {
                auditable.setCreatedAt(LocalDateTime.now());
                auditable.setCreatedBy(getCurrentUser());
            }
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
        }
        return entity;
    }
}
```

8. **Создайте EntityCallback для валидации**
```java
@Component
public class ValidationBeforeSaveCallback<T> implements BeforeSaveCallback<T> {
    @Override
    public T onBeforeSave(T entity, String collection) {
        if (entity instanceof Validatable) {
            Validatable validatable = (Validatable) entity;
            validatable.validate();
        }
        return entity;
    }
}
```

9. **Реализуйте EntityCallback для кэширования**
```java
@Component
public class CacheAfterLoadCallback<T> implements AfterLoadCallback<T> {
    @Override
    public T onAfterLoad(T entity, String collection) {
        if (entity instanceof Cacheable) {
            Cacheable cacheable = (Cacheable) entity;
            cacheable.initializeCache();
        }
        return entity;
    }
}
``` 