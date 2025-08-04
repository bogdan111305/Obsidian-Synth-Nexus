# Интерфейс Callback в JPA

**Entity Callbacks** в JPA — это механизм, позволяющий выполнять пользовательскую логику на различных этапах жизненного цикла сущности. Интерфейс `Callback` является базовым для всех обратных вызовов в JPA.

## 1. Обзор интерфейса Callback

### 1.1. Что такое Callback?

`Callback` — это интерфейс в JPA, который определяет методы для обработки различных событий жизненного цикла сущности. Он позволяет выполнять логику до и после основных операций с сущностями (сохранение, обновление, удаление, загрузка).

### 1.2. Основные характеристики

- **Автоматическое выполнение**: Callbacks выполняются автоматически при соответствующих операциях
- **Жизненный цикл**: Привязаны к конкретным этапам жизненного цикла сущности
- **Контекст**: Имеют доступ к состоянию сущности и метаданным
- **Транзакционность**: Выполняются в рамках текущей транзакции

## 2. Аннотации для Entity Callbacks

JPA предоставляет набор аннотаций для различных этапов жизненного цикла:

### 2.1. @PrePersist

**Описание**: Выполняется перед сохранением новой сущности в базу данных.

**Использование**:
```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private LocalDateTime createdAt;
    
    @PrePersist
    public void prePersist() {
        this.createdAt = LocalDateTime.now();
        System.out.println("PrePersist: User will be saved with ID: " + id);
    }
}
```

**Типичные применения**:
- Установка временных меток
- Генерация значений по умолчанию
- Валидация данных
- Логирование

### 2.2. @PostPersist

**Описание**: Выполняется после сохранения сущности в базу данных.

**Использование**:
```java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private LocalDateTime createdAt;
    
    @PostPersist
    public void postPersist() {
        System.out.println("PostPersist: Order saved with ID: " + id);
        // Можно отправить уведомление или выполнить другие действия
    }
}
```

**Типичные применения**:
- Отправка уведомлений
- Обновление кэша
- Логирование успешных операций
- Триггер внешних систем

### 2.3. @PreUpdate

**Описание**: Выполняется перед обновлением существующей сущности.

**Использование**:
```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private LocalDateTime updatedAt;
    private String updatedBy;
    
    @PreUpdate
    public void preUpdate() {
        this.updatedAt = LocalDateTime.now();
        // Получение текущего пользователя из контекста
        this.updatedBy = SecurityContextHolder.getContext().getAuthentication().getName();
        System.out.println("PreUpdate: Product will be updated: " + id);
    }
}
```

**Типичные применения**:
- Обновление временных меток
- Аудит изменений
- Валидация перед обновлением
- Логирование изменений

### 2.4. @PostUpdate

**Описание**: Выполняется после обновления сущности.

**Использование**:
```java
@Entity
public class Inventory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String productName;
    private int quantity;
    private LocalDateTime lastUpdated;
    
    @PostUpdate
    public void postUpdate() {
        System.out.println("PostUpdate: Inventory updated for product: " + productName);
        // Можно отправить уведомление о низком количестве
        if (quantity < 10) {
            // Логика уведомления
        }
    }
}
```

**Типичные применения**:
- Отправка уведомлений об изменениях
- Обновление индексов поиска
- Синхронизация с внешними системами
- Аналитика изменений

### 2.5. @PreRemove

**Описание**: Выполняется перед удалением сущности.

**Использование**:
```java
@Entity
public class Document {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String fileName;
    private String filePath;
    
    @PreRemove
    public void preRemove() {
        System.out.println("PreRemove: Document will be deleted: " + fileName);
        // Можно выполнить резервное копирование или архивирование
        backupDocument();
    }
    
    private void backupDocument() {
        // Логика резервного копирования
    }
}
```

**Типичные применения**:
- Резервное копирование данных
- Проверка зависимостей
- Логирование удаления
- Архивирование данных

### 2.6. @PostRemove

**Описание**: Выполняется после удаления сущности.

**Использование**:
```java
@Entity
public class File {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String fileName;
    private String filePath;
    
    @PostRemove
    public void postRemove() {
        System.out.println("PostRemove: File deleted: " + fileName);
        // Удаление физического файла
        deletePhysicalFile();
    }
    
    private void deletePhysicalFile() {
        try {
            Files.deleteIfExists(Paths.get(filePath));
        } catch (IOException e) {
            System.err.println("Error deleting file: " + e.getMessage());
        }
    }
}
```

**Типичные применения**:
- Очистка связанных ресурсов
- Удаление физических файлов
- Обновление кэша
- Уведомления об удалении

### 2.7. @PostLoad

**Описание**: Выполняется после загрузки сущности из базы данных.

