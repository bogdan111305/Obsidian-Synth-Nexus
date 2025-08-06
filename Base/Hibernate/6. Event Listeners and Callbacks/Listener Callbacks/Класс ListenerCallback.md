# Класс ListenerCallback в JPA

**ListenerCallback** — это механизм в JPA, который позволяет создавать переиспользуемые слушатели событий жизненного цикла сущностей. В отличие от Entity Callbacks, которые привязаны к конкретным сущностям, ListenerCallback может быть применен к множеству сущностей через аннотацию `@EntityListeners`.

## 1. Обзор ListenerCallback

### 1.1. Что такое ListenerCallback?

`ListenerCallback` — это интерфейс, который определяет методы для обработки событий жизненного цикла сущностей. Он позволяет создавать переиспользуемую логику, которая может быть применена к различным сущностям.

### 1.2. Преимущества ListenerCallback

- **Переиспользование**: Один слушатель может использоваться для множества сущностей
- **Разделение ответственности**: Логика слушателя отделена от бизнес-логики сущности
- **Тестируемость**: Легче тестировать слушатели отдельно от сущностей
- **Гибкость**: Можно комбинировать несколько слушателей для одной сущности

## 2. Основные интерфейсы

### 2.1. PrePersistCallback

**Описание**: Выполняется перед сохранением сущности.

**Сигнатура**:
```java
public interface PrePersistCallback<T> {
    void prePersist(T entity);
}
```

**Пример реализации**:
```java
public class AuditPrePersistCallback implements PrePersistCallback<Object> {
    
    @Override
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
        }
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

### 2.2. PostPersistCallback

**Описание**: Выполняется после сохранения сущности.

**Сигнатура**:
```java
public interface PostPersistCallback<T> {
    void postPersist(T entity);
}
```

**Пример реализации**:
```java
public class NotificationPostPersistCallback implements PostPersistCallback<Object> {
    
    @Override
    public void postPersist(Object entity) {
        if (entity instanceof Notifiable) {
            Notifiable notifiable = (Notifiable) entity;
            sendNotification(notifiable);
        }
    }
    
    private void sendNotification(Notifiable entity) {
        // Логика отправки уведомления
        System.out.println("Notification sent for: " + entity.getClass().getSimpleName());
    }
}
```

### 2.3. PreUpdateCallback

**Описание**: Выполняется перед обновлением сущности.

**Сигнатура**:
```java
public interface PreUpdateCallback<T> {
    void preUpdate(T entity);
}
```

**Пример реализации**:
```java
public class AuditPreUpdateCallback implements PreUpdateCallback<Object> {
    
    @Override
    public void preUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
        }
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

### 2.4. PostUpdateCallback

**Описание**: Выполняется после обновления сущности.

**Сигнатура**:
```java
public interface PostUpdateCallback<T> {
    void postUpdate(T entity);
}
```

**Пример реализации**:
```java
public class ChangeTrackingPostUpdateCallback implements PostUpdateCallback<Object> {
    
    @Override
    public void postUpdate(Object entity) {
        if (entity instanceof Trackable) {
            Trackable trackable = (Trackable) entity;
            trackChanges(trackable);
        }
    }
    
    private void trackChanges(Trackable entity) {
        // Логика отслеживания изменений
        System.out.println("Changes tracked for: " + entity.getClass().getSimpleName());
    }
}
```

### 2.5. PreRemoveCallback

**Описание**: Выполняется перед удалением сущности.

**Сигнатура**:
```java
public interface PreRemoveCallback<T> {
    void preRemove(T entity);
}
```

**Пример реализации**:
```java
public class BackupPreRemoveCallback implements PreRemoveCallback<Object> {
    
    @Override
    public void preRemove(Object entity) {
        if (entity instanceof Backupable) {
            Backupable backupable = (Backupable) entity;
            createBackup(backupable);
        }
    }
    
    private void createBackup(Backupable entity) {
        // Логика создания резервной копии
        System.out.println("Backup created for: " + entity.getClass().getSimpleName());
    }
}
```

### 2.6. PostRemoveCallback

**Описание**: Выполняется после удаления сущности.

**Сигнатура**:
```java
public interface PostRemoveCallback<T> {
    void postRemove(T entity);
}
```

**Пример реализации**:
```java
public class CleanupPostRemoveCallback implements PostRemoveCallback<Object> {
    
    @Override
    public void postRemove(Object entity) {
        if (entity instanceof Cleanable) {
            Cleanable cleanable = (Cleanable) entity;
            cleanupResources(cleanable);
        }
    }
    
    private void cleanupResources(Cleanable entity) {
        // Логика очистки ресурсов
        System.out.println("Resources cleaned for: " + entity.getClass().getSimpleName());
    }
}
```

### 2.7. PostLoadCallback

**Описание**: Выполняется после загрузки сущности.

**Сигнатура**:
```java
public interface PostLoadCallback<T> {
    void postLoad(T entity);
}
```

