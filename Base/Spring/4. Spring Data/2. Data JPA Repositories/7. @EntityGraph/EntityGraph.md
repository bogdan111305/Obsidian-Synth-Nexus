# @EntityGraph в Spring Data JPA

## Обзор

`@EntityGraph` - это мощная аннотация Spring Data JPA, которая позволяет контролировать загрузку связанных сущностей и избегать проблемы N+1 запросов. Она предоставляет гибкий способ определения того, какие ассоциации должны быть загружены eagerly или lazily.

## Аннотация @EntityGraph

### Базовая структура

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface EntityGraph {
    String value() default "";           // Имя именованного графа
    String type() default "FETCH";       // Тип графа: FETCH или LOAD
    AttributeNode[] attributePaths() default {}; // Пути к атрибутам
}

@Target({})
@Retention(RetentionPolicy.RUNTIME)
public @interface AttributeNode {
    String value();                      // Имя атрибута
    String[] subgraph() default {};      // Подграфы
}
```

### Типы EntityGraph

```java
public enum EntityGraphType {
    FETCH,  // Загружает указанные атрибуты eagerly, остальные lazily
    LOAD    // Загружает указанные атрибуты eagerly, остальные по умолчанию
}
```

### Базовое использование

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Role> roles;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
    
    // Геттеры и сеттеры
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простой EntityGraph
    @EntityGraph(attributePaths = {"department"})
    List<User> findByActiveTrue();
    
    // EntityGraph с множественными атрибутами
    @EntityGraph(attributePaths = {"department", "roles"})
    List<User> findByDepartmentName(String departmentName);
    
    // EntityGraph с подграфами
    @EntityGraph(attributePaths = {
        "department",
        "roles",
        "orders"
    })
    List<User> findAllWithDetails();
    
    // EntityGraph с типом LOAD
    @EntityGraph(type = EntityGraphType.LOAD, attributePaths = {"department"})
    List<User> findByAgeGreaterThan(int age);
}
```

## Именованные графы @NamedEntityGraph

### Определение именованных графов

```java
@Entity
@NamedEntityGraphs({
    @NamedEntityGraph(
        name = "User.withDepartment",
        attributeNodes = {
            @NamedAttributeNode("department")
        }
    ),
    @NamedEntityGraph(
        name = "User.withDepartmentAndRoles",
        attributeNodes = {
            @NamedAttributeNode("department"),
            @NamedAttributeNode("roles")
        }
    ),
    @NamedEntityGraph(
        name = "User.withAllDetails",
        attributeNodes = {
            @NamedAttributeNode("department"),
            @NamedAttributeNode("roles"),
            @NamedAttributeNode("orders")
        }
    ),
    @NamedEntityGraph(
        name = "User.withDepartmentAndSubgraphs",
        attributeNodes = {
            @NamedAttributeNode("department"),
            @NamedAttributeNode(value = "roles", subgraph = "rolesSubgraph")
        },
        subgraphs = {
            @NamedSubgraph(
                name = "rolesSubgraph",
                attributeNodes = {
                    @NamedAttributeNode("permissions")
                }
            )
        }
    )
})
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Role> roles;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
    
    // Геттеры и сеттеры
}

@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    private List<User> users;
    
    // Геттеры и сеттеры
}

@Entity
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    @OneToMany(mappedBy = "role", fetch = FetchType.LAZY)
    private List<Permission> permissions;
    
    // Геттеры и сеттеры
}
```

### Использование именованных графов

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Использование именованного графа
    @EntityGraph("User.withDepartment")
    List<User> findByActiveTrue();
    
    // Использование именованного графа с параметрами
    @EntityGraph("User.withDepartmentAndRoles")
    List<User> findByDepartmentName(String departmentName);
    
    // Использование сложного именованного графа
    @EntityGraph("User.withDepartmentAndSubgraphs")
    List<User> findAllWithDetails();
    
    // Комбинирование с @Query
    @Query("SELECT u FROM User u WHERE u.age > :minAge")
    @EntityGraph("User.withDepartment")
    List<User> findUsersByAge(@Param("minAge") int minAge);
    
    // Использование с пагинацией
    @EntityGraph("User.withDepartmentAndRoles")
    Page<User> findByActiveTrue(Pageable pageable);
    
    // Использование с Slice
    @EntityGraph("User.withDepartment")
    Slice<User> findByDepartmentName(String departmentName, Pageable pageable);
}
```

## Свойство attributePaths в @EntityGraph

### Базовое использование attributePaths

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простой attributePaths
    @EntityGraph(attributePaths = {"department"})
    List<User> findByActiveTrue();
    
    // Множественные attributePaths
    @EntityGraph(attributePaths = {"department", "roles"})
    List<User> findByDepartmentName(String departmentName);
    
    // Вложенные attributePaths
    @EntityGraph(attributePaths = {
        "department",
        "roles",
        "roles.permissions"
    })
    List<User> findAllWithNestedDetails();
    
    // Глубоко вложенные attributePaths
    @EntityGraph(attributePaths = {
        "department",
        "department.users",
        "roles",
        "roles.permissions",
        "orders",
        "orders.items"
    })
    List<User> findAllWithDeepNesting();
}
```

