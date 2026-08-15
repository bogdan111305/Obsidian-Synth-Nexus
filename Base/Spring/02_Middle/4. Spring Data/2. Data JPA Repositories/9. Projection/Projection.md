# Projection в Spring Data JPA

## Обзор

Projection в Spring Data JPA позволяет выбирать только нужные поля из сущностей, что значительно улучшает производительность и уменьшает объем передаваемых данных. Spring Data JPA поддерживает несколько типов проекций: Class-based, Generic Class-based, Interface-based и проекции с использованием SpEL.

## Class-based Projections

### Базовая структура

Class-based проекции используют обычные классы для определения структуры возвращаемых данных:

```java
// Простая проекция
public class UserSummary {
    private Long id;
    private String name;
    private String email;
    
    // Конструктор
    public UserSummary(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
    
    // Геттеры
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}

// Проекция с вложенными объектами
public class UserWithDepartment {
    private Long id;
    private String name;
    private String email;
    private DepartmentSummary department;
    
    public UserWithDepartment(Long id, String name, String email, DepartmentSummary department) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.department = department;
    }
    
    // Геттеры
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public DepartmentSummary getDepartment() { return department; }
}

public class DepartmentSummary {
    private Long id;
    private String name;
    
    public DepartmentSummary(Long id, String name) {
        this.id = id;
        this.name = name;
    }
    
    // Геттеры
    public Long getId() { return id; }
    public String getName() { return name; }
}
```

### Использование в Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простая проекция
    List<UserSummary> findByActiveTrue();
    
    // Проекция с условием
    List<UserSummary> findByDepartmentName(String departmentName);
    
    // Проекция с пагинацией
    Page<UserSummary> findByActiveTrue(Pageable pageable);
    
    // Проекция с сортировкой
    List<UserSummary> findByActiveTrueOrderByNameAsc();
    
    // Проекция с вложенными объектами
    List<UserWithDepartment> findByDepartmentId(Long departmentId);
    
    // Проекция с @Query
    @Query("SELECT u.id as id, u.name as name, u.email as email " +
           "FROM User u WHERE u.active = true")
    List<UserSummary> findActiveUsersWithProjection();
    
    // Проекция с JOIN
    @Query("SELECT u.id as id, u.name as name, u.email as email, " +
           "d.id as departmentId, d.name as departmentName " +
           "FROM User u JOIN u.department d WHERE u.active = true")
    List<UserWithDepartment> findActiveUsersWithDepartment();
}
```

### Сложные сценарии

```java
// Проекция с вычисляемыми полями
public class UserStatistics {
    private Long id;
    private String name;
    private Long orderCount;
    private BigDecimal totalSpent;
    
    public UserStatistics(Long id, String name, Long orderCount, BigDecimal totalSpent) {
        this.id = id;
        this.name = name;
        this.orderCount = orderCount;
        this.totalSpent = totalSpent;
    }
    
    // Геттеры
    public Long getId() { return id; }
    public String getName() { return name; }
    public Long getOrderCount() { return orderCount; }
    public BigDecimal getTotalSpent() { return totalSpent; }
}

// Проекция с агрегацией
public class DepartmentStatistics {
    private String departmentName;
    private Long userCount;
    private Double averageSalary;
    
    public DepartmentStatistics(String departmentName, Long userCount, Double averageSalary) {
        this.departmentName = departmentName;
        this.userCount = userCount;
        this.averageSalary = averageSalary;
    }
    
    // Геттеры
    public String getDepartmentName() { return departmentName; }
    public Long getUserCount() { return userCount; }
    public Double getAverageSalary() { return averageSalary; }
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Проекция с агрегацией
    @Query("SELECT d.name as departmentName, COUNT(u) as userCount, AVG(u.salary) as averageSalary " +
           "FROM User u JOIN u.department d GROUP BY d.name")
    List<DepartmentStatistics> getDepartmentStatistics();
    
