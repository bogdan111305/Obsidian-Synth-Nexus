# Interface-based Projections

## Обзор

Interface-based Projections в Spring Data JPA позволяют создавать проекции данных на основе интерфейсов. Это позволяет загружать только необходимые поля из сущностей, что улучшает производительность и уменьшает объем передаваемых данных.

## Основные возможности

### Типы проекций
- **Closed Projections** - проекции с фиксированным набором свойств
- **Open Projections** - проекции с динамическими свойствами
- **Class-based Projections** - проекции на основе классов
- **Dynamic Projections** - проекции, определяемые во время выполнения

### Структура интерфейса

```java
public interface UserProjection {
    String getEmail();
    String getName();
    String getRole();
}
```

## Closed Projections

### Базовые проекции
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    List<UserProjection> findByActive(boolean active);
    Optional<UserProjection> findByEmail(String email);
    UserProjection findByRole(String role);
}

public interface UserProjection {
    String getEmail();
    String getName();
    String getRole();
    boolean isActive();
}
```

### Проекции с вложенными свойствами
```java
@Entity
public class User {
    @Id
    private Long id;
    private String email;
    private String name;
    
    @ManyToOne
    private Role role;
    
    @OneToOne
    private Profile profile;
}

public interface UserWithRoleProjection {
    String getEmail();
    String getName();
    RoleProjection getRole();
    ProfileProjection getProfile();
}

public interface RoleProjection {
    String getName();
    String getDescription();
}

public interface ProfileProjection {
    String getCity();
    String getCountry();
}
```

### Использование в репозитории
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    List<UserWithRoleProjection> findByActive(boolean active);
    Optional<UserWithRoleProjection> findById(Long id);
    List<UserWithRoleProjection> findByRoleName(String roleName);
}
```

## Open Projections

### Динамические проекции
```java
public interface UserOpenProjection {
    String getEmail();
    String getName();
    
    @Value("#{target.email + ' - ' + target.name}")
    String getDisplayName();
    
    @Value("#{target.role?.name ?: 'No Role'}")
    String getRoleName();
}
```

### Проекции с вычисляемыми полями
```java
public interface UserSummaryProjection {
    String getEmail();
    String getName();
    
    @Value("#{target.orders?.size() ?: 0}")
    int getOrderCount();
    
    @Value("#{target.isActive() ? 'Active' : 'Inactive'}")
    String getStatus();
    
    @Value("#{target.createdDate?.format(T(java.time.format.DateTimeFormatter).ofPattern('yyyy-MM-dd'))}")
    String getFormattedCreatedDate();
}
```

## Сложные проекции

### Проекции с коллекциями
```java
public interface UserWithOrdersProjection {
    String getEmail();
    String getName();
    List<OrderProjection> getOrders();
}

public interface OrderProjection {
    Long getId();
    String getStatus();
    BigDecimal getTotal();
    
    @Value("#{target.items?.size() ?: 0}")
    int getItemCount();
}
```

### Проекции с условной логикой
```java
public interface UserConditionalProjection {
    String getEmail();
    String getName();
    
    @Value("#{target.orders?.stream()?.filter(o -> o.status == 'COMPLETED')?.count() ?: 0}")
    long getCompletedOrderCount();
    
    @Value("#{target.orders?.stream()?.mapToDouble(o -> o.total)?.sum() ?: 0.0}")
    BigDecimal getTotalOrderValue();
}
```

## Проекции с параметрами

### Проекции с дополнительными параметрами
```java
public interface UserWithCustomFieldProjection {
    String getEmail();
    String getName();
    
    @Value("#{args[0] + ' - ' + target.name}")
    String getCustomName(String prefix);
    
    @Value("#{target.email + '@' + args[0]}")
    String getCustomEmail(String domain);
}
```

### Использование в репозитории
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE u.active = ?1")
    List<UserWithCustomFieldProjection> findActiveUsersWithCustomFields(boolean active);
}
```

## Динамические проекции

### Проекции, определяемые во время выполнения
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    <T> List<T> findByActive(boolean active, Class<T> projectionClass);
    <T> Optional<T> findByEmail(String email, Class<T> projectionClass);
    <T> List<T> findByRole(String role, Class<T> projectionClass);
}
```

### Использование динамических проекций
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<UserProjection> getActiveUsers() {
        return userRepository.findByActive(true, UserProjection.class);
    }
    
    public List<UserWithRoleProjection> getActiveUsersWithRole() {
        return userRepository.findByActive(true, UserWithRoleProjection.class);
    }
    
    public List<UserSummaryProjection> getActiveUsersSummary() {
        return userRepository.findByActive(true, UserSummaryProjection.class);
    }
}
```

## Проекции с пагинацией

### Проекции в Page
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Page<UserProjection> findByActive(boolean active, Pageable pageable);
    Slice<UserWithRoleProjection> findByRole(String role, Pageable pageable);
}
```