### Сложные сценарии с attributePaths

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Role> roles;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Address> addresses;
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_groups",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "group_id")
    )
    private List<Group> groups;
    
    // Геттеры и сеттеры
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Селективная загрузка атрибутов
    @EntityGraph(attributePaths = {
        "department",
        "roles"
    })
    List<User> findUsersWithDepartmentAndRoles();
    
    // Загрузка с коллекциями
    @EntityGraph(attributePaths = {
        "roles",
        "orders",
        "addresses"
    })
    List<User> findUsersWithCollections();
    
    // Загрузка с ManyToMany
    @EntityGraph(attributePaths = {
        "groups",
        "roles"
    })
    List<User> findUsersWithGroups();
    
    // Комбинированная загрузка
    @EntityGraph(attributePaths = {
        "department",
        "roles",
        "orders",
        "addresses",
        "groups"
    })
    List<User> findAllWithAllAssociations();
}
```

### Динамическое создание EntityGraph

```java
@Service
public class DynamicEntityGraphService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> findUsersWithDynamicGraph(Set<String> attributes) {
        // Создание динамического EntityGraph
        EntityGraph<User> entityGraph = EntityGraphs.create("User", attributes);
        
        // Использование в запросе
        return userRepository.findAll(entityGraph);
    }
    
    public List<User> findUsersWithConditionalGraph(boolean includeDepartment, 
                                                   boolean includeRoles, 
                                                   boolean includeOrders) {
        List<String> attributePaths = new ArrayList<>();
        
        if (includeDepartment) {
            attributePaths.add("department");
        }
        if (includeRoles) {
            attributePaths.add("roles");
        }
        if (includeOrders) {
            attributePaths.add("orders");
        }
        
        // Создание EntityGraph с условными атрибутами
        EntityGraph<User> entityGraph = EntityGraphs.create("User", attributePaths);
        
        return userRepository.findAll(entityGraph);
    }
}

// Утилитный класс для создания EntityGraph
public class EntityGraphs {
    
    public static <T> EntityGraph<T> create(String entityName, Collection<String> attributePaths) {
        // Логика создания EntityGraph
        return null; // Упрощенная реализация
    }
}
```

## Конфликт Pageable при получении EAGER связей

### Проблема конфликта

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Проблема: EntityGraph может конфликтовать с пагинацией
    @EntityGraph(attributePaths = {"department", "roles"})
    Page<User> findByActiveTrue(Pageable pageable);
    
    // Проблема: Сложные графы могут замедлить пагинацию
    @EntityGraph(attributePaths = {
        "department",
        "roles",
        "orders",
        "addresses"
    })
    Page<User> findByDepartmentName(String departmentName, Pageable pageable);
}
```