    // Проекция с подзапросом
    @Query("SELECT u.id as id, u.name as name, " +
           "(SELECT COUNT(o) FROM Order o WHERE o.user = u) as orderCount, " +
           "(SELECT SUM(o.total) FROM Order o WHERE o.user = u) as totalSpent " +
           "FROM User u WHERE u.active = true")
    List<UserStatistics> getUserStatistics();
}
```

## Generic Class-based Projections

### Базовая структура

Generic проекции позволяют создавать переиспользуемые классы проекций:

```java
// Базовый класс для проекций
public abstract class BaseProjection<T> {
    private T id;
    private String name;
    
    public BaseProjection(T id, String name) {
        this.id = id;
        this.name = name;
    }
    
    // Геттеры
    public T getId() { return id; }
    public String getName() { return name; }
}

// Специализированные проекции
public class UserProjection extends BaseProjection<Long> {
    private String email;
    
    public UserProjection(Long id, String name, String email) {
        super(id, name);
        this.email = email;
    }
    
    public String getEmail() { return email; }
}

public class DepartmentProjection extends BaseProjection<Long> {
    private String location;
    
    public DepartmentProjection(Long id, String name, String location) {
        super(id, name);
        this.location = location;
    }
    
    public String getLocation() { return location; }
}

// Generic проекция с типами
public class GenericProjection<T, R> {
    private T id;
    private String name;
    private R additionalData;
    
    public GenericProjection(T id, String name, R additionalData) {
        this.id = id;
        this.name = name;
        this.additionalData = additionalData;
    }
    
    // Геттеры
    public T getId() { return id; }
    public String getName() { return name; }
    public R getAdditionalData() { return additionalData; }
}
```

### Использование в Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Использование специализированных проекций
    List<UserProjection> findByActiveTrue();
    
    List<DepartmentProjection> findDepartmentsByActiveUsers();
    
    // Использование generic проекций
    @Query("SELECT u.id as id, u.name as name, u.email as additionalData " +
           "FROM User u WHERE u.active = true")
    List<GenericProjection<Long, String>> findUsersAsGeneric();
    
    @Query("SELECT d.id as id, d.name as name, d.location as additionalData " +
           "FROM Department d")
    List<GenericProjection<Long, String>> findDepartmentsAsGeneric();
}
```

### Сложные generic проекции

```java
// Generic проекция с множественными типами
public class MultiTypeProjection<T, U, V> {
    private T id;
    private U primaryData;
    private V secondaryData;
    
    public MultiTypeProjection(T id, U primaryData, V secondaryData) {
        this.id = id;
        this.primaryData = primaryData;
        this.secondaryData = secondaryData;
    }
    
    // Геттеры
    public T getId() { return id; }
    public U getPrimaryData() { return primaryData; }
    public V getSecondaryData() { return secondaryData; }
}

// Generic проекция с коллекциями
public class CollectionProjection<T, E> {
    private T id;
    private String name;
    private Collection<E> items;
    
    public CollectionProjection(T id, String name, Collection<E> items) {
        this.id = id;
        this.name = name;
        this.items = items;
    }
    
    // Геттеры
    public T getId() { return id; }
    public String getName() { return name; }
    public Collection<E> getItems() { return items; }
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Multi-type проекция
    @Query("SELECT u.id as id, u.name as primaryData, u.email as secondaryData " +
           "FROM User u WHERE u.active = true")
    List<MultiTypeProjection<Long, String, String>> findUsersAsMultiType();
    
    // Collection проекция
    @Query("SELECT u.id as id, u.name as name, u.roles as items " +
           "FROM User u WHERE u.active = true")
    List<CollectionProjection<Long, Role>> findUsersWithRoles();
}
```

## Interface-based Projections

### Базовая структура

Interface-based проекции используют интерфейсы для определения структуры данных:

