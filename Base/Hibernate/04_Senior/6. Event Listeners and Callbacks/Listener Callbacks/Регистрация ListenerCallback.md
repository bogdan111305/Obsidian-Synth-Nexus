# Регистрация ListenerCallback - @EntityListeners

**@EntityListeners** — это аннотация в JPA, которая позволяет регистрировать слушатели событий жизненного цикла для сущностей. Эта аннотация является основным способом подключения ListenerCallback к сущностям.

## 1. Обзор аннотации @EntityListeners

### 1.1. Что такое @EntityListeners?

`@EntityListeners` — это аннотация, которая указывает JPA, какие слушатели событий жизненного цикла должны быть применены к сущности. Она принимает массив классов слушателей.

### 1.2. Синтаксис

```java
@EntityListeners({ListenerClass1.class, ListenerClass2.class, ...})
@Entity
public class EntityClass {
    // поля сущности
}
```

### 1.3. Преимущества использования @EntityListeners

- **Переиспользование**: Один слушатель может использоваться для множества сущностей
- **Разделение ответственности**: Логика слушателя отделена от бизнес-логики
- **Гибкость**: Можно комбинировать несколько слушателей
- **Тестируемость**: Легче тестировать слушатели отдельно

## 2. Базовое использование

### 2.1. Регистрация одного слушателя

```java
@Entity
@EntityListeners(AuditListener.class)
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

### 2.2. Регистрация нескольких слушателей

```java
@Entity
@EntityListeners({AuditListener.class, ValidationListener.class, LoggingListener.class})
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private BigDecimal price;
    
    private int stockQuantity;
    
    // Аудит поля
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String createdBy;
    private String updatedBy;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    
    public int getStockQuantity() { return stockQuantity; }
    public void setStockQuantity(int stockQuantity) { this.stockQuantity = stockQuantity; }
    
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

## 3. Создание слушателей

### 3.1. Аудит слушатель

```java
public class AuditListener {
    
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
            auditable.setUpdatedBy(getCurrentUser());
            
            System.out.println("AuditListener: PrePersist executed for " + entity.getClass().getSimpleName());
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
            
            System.out.println("AuditListener: PreUpdate executed for " + entity.getClass().getSimpleName());
        }
    }
    
    public void postPersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("AuditListener: PostPersist executed for " + entity.getClass().getSimpleName() + " with ID: " + auditable.getId());
        }
    }
    
    public void postUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("AuditListener: PostUpdate executed for " + entity.getClass().getSimpleName() + " with ID: " + auditable.getId());
        }
    }
    
    public void preRemove(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("AuditListener: PreRemove executed for " + entity.getClass().getSimpleName() + " with ID: " + auditable.getId());
        }
    }
    
    public void postRemove(Object entity) {
        System.out.println("AuditListener: PostRemove executed for " + entity.getClass().getSimpleName());
    }
    
    public void postLoad(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("AuditListener: PostLoad executed for " + entity.getClass().getSimpleName() + " with ID: " + auditable.getId());
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

### 3.2. Валидация слушатель

```java
public class ValidationListener {
    
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
                System.out.println("ValidationListener: Validation passed for " + operation + " of " + entity.getClass().getSimpleName());
            } catch (ValidationException e) {
                System.err.println("ValidationListener: Validation failed for " + operation + " of " + entity.getClass().getSimpleName() + ": " + e.getMessage());
                throw e;
            }
        }
    }
}
```

### 3.3. Логирование слушатель

```java
public class LoggingListener {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingListener.class);
    
    public void prePersist(Object entity) {
        logger.info("LoggingListener: PrePersist - {} will be saved", entity.getClass().getSimpleName());
    }
    
    public void postPersist(Object entity) {
        logger.info("LoggingListener: PostPersist - {} saved successfully", entity.getClass().getSimpleName());
    }
    
    public void preUpdate(Object entity) {
        logger.info("LoggingListener: PreUpdate - {} will be updated", entity.getClass().getSimpleName());
    }
    
    public void postUpdate(Object entity) {
        logger.info("LoggingListener: PostUpdate - {} updated successfully", entity.getClass().getSimpleName());
    }
    
    public void preRemove(Object entity) {
        logger.info("LoggingListener: PreRemove - {} will be deleted", entity.getClass().getSimpleName());
    }
    
    public void postRemove(Object entity) {
        logger.info("LoggingListener: PostRemove - {} deleted successfully", entity.getClass().getSimpleName());
    }
    
    public void postLoad(Object entity) {
        logger.info("LoggingListener: PostLoad - {} loaded successfully", entity.getClass().getSimpleName());
    }
}
```

## 4. Специализированные слушатели

### 4.1. Слушатель для документов

```java
public class DocumentListener {
    