### Решения проблемы конфликта

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Решение 1: Использование @QueryHints для оптимизации
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "50"),
        @QueryHint(name = HINT_CACHEABLE, value = "true")
    })
    @EntityGraph(attributePaths = {"department", "roles"})
    Page<User> findByActiveTrueOptimized(Pageable pageable);
    
    // Решение 2: Разделение запросов
    @EntityGraph(attributePaths = {"department"})
    Page<User> findByActiveTrueWithDepartment(Pageable pageable);
    
    @EntityGraph(attributePaths = {"roles"})
    Page<User> findByActiveTrueWithRoles(Pageable pageable);
    
    // Решение 3: Использование нативных запросов
    @Query(value = "SELECT DISTINCT u.* FROM users u " +
                   "LEFT JOIN departments d ON u.department_id = d.id " +
                   "LEFT JOIN user_roles ur ON u.id = ur.user_id " +
                   "LEFT JOIN roles r ON ur.role_id = r.id " +
                   "WHERE u.active = true",
           countQuery = "SELECT COUNT(DISTINCT u.id) FROM users u " +
                       "WHERE u.active = true",
           nativeQuery = true)
    Page<User> findByActiveTrueNative(Pageable pageable);
    
    // Решение 4: Использование @Query с JOIN FETCH
    @Query("SELECT DISTINCT u FROM User u " +
           "LEFT JOIN FETCH u.department " +
           "LEFT JOIN FETCH u.roles " +
           "WHERE u.active = true")
    Page<User> findByActiveTrueWithJoinFetch(Pageable pageable);
}
```

### Оптимизация производительности

```java
@Service
public class OptimizedUserService {
    
    @Autowired
    private UserRepository userRepository;
    
    // Оптимизированный метод с мониторингом производительности
    public Page<User> getOptimizedUsers(int page, int size) {
        long startTime = System.currentTimeMillis();
        
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        Page<User> result = userRepository.findByActiveTrueOptimized(pageable);
        
        long endTime = System.currentTimeMillis();
        System.out.println("Время выполнения: " + (endTime - startTime) + "ms");
        
        return result;
    }
    
    // Метод с условной загрузкой ассоциаций
    public Page<User> getUsersWithConditionalGraph(int page, int size, boolean includeDetails) {
        Pageable pageable = PageRequest.of(page, size);
        
        if (includeDetails) {
            return userRepository.findByActiveTrueWithDepartment(pageable);
        } else {
            return userRepository.findByActiveTrue(pageable);
        }
    }
    
    // Метод с кэшированием
    @Cacheable(value = "users", key = "#page + '_' + #size")
    public Page<User> getCachedUsers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByActiveTrueOptimized(pageable);
    }
}
```

## Сложные сценарии использования

### Многоуровневые EntityGraph

```java
@Entity
@NamedEntityGraphs({
    @NamedEntityGraph(
        name = "User.withDeepNesting",
        attributeNodes = {
            @NamedAttributeNode("department"),
            @NamedAttributeNode(value = "roles", subgraph = "rolesSubgraph"),
            @NamedAttributeNode(value = "orders", subgraph = "ordersSubgraph")
        },
        subgraphs = {
            @NamedSubgraph(
                name = "rolesSubgraph",
                attributeNodes = {
                    @NamedAttributeNode("permissions")
                }
            ),
            @NamedSubgraph(
                name = "ordersSubgraph",
                attributeNodes = {
                    @NamedAttributeNode("items"),
                    @NamedAttributeNode("shipping")
                }
            )
        }
    )
})
public class User {
    // ... существующие поля ...
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
    
    // ... остальные поля ...
}

@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
    
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "shipping_id")
    private Shipping shipping;
    
    // Геттеры и сеттеры
}
```

### Условные EntityGraph

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Базовый метод без EntityGraph
    List<User> findByActiveTrue();
    
    // Метод с минимальным EntityGraph
    @EntityGraph(attributePaths = {"department"})
    List<User> findByActiveTrueWithDepartment();
    
    // Метод с полным EntityGraph
    @EntityGraph(attributePaths = {
        "department",
        "roles",
        "orders",
        "addresses"
    })
    List<User> findByActiveTrueWithAllDetails();
}

@Service
public class ConditionalEntityGraphService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> getUsersWithConditionalGraph(UserGraphType graphType) {
        switch (graphType) {
            case NONE:
                return userRepository.findByActiveTrue();
            case DEPARTMENT:
                return userRepository.findByActiveTrueWithDepartment();
            case ALL_DETAILS:
                return userRepository.findByActiveTrueWithAllDetails();
            default:
                return userRepository.findByActiveTrue();
        }
    }
    
    public enum UserGraphType {
        NONE, DEPARTMENT, ALL_DETAILS
    }
}
```