### Использование с пагинацией
```java
@Service
public class UserService {
    
    public Page<UserProjection> getActiveUsers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("email"));
        return userRepository.findByActive(true, pageable);
    }
    
    public Slice<UserWithRoleProjection> getUsersByRole(String role, int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByRole(role, pageable);
    }
}
```

## Проекции с сортировкой

### Проекции с Sort
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    List<UserProjection> findByActive(boolean active, Sort sort);
    List<UserWithRoleProjection> findByRole(String role, Sort sort);
}
```

### Использование сортировки
```java
@Service
public class UserService {
    
    public List<UserProjection> getActiveUsersSorted() {
        Sort sort = Sort.by("email").ascending().and(Sort.by("name").ascending());
        return userRepository.findByActive(true, sort);
    }
}
```

## Производительность

### Оптимизация запросов
```java
// Хорошо - проекция загружает только необходимые поля
@Query("SELECT u.email, u.name, u.role FROM User u WHERE u.active = ?1")
List<UserProjection> findActiveUsers(boolean active);

// Плохо - загружает всю сущность
List<User> findActiveUsers(boolean active);
```

### Сравнение производительности
```java
@Component
public class ProjectionPerformanceMonitor {
    
    public void comparePerformance() {
        long startTime = System.currentTimeMillis();
        
        // С проекцией
        List<UserProjection> projections = userRepository.findByActive(true);
        long projectionTime = System.currentTimeMillis() - startTime;
        
        startTime = System.currentTimeMillis();
        
        // Без проекции
        List<User> entities = userRepository.findByActive(true);
        long entityTime = System.currentTimeMillis() - startTime;
        
        System.out.println("Projection time: " + projectionTime + "ms");
        System.out.println("Entity time: " + entityTime + "ms");
    }
}
```

## Лучшие практики

### Структурирование проекций
```java
// Базовые проекции
public interface UserBasicProjection {
    String getEmail();
    String getName();
}

// Расширенные проекции
public interface UserExtendedProjection extends UserBasicProjection {
    String getRole();
    boolean isActive();
}

// Полные проекции
public interface UserFullProjection extends UserExtendedProjection {
    RoleProjection getRole();
    ProfileProjection getProfile();
}
```

### Использование в сервисах
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<UserBasicProjection> getUsersForList() {
        // Минимальные данные для списка
        return userRepository.findByActive(true);
    }
    
    public List<UserExtendedProjection> getUsersForGrid() {
        // Расширенные данные для таблицы
        return userRepository.findByActive(true);
    }
    
    public UserFullProjection getUserForDetail(Long id) {
        // Полные данные для детальной страницы
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

### Валидация проекций
```java
@Component
public class ProjectionValidator {
    
    public void validateProjection(Object projection) {
        if (projection == null) {
            throw new IllegalArgumentException("Projection cannot be null");
        }
        
        // Проверка наличия необходимых методов
        Method[] methods = projection.getClass().getMethods();
        boolean hasRequiredMethods = Arrays.stream(methods)
            .anyMatch(method -> method.getName().equals("getEmail"));
        
        if (!hasRequiredMethods) {
            throw new IllegalArgumentException("Projection must have required methods");
        }
    }
}
```

## Отладка и логирование

### Включение логирования
```properties
logging.level.org.springframework.data.jpa.repository.query=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### Отладочная информация
```java
@Component
public class ProjectionDebugger {
    
    public void debugProjection(String methodName, List<?> projections) {
        System.out.println("Method: " + methodName);
        System.out.println("Projections count: " + projections.size());
        
        if (!projections.isEmpty()) {
            Object first = projections.get(0);
            System.out.println("Projection class: " + first.getClass().getName());
            
            // Вызов методов проекции для проверки
            if (first instanceof UserProjection) {
                UserProjection projection = (UserProjection) first;
                System.out.println("Email: " + projection.getEmail());
                System.out.println("Name: " + projection.getName());
            }
        }
    }
}
```

## Лучшие практики

1. **Используйте проекции** для загрузки только необходимых данных
2. **Создавайте иерархию проекций** для разных сценариев использования
3. **Применяйте Open Projections** для вычисляемых полей
4. **Используйте динамические проекции** для гибкости
5. **Мониторьте производительность** проекций
6. **Тестируйте проекции** в unit-тестах
7. **Документируйте сложные проекции** для понимания их назначения 