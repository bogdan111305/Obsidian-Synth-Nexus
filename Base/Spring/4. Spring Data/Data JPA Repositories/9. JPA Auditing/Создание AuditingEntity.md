# Создание AuditingEntity

## Обзор

AuditingEntity в Spring Data JPA позволяет автоматически отслеживать изменения в сущностях, такие как время создания, время последнего изменения, кто создал и кто изменил запись. Это полезно для аудита и отслеживания истории изменений.

## Основные возможности

### Автоматическое отслеживание
- Время создания записи
- Время последнего изменения
- Пользователь, создавший запись
- Пользователь, изменивший запись

### Структура AuditingEntity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditingEntity {
    
    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private LocalDateTime lastModifiedDate;
    
    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "last_modified_by")
    private String lastModifiedBy;
}
```

## Базовая реализация

### Простая AuditingEntity
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditingEntity {
    
    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private LocalDateTime lastModifiedDate;
    
    // Геттеры и сеттеры
    public LocalDateTime getCreatedDate() {
        return createdDate;
    }
    
    public void setCreatedDate(LocalDateTime createdDate) {
        this.createdDate = createdDate;
    }
    
    public LocalDateTime getLastModifiedDate() {
        return lastModifiedDate;
    }
    
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) {
        this.lastModifiedDate = lastModifiedDate;
    }
}
```

### Расширенная AuditingEntity
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditingEntity {
    
    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private LocalDateTime lastModifiedDate;
    
    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "last_modified_by", length = 100)
    private String lastModifiedBy;
    
    @Version
    @Column(name = "version")
    private Long version;
    
    // Геттеры и сеттеры
    public LocalDateTime getCreatedDate() {
        return createdDate;
    }
    
    public void setCreatedDate(LocalDateTime createdDate) {
        this.createdDate = createdDate;
    }
    
    public LocalDateTime getLastModifiedDate() {
        return lastModifiedDate;
    }
    
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) {
        this.lastModifiedDate = lastModifiedDate;
    }
    
    public String getCreatedBy() {
        return createdBy;
    }
    
    public void setCreatedBy(String createdBy) {
        this.createdBy = createdBy;
    }
    
    public String getLastModifiedBy() {
        return lastModifiedBy;
    }
    
    public void setLastModifiedBy(String lastModifiedBy) {
        this.lastModifiedBy = lastModifiedBy;
    }
    
    public Long getVersion() {
        return version;
    }
    
    public void setVersion(Long version) {
        this.version = version;
    }
}
```

## Использование в сущностях

### Простая сущность с аудитом
```java
@Entity
@Table(name = "users")
public class User extends AuditingEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "email", nullable = false, unique = true)
    private String email;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "active", nullable = false)
    private boolean active = true;
    
    // Геттеры и сеттеры
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
    
    public String getEmail() {
        return email;
    }
    
    public void setEmail(String email) {
        this.email = email;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public boolean isActive() {
        return active;
    }
    
    public void setActive(boolean active) {
        this.active = active;
    }
}
```

### Сложная сущность с аудитом
```java
@Entity
@Table(name = "orders")
public class Order extends AuditingEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", nullable = false, unique = true)
    private String orderNumber;
    
    @Column(name = "status", nullable = false)
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "total_amount", nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    // Геттеры и сеттеры
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
    
    public String getOrderNumber() {
        return orderNumber;
    }
    
    public void setOrderNumber(String orderNumber) {
        this.orderNumber = orderNumber;
    }
    
    public OrderStatus getStatus() {
        return status;
    }
    
    public void setStatus(OrderStatus status) {
        this.status = status;
    }
    
    public BigDecimal getTotalAmount() {
        return totalAmount;
    }
    
    public void setTotalAmount(BigDecimal totalAmount) {
        this.totalAmount = totalAmount;
    }
    
    public User getUser() {
        return user;
    }
    
    public void setUser(User user) {
        this.user = user;
    }
    
    public List<OrderItem> getItems() {
        return items;
    }
    
    public void setItems(List<OrderItem> items) {
        this.items = items;
    }
}
```

## Кастомные AuditingEntity

### AuditingEntity с дополнительными полями
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditingEntity {
    
    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private LocalDateTime lastModifiedDate;
    
    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "last_modified_by", length = 100)
    private String lastModifiedBy;
    
    @Column(name = "deleted", nullable = false)
    private boolean deleted = false;
    
    @Column(name = "deleted_date")
    private LocalDateTime deletedDate;
    
    @Column(name = "deleted_by", length = 100)
    private String deletedBy;
    
    // Геттеры и сеттеры
    public LocalDateTime getCreatedDate() {
        return createdDate;
    }
    
    public void setCreatedDate(LocalDateTime createdDate) {
        this.createdDate = createdDate;
    }
    
    public LocalDateTime getLastModifiedDate() {
        return lastModifiedDate;
    }
    
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) {
        this.lastModifiedDate = lastModifiedDate;
    }
    
    public String getCreatedBy() {
        return createdBy;
    }
    
    public void setCreatedBy(String createdBy) {
        this.createdBy = createdBy;
    }
    
    public String getLastModifiedBy() {
        return lastModifiedBy;
    }
    
    public void setLastModifiedBy(String lastModifiedBy) {
        this.lastModifiedBy = lastModifiedBy;
    }
    
    public boolean isDeleted() {
        return deleted;
    }
    
    public void setDeleted(boolean deleted) {
        this.deleted = deleted;
    }
    
    public LocalDateTime getDeletedDate() {
        return deletedDate;
    }
    
    public void setDeletedDate(LocalDateTime deletedDate) {
        this.deletedDate = deletedDate;
    }
    
    public String getDeletedBy() {
        return deletedBy;
    }
    
    public void setDeletedBy(String deletedBy) {
        this.deletedBy = deletedBy;
    }
}
```