```java
// Простая интерфейсная проекция
public interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
}

// Проекция с вложенными интерфейсами
public interface UserWithDepartment {
    Long getId();
    String getName();
    String getEmail();
    DepartmentSummary getDepartment();
}

public interface DepartmentSummary {
    Long getId();
    String getName();
}

// Проекция с вычисляемыми свойствами
public interface UserStatistics {
    Long getId();
    String getName();
    Long getOrderCount();
    BigDecimal getTotalSpent();
    
    // Вычисляемое свойство
    default String getDisplayName() {
        return getName() + " (ID: " + getId() + ")";
    }
    
    // Вычисляемое свойство с условием
    default boolean isHighSpender() {
        return getTotalSpent() != null && getTotalSpent().compareTo(new BigDecimal("1000")) > 0;
    }
}
```

### Использование в Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простая интерфейсная проекция
    List<UserSummary> findByActiveTrue();
    
    // Проекция с условием
    List<UserSummary> findByDepartmentName(String departmentName);
    
    // Проекция с пагинацией
    Page<UserSummary> findByActiveTrue(Pageable pageable);
    
    // Проекция с вложенными объектами
    List<UserWithDepartment> findByDepartmentId(Long departmentId);
    
    // Проекция с @Query
    @Query("SELECT u.id as id, u.name as name, u.email as email " +
           "FROM User u WHERE u.active = true")
    List<UserSummary> findActiveUsersWithProjection();
    
    // Проекция с JOIN
    @Query("SELECT u.id as id, u.name as name, u.email as email, " +
           "d.id as departmentId, d.name as departmentName " +
           "FROM User u JOIN u.department d WHERE u.active = true")
    List<UserWithDepartment> findActiveUsersWithDepartment();
    
    // Проекция с агрегацией
    @Query("SELECT u.id as id, u.name as name, " +
           "(SELECT COUNT(o) FROM Order o WHERE o.user = u) as orderCount, " +
           "(SELECT SUM(o.total) FROM Order o WHERE o.user = u) as totalSpent " +
           "FROM User u WHERE u.active = true")
    List<UserStatistics> getUserStatistics();
}
```

### Сложные интерфейсные проекции

```java
// Проекция с generic типами
public interface GenericProjection<T> {
    T getId();
    String getName();
}

// Проекция с множественными интерфейсами
public interface UserDetails extends UserSummary, UserStatistics {
    String getPhoneNumber();
    LocalDate getRegistrationDate();
    
    // Вычисляемое свойство
    default long getDaysSinceRegistration() {
        return getRegistrationDate() != null ? 
            ChronoUnit.DAYS.between(getRegistrationDate(), LocalDate.now()) : 0;
    }
}

// Проекция с условными свойствами
public interface ConditionalUserProjection {
    Long getId();
    String getName();
    String getEmail();
    boolean isActive();
    
    // Условное свойство
    default String getStatus() {
        return isActive() ? "Active" : "Inactive";
    }
    
    // Условное свойство с форматированием
    default String getFormattedEmail() {
        return getEmail() != null ? getEmail().toLowerCase() : "N/A";
    }
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Generic проекция
    List<GenericProjection<Long>> findUsersAsGeneric();
    
    // Множественная проекция
    List<UserDetails> findUserDetails();
    
    // Условная проекция
    List<ConditionalUserProjection> findUsersWithConditionalProperties();
}
```

## SpEL в Projections

### Базовая структура

SpEL (Spring Expression Language) позволяет создавать динамические проекции:

```java
// Проекция с SpEL
public interface UserSpELProjection {
    Long getId();
    String getName();
    String getEmail();
    
    // SpEL выражение
    @Value("#{target.department?.name ?: 'No Department'}")
    String getDepartmentName();
    
    // SpEL с вычислениями
    @Value("#{target.age >= 18 ? 'Adult' : 'Minor'}")
    String getAgeCategory();
    
