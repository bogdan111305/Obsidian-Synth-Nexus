# Аннотация @EntityGraph

## Обзор

Аннотация `@EntityGraph` в Spring Data JPA позволяет контролировать загрузку связанных сущностей, определяя, какие ассоциации должны быть загружены eagerly (жадно) или lazily (лениво). Это помогает оптимизировать производительность запросов и избежать проблем с N+1 запросами.

## Основные возможности

### Типы EntityGraph
- **Fetch Graph** - загружает только указанные атрибуты
- **Load Graph** - загружает указанные атрибуты + все атрибуты по умолчанию

### Структура аннотации

```java
@EntityGraph(
    type = EntityGraph.EntityGraphType.FETCH,
    attributePaths = {"attribute1", "attribute2"}
)
```

## Базовые примеры

### Простой EntityGraph
```java
@Entity
public class User {
    @Id
    private Long id;
    private String email;
    private String name;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Role role;
    
    @OneToOne(fetch = FetchType.LAZY)
    private Profile profile;
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph(attributePaths = {"orders", "role"})
    List<User> findByActive(boolean active);
    
    @EntityGraph(attributePaths = {"profile"})
    Optional<User> findByEmail(String email);
}
```

### EntityGraph с вложенными атрибутами
```java
@Entity
public class Order {
    @Id
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private User user;
    
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Payment payment;
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @EntityGraph(attributePaths = {"user", "items", "payment"})
    List<Order> findByUserId(Long userId);
    
    @EntityGraph(attributePaths = {"user.profile", "items.product"})
    Optional<Order> findById(Long id);
}
```

## Типы EntityGraph

### Fetch Graph
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph(
        type = EntityGraph.EntityGraphType.FETCH,
        attributePaths = {"orders", "role"}
    )
    List<User> findUsersWithOrdersAndRole();
}
```

**Fetch Graph** загружает только указанные атрибуты, игнорируя настройки по умолчанию.

### Load Graph
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph(
        type = EntityGraph.EntityGraphType.LOAD,
        attributePaths = {"orders"}
    )
    List<User> findUsersWithOrders();
}
```

**Load Graph** загружает указанные атрибуты + все атрибуты, настроенные как EAGER по умолчанию.

## Именованные EntityGraph

### Определение в сущности
```java
@Entity
@NamedEntityGraphs({
    @NamedEntityGraph(
        name = "User.withOrders",
        attributeNodes = {
            @NamedAttributeNode("orders"),
            @NamedAttributeNode("role")
        }
    ),
    @NamedEntityGraph(
        name = "User.withProfile",
        attributeNodes = {
            @NamedAttributeNode("profile"),
            @NamedAttributeNode("role")
        }
    ),
    @NamedEntityGraph(
        name = "User.withOrdersAndItems",
        attributeNodes = {
            @NamedAttributeNode(value = "orders", subgraph = "orders.items")
        },
        subgraphs = {
            @NamedSubgraph(
                name = "orders.items",
                attributeNodes = {
                    @NamedAttributeNode("items")
                }
            )
        }
    )
})
public class User {
    // поля сущности
}
```

### Использование именованных EntityGraph
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph("User.withOrders")
    List<User> findUsersWithOrders();
    
    @EntityGraph("User.withProfile")
    Optional<User> findByIdWithProfile(Long id);
    
    @EntityGraph("User.withOrdersAndItems")
    List<User> findUsersWithOrdersAndItems();
}
```

## Сложные EntityGraph

### Множественные атрибуты
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph(attributePaths = {
        "orders", 
        "role", 
        "profile", 
        "orders.items", 
        "orders.payment"
    })
    List<User> findUsersWithCompleteData();
}
```