### EntityGraph с фильтрацией

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // EntityGraph с фильтрацией по связанным сущностям
    @EntityGraph(attributePaths = {"department", "roles"})
    @Query("SELECT u FROM User u " +
           "JOIN u.department d " +
           "WHERE d.name = :departmentName " +
           "AND u.active = true")
    List<User> findActiveUsersByDepartmentWithGraph(@Param("departmentName") String departmentName);
    
    // EntityGraph с пагинацией и фильтрацией
    @EntityGraph(attributePaths = {"department"})
    @Query("SELECT u FROM User u " +
           "JOIN u.department d " +
           "WHERE d.name = :departmentName " +
           "AND u.age >= :minAge")
    Page<User> findUsersByDepartmentAndAge(
        @Param("departmentName") String departmentName,
        @Param("minAge") int minAge,
        Pageable pageable
    );
}
```

## Лучшие практики

### 1. Выбор правильного EntityGraph

```java
@Service
public class EntityGraphBestPractices {
    
    // Хорошо: Минимальный EntityGraph для списков
    @EntityGraph(attributePaths = {"department"})
    public List<User> getUsersForList() {
        return userRepository.findByActiveTrue();
    }
    
    // Хорошо: Полный EntityGraph для детального просмотра
    @EntityGraph(attributePaths = {
        "department",
        "roles",
        "orders"
    })
    public User getUserForDetails(Long userId) {
        return userRepository.findById(userId).orElse(null);
    }
    
    // Плохо: Избыточный EntityGraph для списков
    @EntityGraph(attributePaths = {
        "department",
        "roles",
        "orders",
        "addresses",
        "groups"
    })
    public List<User> getUsersForListBad() {
        return userRepository.findByActiveTrue(); // Избыточная загрузка
    }
}
```

### 2. Оптимизация с пагинацией

```java
@Service
public class PaginationOptimizationService {
    
    @Autowired
    private UserRepository userRepository;
    
    // Хорошо: Минимальный EntityGraph для пагинации
    public Page<User> getUsersForPagination(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        return userRepository.findByActiveTrueWithDepartment(pageable);
    }
    
    // Хорошо: Разделение запросов для сложных случаев
    public UserDetailsDTO getUserDetails(Long userId) {
        User user = userRepository.findById(userId).orElse(null);
        if (user != null) {
            // Дополнительная загрузка при необходимости
            user = userRepository.findByIdWithAllDetails(userId).orElse(user);
        }
        return convertToDTO(user);
    }
}
```

### 3. Мониторинг производительности

```java
@Service
public class EntityGraphPerformanceService {
    
    @Autowired
    private UserRepository userRepository;
    
    public <T> T monitorEntityGraphQuery(Supplier<T> querySupplier, String queryName) {
        long startTime = System.currentTimeMillis();
        
        try {
            T result = querySupplier.get();
            
            long endTime = System.currentTimeMillis();
            long duration = endTime - startTime;
            
            logQueryPerformance(queryName, duration);
            
            return result;
        } catch (Exception e) {
            logQueryError(queryName, e);
            throw e;
        }
    }
    
    private void logQueryPerformance(String queryName, long duration) {
        System.out.printf("EntityGraph запрос '%s' выполнен за %dms%n", queryName, duration);
    }
    
    private void logQueryError(String queryName, Exception e) {
        System.err.printf("Ошибка в EntityGraph запросе '%s': %s%n", queryName, e.getMessage());
    }
}
```

### 4. Кэширование результатов

```java
@Service
public class CachedEntityGraphService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Cacheable(value = "users", key = "#userId")
    public User getUserWithCachedGraph(Long userId) {
        return userRepository.findByIdWithAllDetails(userId).orElse(null);
    }
    
    @Cacheable(value = "userLists", key = "#page + '_' + #size")
    public Page<User> getCachedUsersWithGraph(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByActiveTrueWithDepartment(pageable);
    }
}
```

## Заключение

`@EntityGraph` предоставляет мощные возможности для оптимизации загрузки связанных сущностей в Spring Data JPA. Правильное использование этой аннотации позволяет:

- Избежать проблемы N+1 запросов
- Контролировать загрузку ассоциаций
- Оптимизировать производительность приложений
- Создавать гибкие и масштабируемые решения

Ключевые моменты для успешного использования:

1. **Выбирайте минимальный EntityGraph** для списков и пагинации
2. **Используйте полный EntityGraph** только для детального просмотра
3. **Мониторьте производительность** и оптимизируйте при необходимости
4. **Разделяйте сложные запросы** на более простые при конфликтах с пагинацией
5. **Используйте кэширование** для часто запрашиваемых данных