    // SpEL с форматированием
    @Value("#{target.name + ' (' + target.email + ')'}")
    String getDisplayName();
    
    // SpEL с условной логикой
    @Value("#{target.salary > 50000 ? 'High' : target.salary > 30000 ? 'Medium' : 'Low'}")
    String getSalaryCategory();
}
```

### Сложные SpEL выражения

```java
// Проекция с сложными SpEL выражениями
public interface AdvancedSpELProjection {
    Long getId();
    String getName();
    String getEmail();
    
    // SpEL с коллекциями
    @Value("#{target.roles?.size() ?: 0}")
    Integer getRoleCount();
    
    // SpEL с агрегацией
    @Value("#{target.orders?.stream()?.mapToDouble(o -> o.total)?.sum() ?: 0.0}")
    Double getTotalSpent();
    
    // SpEL с условной логикой
    @Value("#{target.lastLoginDate != null ? " +
           "T(java.time.temporal.ChronoUnit.DAYS).between(target.lastLoginDate, T(java.time.LocalDateTime).now()) : -1}")
    Long getDaysSinceLastLogin();
    
    // SpEL с форматированием даты
    @Value("#{target.createdDate != null ? " +
           "T(java.time.format.DateTimeFormatter).ofPattern('yyyy-MM-dd').format(target.createdDate) : 'N/A'}")
    String getFormattedCreatedDate();
    
    // SpEL с проверкой null
    @Value("#{target.department != null ? target.department.name : 'Unknown'}")
    String getDepartmentName();
    
    // SpEL с математическими операциями
    @Value("#{target.salary != null ? target.salary * 12 : 0}")
    BigDecimal getAnnualSalary();
}
```

### Использование в Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простая SpEL проекция
    List<UserSpELProjection> findByActiveTrue();
    
    // SpEL проекция с условием
    List<UserSpELProjection> findByDepartmentName(String departmentName);
    
    // SpEL проекция с @Query
    @Query("SELECT u FROM User u WHERE u.active = true")
    List<UserSpELProjection> findActiveUsersWithSpEL();
    
    // Сложная SpEL проекция
    List<AdvancedSpELProjection> findUsersWithAdvancedSpEL();
    
    // SpEL проекция с пагинацией
    Page<UserSpELProjection> findByActiveTrue(Pageable pageable);
}
```

### Динамические SpEL выражения

```java
// Проекция с динамическими SpEL
public interface DynamicSpELProjection {
    Long getId();
    String getName();
    
    // Динамическое SpEL выражение
    @Value("#{@userService.calculateUserScore(target)}")
    Integer getUserScore();
    
    // SpEL с вызовом методов
    @Value("#{@emailService.formatEmail(target.email)}")
    String getFormattedEmail();
    
    // SpEL с условной логикой
    @Value("#{@userService.isUserEligibleForPromotion(target) ? 'Eligible' : 'Not Eligible'}")
    String getPromotionStatus();
}

@Service
public class UserService {
    
    public Integer calculateUserScore(User user) {
        int score = 0;
        if (user.getActive()) score += 10;
        if (user.getOrders() != null) score += user.getOrders().size() * 5;
        if (user.getLastLoginDate() != null) {
            long daysSinceLogin = ChronoUnit.DAYS.between(user.getLastLoginDate(), LocalDateTime.now());
            if (daysSinceLogin <= 7) score += 20;
        }
        return score;
    }
    
    public boolean isUserEligibleForPromotion(User user) {
        return user.getActive() && 
               user.getSalary() != null && 
               user.getSalary().compareTo(new BigDecimal("50000")) < 0;
    }
}

@Service
public class EmailService {
    
    public String formatEmail(String email) {
        return email != null ? email.toLowerCase().trim() : "N/A";
    }
}
```

## Лучшие практики

### 1. Выбор правильного типа проекции