**Использование**:
```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String firstName;
    private String lastName;
    
    @Transient
    private String fullName;
    
    @PostLoad
    public void postLoad() {
        this.fullName = firstName + " " + lastName;
        System.out.println("PostLoad: User loaded: " + fullName);
    }
}
```

**Типичные применения**:
- Вычисление производных полей
- Инициализация кэша
- Логирование загрузки
- Валидация загруженных данных

## 3. Комбинированное использование

### 3.1. Полный пример с аудитом

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String email;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Column(name = "created_by")
    private String createdBy;
    
    @Column(name = "updated_by")
    private String updatedBy;
    
    @Column(name = "is_active")
    private boolean active = true;
    
    // PrePersist - перед сохранением
    @PrePersist
    public void prePersist() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
        this.createdBy = getCurrentUser();
        this.updatedBy = getCurrentUser();
        System.out.println("PrePersist: Creating user: " + username);
    }
    
    // PostPersist - после сохранения
    @PostPersist
    public void postPersist() {
        System.out.println("PostPersist: User created with ID: " + id);
        // Можно отправить приветственное письмо
        sendWelcomeEmail();
    }
    
    // PreUpdate - перед обновлением
    @PreUpdate
    public void preUpdate() {
        this.updatedAt = LocalDateTime.now();
        this.updatedBy = getCurrentUser();
        System.out.println("PreUpdate: Updating user: " + username);
    }
    
    // PostUpdate - после обновления
    @PostUpdate
    public void postUpdate() {
        System.out.println("PostUpdate: User updated: " + username);
        // Можно отправить уведомление об изменении
        sendUpdateNotification();
    }
    
    // PreRemove - перед удалением
    @PreRemove
    public void preRemove() {
        System.out.println("PreRemove: Deleting user: " + username);
        // Можно создать резервную копию
        createBackup();
    }
    
    // PostRemove - после удаления
    @PostRemove
    public void postRemove() {
        System.out.println("PostRemove: User deleted: " + username);
        // Очистка связанных ресурсов
        cleanupResources();
    }
    
    // PostLoad - после загрузки
    @PostLoad
    public void postLoad() {
        System.out.println("PostLoad: User loaded: " + username);
        // Можно обновить статистику или кэш
        updateUserStatistics();
    }
    
    // Вспомогательные методы
    private String getCurrentUser() {
        try {
            return SecurityContextHolder.getContext().getAuthentication().getName();
        } catch (Exception e) {
            return "system";
        }
    }
    
    private void sendWelcomeEmail() {
        // Логика отправки приветственного письма
        System.out.println("Sending welcome email to: " + email);
    }
    
    private void sendUpdateNotification() {
        // Логика отправки уведомления об обновлении
        System.out.println("Sending update notification to: " + email);
    }
    
    private void createBackup() {
        // Логика создания резервной копии
        System.out.println("Creating backup for user: " + username);
    }
    
    private void cleanupResources() {
        // Логика очистки ресурсов
        System.out.println("Cleaning up resources for user: " + username);
    }
    
    private void updateUserStatistics() {
        // Логика обновления статистики
        System.out.println("Updating statistics for user: " + username);
    }
    
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
    
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
}
```

### 3.2. Пример с валидацией

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private BigDecimal price;
    
    @Column(name = "stock_quantity")
    private int stockQuantity;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    // PrePersist - валидация перед сохранением
    @PrePersist
    public void prePersist() {
        validateProduct();
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
        System.out.println("PrePersist: Product validated and ready for save: " + name);
    }
    
    // PreUpdate - валидация перед обновлением
    @PreUpdate
    public void preUpdate() {
        validateProduct();
        this.updatedAt = LocalDateTime.now();
        System.out.println("PreUpdate: Product validated and ready for update: " + name);
    }
    
    // PostLoad - вычисление производных полей
    @PostLoad
    public void postLoad() {
        System.out.println("PostLoad: Product loaded: " + name + ", Stock: " + stockQuantity);
        // Можно вычислить статус продукта
        updateProductStatus();
    }
    
    // Валидация продукта
    private void validateProduct() {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("Product name cannot be empty");
        }
        
        if (price == null || price.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Product price must be positive");
        }
        
        if (stockQuantity < 0) {
            throw new IllegalArgumentException("Stock quantity cannot be negative");
        }
    }
    
    // Обновление статуса продукта
    private void updateProductStatus() {
        if (stockQuantity == 0) {
            System.out.println("Product " + name + " is out of stock");
        } else if (stockQuantity < 10) {
            System.out.println("Product " + name + " has low stock: " + stockQuantity);
        }
    }
    
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
}
```

## 4. Порядок выполнения Callbacks