    public void prePersist(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            document.setCreatedAt(LocalDateTime.now());
            document.setStatus("DRAFT");
            System.out.println("DocumentListener: Document prepared for save: " + document.getFileName());
        }
    }
    
    public void postPersist(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            System.out.println("DocumentListener: Document saved with ID: " + document.getId());
            sendDocumentNotification(document, "CREATED");
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            document.setUpdatedAt(LocalDateTime.now());
            System.out.println("DocumentListener: Document prepared for update: " + document.getFileName());
        }
    }
    
    public void postUpdate(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            System.out.println("DocumentListener: Document updated: " + document.getFileName());
            sendDocumentNotification(document, "UPDATED");
        }
    }
    
    public void preRemove(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            System.out.println("DocumentListener: Document will be deleted: " + document.getFileName());
            createDocumentBackup(document);
        }
    }
    
    public void postRemove(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            System.out.println("DocumentListener: Document deleted: " + document.getFileName());
            cleanupDocumentFiles(document);
        }
    }
    
    private void sendDocumentNotification(Document document, String action) {
        // Логика отправки уведомления
        System.out.println("Document notification sent for: " + document.getFileName() + " - " + action);
    }
    
    private void createDocumentBackup(Document document) {
        // Логика создания резервной копии
        System.out.println("Document backup created for: " + document.getFileName());
    }
    
    private void cleanupDocumentFiles(Document document) {
        // Логика очистки файлов
        System.out.println("Document files cleaned for: " + document.getFileName());
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
            System.out.println("OrderListener: Order prepared for save: " + order.getOrderNumber());
        }
    }
    
    public void postPersist(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            System.out.println("OrderListener: Order saved with ID: " + order.getId());
            sendOrderNotification(order, "CREATED");
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            order.setUpdatedAt(LocalDateTime.now());
            System.out.println("OrderListener: Order prepared for update: " + order.getOrderNumber());
        }
    }
    
    public void postUpdate(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            System.out.println("OrderListener: Order updated: " + order.getOrderNumber());
            checkOrderStatusChange(order);
            sendOrderNotification(order, "UPDATED");
        }
    }
    
    public void preRemove(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            System.out.println("OrderListener: Order will be deleted: " + order.getOrderNumber());
            validateOrderDeletion(order);
        }
    }
    
    public void postRemove(Object entity) {
        if (entity instanceof Order) {
            Order order = (Order) entity;
            System.out.println("OrderListener: Order deleted: " + order.getOrderNumber());
            cleanupOrderData(order);
        }
    }
    
    private String generateOrderNumber() {
        return "ORD-" + System.currentTimeMillis();
    }
    
    private void sendOrderNotification(Order order, String action) {
        // Логика отправки уведомления
        System.out.println("Order notification sent for: " + order.getOrderNumber() + " - " + action);
    }
    
    private void checkOrderStatusChange(Order order) {
        // Логика проверки изменения статуса
        System.out.println("Order status change checked for: " + order.getOrderNumber());
    }
    
    private void validateOrderDeletion(Order order) {
        // Логика валидации удаления
        if ("CONFIRMED".equals(order.getStatus())) {
            throw new IllegalStateException("Cannot delete confirmed order: " + order.getOrderNumber());
        }
    }
    
    private void cleanupOrderData(Order order) {
        // Логика очистки данных заказа
        System.out.println("Order data cleaned for: " + order.getOrderNumber());
    }
}
```

## 5. Универсальные слушатели

### 5.1. Универсальный аудит слушатель

```java
public class UniversalAuditListener {
    
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
            auditable.setUpdatedBy(getCurrentUser());
            
            System.out.println("UniversalAuditListener: PrePersist executed for " + entity.getClass().getSimpleName());
        }
    }
    
    public void preUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
            
            System.out.println("UniversalAuditListener: PreUpdate executed for " + entity.getClass().getSimpleName());
        }
    }
    
    public void postPersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("UniversalAuditListener: PostPersist executed for " + entity.getClass().getSimpleName() + " with ID: " + auditable.getId());
        }
    }
    
    public void postUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("UniversalAuditListener: PostUpdate executed for " + entity.getClass().getSimpleName() + " with ID: " + auditable.getId());
        }
    }
    
    public void preRemove(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("UniversalAuditListener: PreRemove executed for " + entity.getClass().getSimpleName() + " with ID: " + auditable.getId());
        }
    }
    
    public void postRemove(Object entity) {
        System.out.println("UniversalAuditListener: PostRemove executed for " + entity.getClass().getSimpleName());
    }
    
    public void postLoad(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            System.out.println("UniversalAuditListener: PostLoad executed for " + entity.getClass().getSimpleName() + " with ID: " + auditable.getId());
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

### 5.2. Универсальный валидатор

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
                System.out.println("UniversalValidationListener: Validation passed for " + operation + " of " + entity.getClass().getSimpleName());
            } catch (ValidationException e) {
                System.err.println("UniversalValidationListener: Validation failed for " + operation + " of " + entity.getClass().getSimpleName() + ": " + e.getMessage());
                throw e;
            }
        }
    }
}
```

## 6. Применение слушателей к сущностям

### 6.1. Сущность с аудитом

```java
@Entity
@EntityListeners(UniversalAuditListener.class)
@Table(name = "audited_entities")
public class AuditedEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    // Аудит поля
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
```

### 6.2. Сущность с валидацией

```java
@Entity
@EntityListeners(UniversalValidationListener.class)
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
```

### 6.3. Сущность с множественными слушателями

```java
@Entity
@EntityListeners({UniversalAuditListener.class, UniversalValidationListener.class, LoggingListener.class})
@Table(name = "complex_entities")
public class ComplexEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private BigDecimal price;
    
    // Аудит поля
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String createdBy;
    private String updatedBy;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    
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