### AuditingEntity с UUID
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditingEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(name = "id", nullable = false, updatable = false)
    private UUID id;
    
    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private LocalDateTime lastModifiedDate;
    
    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "last_modified_by", length = 100)
    private String lastModifiedBy;
    
    // Геттеры и сеттеры
    public UUID getId() {
        return id;
    }
    
    public void setId(UUID id) {
        this.id = id;
    }
    
    public LocalDateTime getCreatedDate() {
        return createdDate;
    }
    
    public void setCreatedDate(LocalDateTime createdDate) {
        this.createdDate = createdDate;
    }
    
    public LocalDateTime getLastModifiedDate() {
        return lastModifiedDate;
    }
    
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) {
        this.lastModifiedDate = lastModifiedDate;
    }
    
    public String getCreatedBy() {
        return createdBy;
    }
    
    public void setCreatedBy(String createdBy) {
        this.createdBy = createdBy;
    }
    
    public String getLastModifiedBy() {
        return lastModifiedBy;
    }
    
    public void setLastModifiedBy(String lastModifiedBy) {
        this.lastModifiedBy = lastModifiedBy;
    }
}
```

## Конфигурация

### Включение JPA Auditing
```java
@Configuration
@EnableJpaAuditing(
    auditorAwareRef = "auditorAwareImpl",
    dateTimeProviderRef = "dateTimeProvider"
)
public class JpaAuditingConfig {
    
    @Bean
    public AuditorAware<String> auditorAwareImpl() {
        return new AuditorAwareImpl();
    }
    
    @Bean
    public DateTimeProvider dateTimeProvider() {
        return () -> Optional.of(LocalDateTime.now());
    }
}
```

### Реализация AuditorAware
```java
@Component
public class AuditorAwareImpl implements AuditorAware<String> {
    
    @Override
    public Optional<String> getCurrentAuditor() {
        // Получение текущего пользователя из контекста безопасности
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication == null || 
            authentication.getPrincipal().equals("anonymousUser")) {
            return Optional.of("system");
        }
        
        return Optional.of(authentication.getName());
    }
}
```

## Использование в репозиториях

### Базовые операции
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    List<User> findByCreatedDateBetween(LocalDateTime start, LocalDateTime end);
    List<User> findByCreatedBy(String createdBy);
    List<User> findByLastModifiedDateAfter(LocalDateTime date);
    List<User> findByLastModifiedBy(String lastModifiedBy);
}
```

### Сложные запросы
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE u.createdDate >= ?1 AND u.createdBy = ?2")
    List<User> findRecentByCreator(LocalDateTime since, String creator);
    
    @Query("SELECT u FROM User u WHERE u.lastModifiedDate >= ?1 AND u.lastModifiedBy != ?2")
    List<User> findRecentlyModifiedByOthers(LocalDateTime since, String excludeUser);
}
```

## Лучшие практики

### Структурирование AuditingEntity
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditingEntity {
    
    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private LocalDateTime lastModifiedDate;
    
    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "last_modified_by", length = 100)
    private String lastModifiedBy;
    
    // Методы для логирования
    public String getAuditInfo() {
        return String.format("Created: %s by %s, Modified: %s by %s",
            createdDate, createdBy, lastModifiedDate, lastModifiedBy);
    }
    
    public boolean isRecentlyModified(Duration threshold) {
        return lastModifiedDate != null && 
               lastModifiedDate.isAfter(LocalDateTime.now().minus(threshold));
    }
}
```

### Валидация AuditingEntity
```java
@Component
public class AuditingEntityValidator {
    
    public void validateAuditingEntity(AuditingEntity entity) {
        if (entity.getCreatedDate() == null) {
            throw new IllegalArgumentException("Created date cannot be null");
        }
        
        if (entity.getCreatedBy() == null || entity.getCreatedBy().trim().isEmpty()) {
            throw new IllegalArgumentException("Created by cannot be null or empty");
        }
        
        if (entity.getLastModifiedDate() != null && 
            entity.getLastModifiedDate().isBefore(entity.getCreatedDate())) {
            throw new IllegalArgumentException("Last modified date cannot be before created date");
        }
    }
}
```

## Отладка и логирование

### Включение логирования
```properties
logging.level.org.springframework.data.jpa.domain.support.AuditingEntityListener=DEBUG
logging.level.org.springframework.data.auditing=DEBUG
```

### Отладочная информация
```java
@Component
public class AuditingEntityDebugger {
    
    public void debugAuditingEntity(AuditingEntity entity) {
        System.out.println("Entity class: " + entity.getClass().getName());
        System.out.println("Created date: " + entity.getCreatedDate());
        System.out.println("Created by: " + entity.getCreatedBy());
        System.out.println("Last modified date: " + entity.getLastModifiedDate());
        System.out.println("Last modified by: " + entity.getLastModifiedBy());
    }
}
```

## Лучшие практики

1. **Используйте @MappedSuperclass** для наследования аудита
2. **Настройте AuditorAware** для автоматического заполнения пользователей
3. **Используйте @EnableJpaAuditing** для включения аудита
4. **Валидируйте аудиторские поля** в бизнес-логике
5. **Документируйте аудиторские поля** для понимания их назначения
6. **Тестируйте аудиторскую функциональность** в unit-тестах
7. **Мониторьте производительность** аудиторских операций 