**Пример реализации**:
```java
public class CachePostLoadCallback implements PostLoadCallback<Object> {
    
    @Override
    public void postLoad(Object entity) {
        if (entity instanceof Cacheable) {
            Cacheable cacheable = (Cacheable) entity;
            initializeCache(cacheable);
        }
    }
    
    private void initializeCache(Cacheable entity) {
        // Логика инициализации кэша
        System.out.println("Cache initialized for: " + entity.getClass().getSimpleName());
    }
}
```

## 3. Создание универсального ListenerCallback

### 3.1. Универсальный аудит слушатель

```java
public class UniversalAuditListener {
    
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
            auditable.setUpdatedBy(getCurrentUser());
            
            System.out.println("PrePersist: Auditable entity will be saved");
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
            
            System.out.println("PreUpdate: Auditable entity will be updated");
        }
    }
    
    public void postPersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("PostPersist: Auditable entity saved with ID: " + auditable.getId());
        }
    }
    
    public void postUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("PostUpdate: Auditable entity updated with ID: " + auditable.getId());
        }
    }
    
    public void preRemove(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("PreRemove: Auditable entity will be deleted with ID: " + auditable.getId());
        }
    }
    
    public void postRemove(Object entity) {
        if (entity instanceof Auditable) {
            System.out.println("PostRemove: Auditable entity deleted");
        }
    }
    
    public void postLoad(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("PostLoad: Auditable entity loaded with ID: " + auditable.getId());
        }
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

### 3.2. Универсальный валидатор

```java
public class UniversalValidationListener {
    
    public void prePersist(Object entity) {
        validateEntity(entity, "persist");
    }
    
    public void preUpdate(Object entity) {
        validateEntity(entity, "update");
    }
    
    private void validateEntity(Object entity, String operation) {
        if (entity instanceof Validatable) {
            Validatable validatable = (Validatable) entity;
            try {
                validatable.validate();
                System.out.println("Validation passed for " + operation + ": " + entity.getClass().getSimpleName());
            } catch (ValidationException e) {
                System.err.println("Validation failed for " + operation + ": " + e.getMessage());
                throw e;
            }
        }
    }
}
```

### 3.3. Универсальный логгер

```java
public class UniversalLoggingListener {
    
    private static final Logger logger = LoggerFactory.getLogger(UniversalLoggingListener.class);
    
    public void prePersist(Object entity) {
        logger.info("PrePersist: {} will be saved", entity.getClass().getSimpleName());
    }
    
    public void postPersist(Object entity) {
        logger.info("PostPersist: {} saved successfully", entity.getClass().getSimpleName());
    }
    
    public void preUpdate(Object entity) {
        logger.info("PreUpdate: {} will be updated", entity.getClass().getSimpleName());
    }
    
    public void postUpdate(Object entity) {
        logger.info("PostUpdate: {} updated successfully", entity.getClass().getSimpleName());
    }
    
    public void preRemove(Object entity) {
        logger.info("PreRemove: {} will be deleted", entity.getClass().getSimpleName());
    }
    
    public void postRemove(Object entity) {
        logger.info("PostRemove: {} deleted successfully", entity.getClass().getSimpleName());
    }
    
    public void postLoad(Object entity) {
        logger.info("PostLoad: {} loaded successfully", entity.getClass().getSimpleName());
    }
}
```

## 4. Специализированные ListenerCallback

### 4.1. Слушатель для документов

```java
public class DocumentListener {
    
    public void prePersist(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            document.setCreatedAt(LocalDateTime.now());
            document.setStatus("DRAFT");
            System.out.println("PrePersist: Document prepared for save: " + document.getFileName());
        }
    }
    
    public void postPersist(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            System.out.println("PostPersist: Document saved with ID: " + document.getId());
            // Можно отправить уведомление о создании документа
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            document.setUpdatedAt(LocalDateTime.now());
            System.out.println("PreUpdate: Document prepared for update: " + document.getFileName());
        }
    }
    
    public void postUpdate(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            System.out.println("PostUpdate: Document updated: " + document.getFileName());
            // Можно отправить уведомление об изменении документа
        }
    }
    
    public void preRemove(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            System.out.println("PreRemove: Document will be deleted: " + document.getFileName());
            // Создание резервной копии
            createBackup(document);
        }
    }
    
    public void postRemove(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            System.out.println("PostRemove: Document deleted: " + document.getFileName());
            // Очистка связанных файлов
            cleanupFiles(document);
        }
    }
    
    private void createBackup(Document document) {
        // Логика создания резервной копии
        System.out.println("Backup created for document: " + document.getFileName());
    }
    
    private void cleanupFiles(Document document) {
        // Логика очистки файлов
        System.out.println("Files cleaned for document: " + document.getFileName());
    }
}
```

### 4.2. Слушатель для заказов

```java
public class OrderListener {
    