## 7. Порядок выполнения слушателей

### 7.1. Последовательность выполнения

Когда к сущности применяется несколько слушателей, они выполняются в порядке, указанном в аннотации `@EntityListeners`:

```java
@Entity
@EntityListeners({AuditListener.class, ValidationListener.class, LoggingListener.class})
public class User {
    // поля сущности
}
```

**Порядок выполнения**:
1. `AuditListener.prePersist()`
2. `ValidationListener.prePersist()`
3. `LoggingListener.prePersist()`
4. Сохранение в БД
5. `LoggingListener.postPersist()`
6. `ValidationListener.postPersist()`
7. `AuditListener.postPersist()`

### 7.2. Пример с логированием порядка

```java
public class OrderedListener {
    
    public void prePersist(Object entity) {
        System.out.println("OrderedListener: PrePersist executed for " + entity.getClass().getSimpleName());
    }
    
    public void postPersist(Object entity) {
        System.out.println("OrderedListener: PostPersist executed for " + entity.getClass().getSimpleName());
    }
    
    public void preUpdate(Object entity) {
        System.out.println("OrderedListener: PreUpdate executed for " + entity.getClass().getSimpleName());
    }
    
    public void postUpdate(Object entity) {
        System.out.println("OrderedListener: PostUpdate executed for " + entity.getClass().getSimpleName());
    }
    
    public void preRemove(Object entity) {
        System.out.println("OrderedListener: PreRemove executed for " + entity.getClass().getSimpleName());
    }
    
    public void postRemove(Object entity) {
        System.out.println("OrderedListener: PostRemove executed for " + entity.getClass().getSimpleName());
    }
    
    public void postLoad(Object entity) {
        System.out.println("OrderedListener: PostLoad executed for " + entity.getClass().getSimpleName());
    }
}
```

## 8. Вопросы для собеседования

### Базовые вопросы

1. **Что такое аннотация @EntityListeners?**
   - Аннотация для регистрации слушателей событий жизненного цикла
   - Принимает массив классов слушателей
   - Позволяет применять переиспользуемую логику к сущностям

2. **Как зарегистрировать несколько слушателей для одной сущности?**
   - Указать массив классов в аннотации @EntityListeners
   - Слушатели выполняются в указанном порядке
   - Можно комбинировать универсальные и специализированные слушатели

3. **В чем преимущества использования @EntityListeners?**
   - Переиспользование логики
   - Разделение ответственности
   - Лучшая тестируемость
   - Гибкость в комбинировании

### Продвинутые вопросы

4. **Как создать универсальный слушатель?**
   - Использовать instanceof для проверки типа
   - Применять интерфейсы для типизации
   - Обрабатывать различные типы сущностей

5. **В каком порядке выполняются слушатели?**
   - В порядке, указанном в аннотации @EntityListeners
   - Сначала pre-методы, затем операция, затем post-методы
   - Для каждого этапа в обратном порядке

6. **Как тестировать слушатели, зарегистрированные через @EntityListeners?**
   - Unit тесты с моками
   - Integration тесты с реальной базой данных
   - Проверка выполнения логики и порядка

### Практические вопросы

7. **Реализуйте универсальный аудит слушатель с @EntityListeners**
```java
@Entity
@EntityListeners(UniversalAuditListener.class)
public class AuditedEntity {
    // поля сущности
}

public class UniversalAuditListener {
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
        }
    }
}
```

8. **Создайте сущность с множественными слушателями**
```java
@Entity
@EntityListeners({AuditListener.class, ValidationListener.class, LoggingListener.class})
public class ComplexEntity {
    // поля сущности
}
```

9. **Реализуйте специализированный слушатель для документов**
```java
@Entity
@EntityListeners(DocumentListener.class)
public class Document {
    // поля сущности
}

public class DocumentListener {
    public void prePersist(Object entity) {
        if (entity instanceof Document) {
            Document document = (Document) entity;
            document.setStatus("DRAFT");
        }
    }
}
``` 