### Условные EntityGraph
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph(attributePaths = {"orders"})
    List<User> findByActive(boolean active);
    
    @EntityGraph(attributePaths = {"orders", "role"})
    List<User> findByRole(String role);
    
    // Без EntityGraph для простых запросов
    List<User> findByEmail(String email);
}
```

## EntityGraph с параметрами

### Динамические EntityGraph
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE u.active = ?1")
    @EntityGraph(attributePaths = {"orders"})
    List<User> findActiveUsersWithOrders(boolean active);
    
    @Query("SELECT u FROM User u WHERE u.role = ?1")
    @EntityGraph(attributePaths = {"profile", "role"})
    List<User> findUsersByRoleWithProfile(String role);
}
```

### EntityGraph с Pageable
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph(attributePaths = {"orders"})
    Page<User> findByActive(boolean active, Pageable pageable);
    
    @EntityGraph(attributePaths = {"role", "profile"})
    Slice<User> findByRole(String role, Pageable pageable);
}
```

## Оптимизация производительности

### Избежание N+1 проблем
```java
// Плохо - N+1 проблема
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByActive(boolean active); // orders загружаются лениво
}

// Хорошо - с EntityGraph
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @EntityGraph(attributePaths = {"orders"})
    List<User> findByActive(boolean active);
}
```

### Селективная загрузка
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Только основные данные
    List<User> findByActive(boolean active);
    
    // С заказами
    @EntityGraph(attributePaths = {"orders"})
    List<User> findActiveUsersWithOrders(boolean active);
    
    // С полным профилем
    @EntityGraph(attributePaths = {"profile", "role", "orders"})
    List<User> findActiveUsersWithFullData(boolean active);
}
```

## Лучшие практики

### Структурирование EntityGraph
```java
@Entity
@NamedEntityGraphs({
    @NamedEntityGraph(
        name = "User.summary",
        attributeNodes = {
            @NamedAttributeNode("role")
        }
    ),
    @NamedEntityGraph(
        name = "User.detail",
        attributeNodes = {
            @NamedAttributeNode("orders"),
            @NamedAttributeNode("profile"),
            @NamedAttributeNode("role")
        }
    ),
    @NamedEntityGraph(
        name = "User.full",
        attributeNodes = {
            @NamedAttributeNode(value = "orders", subgraph = "orders.items"),
            @NamedAttributeNode("profile"),
            @NamedAttributeNode("role")
        },
        subgraphs = {
            @NamedSubgraph(
                name = "orders.items",
                attributeNodes = {
                    @NamedAttributeNode("items")
                }
            )
        }
    )
})
public class User {
    // поля сущности
}
```

### Использование в сервисах
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> getActiveUsers() {
        // Базовый список без связанных данных
        return userRepository.findByActive(true);
    }
    
    public List<User> getActiveUsersWithOrders() {
        // С заказами для отображения в списке
        return userRepository.findActiveUsersWithOrders(true);
    }
    
    public User getUserDetail(Long id) {
        // Полная информация для детальной страницы
        return userRepository.findByIdWithFullData(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

## Отладка и мониторинг

### Включение логирования
```properties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
logging.level.org.springframework.data.jpa.repository.query=DEBUG
```

### Проверка сгенерированных запросов
```java
@Component
public class EntityGraphDebugger {
    
    public void debugEntityGraph(String methodName, List<?> results) {
        System.out.println("Method: " + methodName);
        System.out.println("Results count: " + results.size());
        
        if (!results.isEmpty()) {
            Object first = results.get(0);
            System.out.println("First result class: " + first.getClass().getName());
            
            // Проверка загруженных ассоциаций
            if (first instanceof User) {
                User user = (User) first;
                System.out.println("Orders loaded: " + 
                    (user.getOrders() != null ? user.getOrders().size() : "null"));
            }
        }
    }
}
```

## Лучшие практики

1. **Используйте именованные EntityGraph** для повторяющихся паттернов
2. **Избегайте избыточной загрузки** - загружайте только необходимые данные
3. **Применяйте селективную загрузку** в зависимости от контекста
4. **Мониторьте производительность** сложных EntityGraph
5. **Тестируйте EntityGraph** в unit-тестах
6. **Документируйте сложные EntityGraph** для понимания их назначения
7. **Используйте подграфы** для глубоко вложенных ассоциаций 