    public void prePersist(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            order.setCreatedAt(LocalDateTime.now());
            order.setStatus("PENDING");
            order.setOrderNumber(generateOrderNumber());
            System.out.println("PrePersist: Order prepared for save: " + order.getOrderNumber());
        }
    }
    
    public void postPersist(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            System.out.println("PostPersist: Order saved with ID: " + order.getId());
            // Отправка уведомления о создании заказа
            sendOrderNotification(order);
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            order.setUpdatedAt(LocalDateTime.now());
            System.out.println("PreUpdate: Order prepared for update: " + order.getOrderNumber());
        }
    }
    
    public void postUpdate(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            System.out.println("PostUpdate: Order updated: " + order.getOrderNumber());
            // Проверка изменения статуса
            checkStatusChange(order);
        }
    }
    
    public void preRemove(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            System.out.println("PreRemove: Order will be deleted: " + order.getOrderNumber());
            // Проверка возможности удаления
            validateOrderDeletion(order);
        }
    }
    
    public void postRemove(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            System.out.println("PostRemove: Order deleted: " + order.getOrderNumber());
            // Очистка связанных данных
            cleanupOrderData(order);
        }
    }
    
    private String generateOrderNumber() {
        return "ORD-" + System.currentTimeMillis();
    }
    
    private void sendOrderNotification(Order order) {
        // Логика отправки уведомления
        System.out.println("Order notification sent for: " + order.getOrderNumber());
    }
    
    private void checkStatusChange(Order order) {
        // Логика проверки изменения статуса
        System.out.println("Status change checked for order: " + order.getOrderNumber());
    }
    
    private void validateOrderDeletion(Order order) {
        // Логика валидации удаления
        if ("CONFIRMED".equals(order.getStatus())) {
            throw new IllegalStateException("Cannot delete confirmed order");
        }
    }
    
    private void cleanupOrderData(Order order) {
        // Логика очистки данных заказа
        System.out.println("Order data cleaned for: " + order.getOrderNumber());
    }
}
```

## 5. Комбинирование ListenerCallback

### 5.1. Множественные слушатели

```java
@Entity
@EntityListeners({AuditListener.class, ValidationListener.class, LoggingListener.class})
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String email;
    
    // Аудит поля
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String createdBy;
    private String updatedBy;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
    
    public String getCreatedBy() { return createdBy; }
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
    
    public String getUpdatedBy() { return updatedBy; }
    public void setUpdatedBy(String updatedBy) { this.updatedBy = updatedBy; }
}
```

### 5.2. Специализированные слушатели

```java
@Entity
@EntityListeners({DocumentListener.class, AuditListener.class})
@Table(name = "documents")
public class Document {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String fileName;
    private String filePath;
    private String status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getFileName() { return fileName; }
    public void setFileName(String fileName) { this.fileName = fileName; }
    
    public String getFilePath() { return filePath; }
    public void setFilePath(String filePath) { this.filePath = filePath; }
    
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}
```

## 6. Вопросы для собеседования

### Базовые вопросы

1. **Что такое ListenerCallback в JPA?**
   - Механизм для создания переиспользуемых слушателей событий жизненного цикла
   - Позволяет применять одну логику к множеству сущностей
   - Регистрируется через аннотацию @EntityListeners

2. **Какие преимущества у ListenerCallback по сравнению с Entity Callbacks?**
   - Переиспользование логики
   - Разделение ответственности
   - Лучшая тестируемость
   - Гибкость в комбинировании

3. **Как зарегистрировать ListenerCallback для сущности?**
   - Использовать аннотацию @EntityListeners
   - Указать классы слушателей в массиве
   - Можно комбинировать несколько слушателей

### Продвинутые вопросы

4. **В чем разница между ListenerCallback и Entity Callbacks?**
   - ListenerCallback: переиспользуемый, отдельный класс
   - Entity Callbacks: встроенные в сущность методы с аннотациями

5. **Как создать универсальный ListenerCallback?**
   - Использовать instanceof для проверки типа
   - Применять интерфейсы для типизации
   - Обрабатывать различные типы сущностей

6. **Как тестировать ListenerCallback?**
   - Unit тесты с моками
   - Integration тесты с реальной базой данных
   - Проверка выполнения логики

### Практические вопросы

7. **Реализуйте универсальный аудит слушатель**
```java
public class UniversalAuditListener {
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
        }
    }
}
```

8. **Создайте специализированный слушатель для заказов**
```java
public class OrderListener {
    public void prePersist(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            order.setOrderNumber(generateOrderNumber());
            order.setStatus("PENDING");
        }
    }
    
    public void postPersist(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            sendOrderNotification(order);
        }
    }
}
```

9. **Реализуйте валидацию в ListenerCallback**
```java
public class ValidationListener {
    public void prePersist(Object entity) {
        if (entity instanceof Validatable) {
            Validatable validatable = (Validatable) entity;
            validatable.validate();
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Validatable) {
            Validatable validatable = (Validatable) entity;
            validatable.validate();
        }
    }
}
``` 