```java
@Service
public class ProjectionBestPractices {
    
    @Autowired
    private UserRepository userRepository;
    
    // Хорошо: Interface-based для простых случаев
    public List<UserSummary> getUsersForList() {
        return userRepository.findByActiveTrue();
    }
    
    // Хорошо: Class-based для сложных вычислений
    public List<UserStatistics> getUserStatistics() {
        return userRepository.getUserStatistics();
    }
    
    // Хорошо: SpEL для динамических вычислений
    public List<UserSpELProjection> getUsersWithDynamicData() {
        return userRepository.findByActiveTrue();
    }
    
    // Плохо: Избыточная проекция
    public List<User> getUsersBad() {
        return userRepository.findAll(); // Загружает все поля
    }
}
```

### 2. Оптимизация производительности

```java
@Service
public class ProjectionPerformanceService {
    
    @Autowired
    private UserRepository userRepository;
    
    // Хорошо: Минимальная проекция для списков
    public Page<UserSummary> getUsersForPagination(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByActiveTrue(pageable);
    }
    
    // Хорошо: Полная проекция для детального просмотра
    public UserDetails getUserForDetails(Long userId) {
        return userRepository.findUserDetailsById(userId).orElse(null);
    }
    
    // Хорошо: Специализированная проекция для отчетов
    public List<UserStatistics> getUsersForReport() {
        return userRepository.getUserStatistics();
    }
    
    // Мониторинг производительности
    public <T> T monitorProjectionQuery(Supplier<T> querySupplier, String queryName) {
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
        System.out.printf("Projection query '%s' выполнен за %dms%n", queryName, duration);
    }
    
    private void logQueryError(String queryName, Exception e) {
        System.err.printf("Ошибка в projection query '%s': %s%n", queryName, e.getMessage());
    }
}
```

### 3. Кэширование проекций

```java
@Service
public class CachedProjectionService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Cacheable(value = "userSummaries", key = "#page + '_' + #size")
    public Page<UserSummary> getCachedUserSummaries(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByActiveTrue(pageable);
    }
    
    @Cacheable(value = "userStatistics", key = "'all'")
    public List<UserStatistics> getCachedUserStatistics() {
        return userRepository.getUserStatistics();
    }
    
    @Cacheable(value = "userSpEL", key = "#userId")
    public UserSpELProjection getCachedUserSpEL(Long userId) {
        return userRepository.findUserSpELById(userId).orElse(null);
    }
}
```

### 4. Валидация проекций

```java
@Service
public class ProjectionValidationService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<UserSummary> getValidatedUserSummaries() {
        List<UserSummary> users = userRepository.findByActiveTrue();
        
        // Валидация проекций
        users.forEach(this::validateUserSummary);
        
        return users;
    }
    
    private void validateUserSummary(UserSummary user) {
        if (user.getId() == null) {
            throw new IllegalArgumentException("User ID cannot be null");
        }
        if (user.getName() == null || user.getName().trim().isEmpty()) {
            throw new IllegalArgumentException("User name cannot be null or empty");
        }
        if (user.getEmail() != null && !user.getEmail().contains("@")) {
            throw new IllegalArgumentException("Invalid email format");
        }
    }
}
```

## Заключение

Projection в Spring Data JPA предоставляет мощные возможности для оптимизации запросов и улучшения производительности. Правильное использование различных типов проекций позволяет:

- Уменьшить объем передаваемых данных
- Улучшить производительность приложений
- Создавать гибкие и переиспользуемые структуры данных
- Оптимизировать работу с большими наборами данных

Ключевые моменты для успешного использования:

1. **Выбирайте правильный тип проекции** в зависимости от сценария
2. **Используйте Interface-based проекции** для простых случаев
3. **Применяйте Class-based проекции** для сложных вычислений
4. **Используйте SpEL** для динамических вычислений
5. **Мониторьте производительность** и оптимизируйте при необходимости
6. **Кэшируйте результаты** для часто запрашиваемых данных