### 4.1. Последовательность выполнения

1. **@PrePersist** → Сохранение в БД → **@PostPersist**
2. **@PreUpdate** → Обновление в БД → **@PostUpdate**
3. **@PreRemove** → Удаление из БД → **@PostRemove**
4. Загрузка из БД → **@PostLoad**

### 4.2. Пример с логированием порядка

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    @PrePersist
    public void prePersist() {
        System.out.println("1. @PrePersist executed");
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }
    
    @PostPersist
    public void postPersist() {
        System.out.println("2. @PostPersist executed");
    }
    
    @PreUpdate
    public void preUpdate() {
        System.out.println("3. @PreUpdate executed");
        this.updatedAt = LocalDateTime.now();
    }
    
    @PostUpdate
    public void postUpdate() {
        System.out.println("4. @PostUpdate executed");
    }
    
    @PreRemove
    public void preRemove() {
        System.out.println("5. @PreRemove executed");
    }
    
    @PostRemove
    public void postRemove() {
        System.out.println("6. @PostRemove executed");
    }
    
    @PostLoad
    public void postLoad() {
        System.out.println("7. @PostLoad executed");
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getOrderNumber() { return orderNumber; }
    public void setOrderNumber(String orderNumber) { this.orderNumber = orderNumber; }
    
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}
```

## 5. Ограничения и рекомендации

### 5.1. Ограничения

- **Нет доступа к EntityManager**: Callbacks не могут использовать EntityManager
- **Нет доступа к другим бинам**: Нельзя инжектировать Spring бины
- **Синхронное выполнение**: Callbacks выполняются синхронно
- **Транзакционность**: Выполняются в рамках текущей транзакции

### 5.2. Рекомендации

1. **Используйте для простой логики**: Callbacks подходят для простых операций
2. **Избегайте тяжелых операций**: Не выполняйте долгие операции в Callbacks
3. **Логируйте действия**: Добавляйте логирование для отладки
4. **Обрабатывайте исключения**: Обрабатывайте возможные исключения
5. **Тестируйте сценарии**: Тестируйте все возможные сценарии

### 5.3. Альтернативы

Для сложной логики рассмотрите:
- **Spring Events**: Для бизнес-событий
- **Hibernate Event Listeners**: Для низкоуровневых операций
- **AOP**: Для перекрестных проблем
- **Сервисный слой**: Для сложной бизнес-логики

## 6. Вопросы для собеседования

### Базовые вопросы

1. **Что такое Entity Callbacks в JPA?**
   - Механизм для выполнения логики на различных этапах жизненного цикла сущности
   - Автоматическое выполнение при соответствующих операциях
   - Привязка к конкретным этапам жизненного цикла

2. **Какие аннотации доступны для Entity Callbacks?**
   - @PrePersist, @PostPersist
   - @PreUpdate, @PostUpdate
   - @PreRemove, @PostRemove
   - @PostLoad

3. **В каком порядке выполняются Entity Callbacks?**
   - @PrePersist → Сохранение → @PostPersist
   - @PreUpdate → Обновление → @PostUpdate
   - @PreRemove → Удаление → @PostRemove
   - Загрузка → @PostLoad

### Продвинутые вопросы

4. **Какие ограничения есть у Entity Callbacks?**
   - Нет доступа к EntityManager
   - Нет доступа к Spring бинам
   - Синхронное выполнение
   - Выполнение в рамках транзакции

5. **Когда использовать Entity Callbacks, а когда Spring Events?**
   - Entity Callbacks: для логики, связанной с сущностями
   - Spring Events: для бизнес-событий и сложной логики

6. **Как обрабатывать исключения в Entity Callbacks?**
   - Использовать try-catch блоки
   - Логировать исключения
   - Не прерывать основную операцию

### Практические вопросы

7. **Напишите Entity Callback для аудита**
```java
@Entity
public class AuditedEntity {
    @PrePersist
    public void prePersist() {
        this.createdAt = LocalDateTime.now();
        this.createdBy = getCurrentUser();
    }
    
    @PreUpdate
    public void preUpdate() {
        this.updatedAt = LocalDateTime.now();
        this.updatedBy = getCurrentUser();
    }
}
```

8. **Реализуйте валидацию в Entity Callback**
```java
@Entity
public class ValidatedEntity {
    @PrePersist
    @PreUpdate
    public void validate() {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("Name cannot be empty");
        }
    }
}
```

9. **Создайте Entity Callback для вычисления производных полей**
```java
@Entity
public class CalculatedEntity {
    @PostLoad
    public void calculateDerivedFields() {
        this.fullName = firstName + " " + lastName;
        this.age = Period.between(birthDate, LocalDate.now()).getYears();
    }
}
``` 