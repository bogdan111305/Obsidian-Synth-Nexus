# JPA Auditing в Spring Data JPA

## Обзор

JPA Auditing позволяет автоматически отслеживать изменения в сущностях, такие как время создания, время последнего обновления, кто создал и кто последний раз обновил запись. Это особенно полезно для аудита, отладки и соблюдения требований к безопасности.

## Создание AuditingEntity

### Базовая структура AuditingEntity

```java
// Базовый класс для аудируемых сущностей
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
    
    // Геттеры и сеттеры
    public LocalDateTime getCreatedDate() { return createdDate; }
    public void setCreatedDate(LocalDateTime createdDate) { this.createdDate = createdDate; }
    
    public LocalDateTime getLastModifiedDate() { return lastModifiedDate; }
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) { this.lastModifiedDate = lastModifiedDate; }
    
    public String getCreatedBy() { return createdBy; }
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
    
    public String getLastModifiedBy() { return lastModifiedBy; }
    public void setLastModifiedBy(String lastModifiedBy) { this.lastModifiedBy = lastModifiedBy; }
}

// Альтернативная версия с использованием @Embeddable
@Embeddable
public class AuditInfo {
    
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
    
    // Геттеры и сеттеры
    public LocalDateTime getCreatedDate() { return createdDate; }
    public void setCreatedDate(LocalDateTime createdDate) { this.createdDate = createdDate; }
    
    public LocalDateTime getLastModifiedDate() { return lastModifiedDate; }
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) { this.lastModifiedDate = lastModifiedDate; }
    
    public String getCreatedBy() { return createdBy; }
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
    
    public String getLastModifiedBy() { return lastModifiedBy; }
    public void setLastModifiedBy(String lastModifiedBy) { this.lastModifiedBy = lastModifiedBy; }
}
```

### Использование в сущностях

```java
// Сущность с наследованием от AuditingEntity
@Entity
@Table(name = "users")
public class User extends AuditingEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "email", unique = true, nullable = false)
    private String email;
    
    @Column(name = "active")
    private Boolean active = true;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public Boolean getActive() { return active; }
    public void setActive(Boolean active) { this.active = active; }
    
    public Department getDepartment() { return department; }
    public void setDepartment(Department department) { this.department = department; }
}

// Сущность с использованием @Embedded
@Entity
@Table(name = "departments")
@EntityListeners(AuditingEntityListener.class)
public class Department {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "location")
    private String location;
    
    @Embedded
    private AuditInfo auditInfo;
    
    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    private List<User> users;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getLocation() { return location; }
    public void setLocation(String location) { this.location = location; }
    
    public AuditInfo getAuditInfo() { return auditInfo; }
    public void setAuditInfo(AuditInfo auditInfo) { this.auditInfo = auditInfo; }
    
    public List<User> getUsers() { return users; }
    public void setUsers(List<User> users) { this.users = users; }
}
```

### Расширенная версия AuditingEntity

```java
// Расширенная версия с дополнительными полями
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class ExtendedAuditingEntity {
    
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
    
    @Version
    @Column(name = "version")
    private Long version;
    
    @Column(name = "deleted")
    private Boolean deleted = false;
    
    @Column(name = "deleted_date")
    private LocalDateTime deletedDate;
    
    @Column(name = "deleted_by")
    private String deletedBy;
    
    // Дополнительные методы
    public boolean isDeleted() {
        return deleted != null && deleted;
    }
    
    public void markAsDeleted(String deletedBy) {
        this.deleted = true;
        this.deletedDate = LocalDateTime.now();
        this.deletedBy = deletedBy;
    }
    
    public void restore() {
        this.deleted = false;
        this.deletedDate = null;
        this.deletedBy = null;
    }
    
    // Геттеры и сеттеры
    public LocalDateTime getCreatedDate() { return createdDate; }
    public void setCreatedDate(LocalDateTime createdDate) { this.createdDate = createdDate; }
    
    public LocalDateTime getLastModifiedDate() { return lastModifiedDate; }
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) { this.lastModifiedDate = lastModifiedDate; }
    
    public String getCreatedBy() { return createdBy; }
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
    
    public String getLastModifiedBy() { return lastModifiedBy; }
    public void setLastModifiedBy(String lastModifiedBy) { this.lastModifiedBy = lastModifiedBy; }
    
    public Long getVersion() { return version; }
    public void setVersion(Long version) { this.version = version; }
    
    public Boolean getDeleted() { return deleted; }
    public void setDeleted(Boolean deleted) { this.deleted = deleted; }
    
    public LocalDateTime getDeletedDate() { return deletedDate; }
    public void setDeletedDate(LocalDateTime deletedDate) { this.deletedDate = deletedDate; }
    
    public String getDeletedBy() { return deletedBy; }
    public void setDeletedBy(String deletedBy) { this.deletedBy = deletedBy; }
}
```

## Аннотация @EnableJpaAuditing

### Базовая конфигурация

```java
@Configuration
@EnableJpaAuditing(
    auditorAwareRef = "auditorAwareImpl",
    dateTimeProviderRef = "dateTimeProvider",
    modifyOnCreate = true,
    setDates = true
)
public class JpaAuditingConfig {
    
    @Bean
    public AuditorAware<String> auditorAwareImpl() {
        return new AuditorAwareImpl();
    }
    
    @Bean
    public DateTimeProvider dateTimeProvider() {
        return new DateTimeProvider() {
            @Override
            public Optional<TemporalAccessor> getNow() {
                return Optional.of(LocalDateTime.now());
            }
        };
    }
}

// Реализация AuditorAware
@Component
public class AuditorAwareImpl implements AuditorAware<String> {
    
    @Override
    public Optional<String> getCurrentAuditor() {
        // Получение текущего пользователя из контекста безопасности
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication == null || 
            authentication.getPrincipal().equals("anonymousUser") ||
            !authentication.isAuthenticated()) {
            return Optional.of("system");
        }
        
        return Optional.of(authentication.getName());
    }
}
```

### Расширенная конфигурация

```java
@Configuration
@EnableJpaAuditing(
    auditorAwareRef = "customAuditorAware",
    dateTimeProviderRef = "customDateTimeProvider",
    modifyOnCreate = true,
    setDates = true
)
public class AdvancedJpaAuditingConfig {
    
    @Bean
    public AuditorAware<String> customAuditorAware() {
        return new CustomAuditorAware();
    }
    
    @Bean
    public DateTimeProvider customDateTimeProvider() {
        return new CustomDateTimeProvider();
    }
    
    @Bean
    public DateTimeFormatter dateTimeFormatter() {
        return DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    }
}

// Кастомная реализация AuditorAware
@Component
public class CustomAuditorAware implements AuditorAware<String> {
    
    @Autowired
    private HttpServletRequest request;
    
    @Override
    public Optional<String> getCurrentAuditor() {
        // Получение пользователя из различных источников
        String auditor = getAuditorFromRequest();
        
        if (auditor == null) {
            auditor = getAuditorFromSecurityContext();
        }
        
        if (auditor == null) {
            auditor = getAuditorFromThreadLocal();
        }
        
        return Optional.ofNullable(auditor).or(() -> Optional.of("system"));
    }
    
    private String getAuditorFromRequest() {
        if (request != null) {
            return request.getHeader("X-User-Id");
        }
        return null;
    }
    
    private String getAuditorFromSecurityContext() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication != null && authentication.isAuthenticated() && 
            !authentication.getPrincipal().equals("anonymousUser")) {
            return authentication.getName();
        }
        
        return null;
    }
    
    private String getAuditorFromThreadLocal() {
        return AuditorContext.getCurrentAuditor();
    }
}

// Кастомная реализация DateTimeProvider
@Component
public class CustomDateTimeProvider implements DateTimeProvider {
    
    @Override
    public Optional<TemporalAccessor> getNow() {
        // Использование UTC времени
        return Optional.of(LocalDateTime.now(ZoneOffset.UTC));
    }
}

// ThreadLocal для хранения текущего аудитора
public class AuditorContext {
    private static final ThreadLocal<String> currentAuditor = new ThreadLocal<>();
    
    public static void setCurrentAuditor(String auditor) {
        currentAuditor.set(auditor);
    }
    
    public static String getCurrentAuditor() {
        return currentAuditor.get();
    }
    
    public static void clear() {
        currentAuditor.remove();
    }
}
```

## Тестирование @CreatedDate и @LastModifiedDate

### Базовые тесты

```java
@SpringBootTest
@Transactional
class AuditingTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private DepartmentRepository departmentRepository;
    
    @Test
    void testCreatedDateAndLastModifiedDate() {
        // Создание пользователя
        User user = new User();
        user.setName("John Doe");
        user.setEmail("john@example.com");
        
        // Проверка, что даты не установлены до сохранения
        assertThat(user.getCreatedDate()).isNull();
        assertThat(user.getLastModifiedDate()).isNull();
        
        // Сохранение пользователя
        User savedUser = userRepository.save(user);
        
        // Проверка, что даты установлены после сохранения
        assertThat(savedUser.getCreatedDate()).isNotNull();
        assertThat(savedUser.getLastModifiedDate()).isNotNull();
        assertThat(savedUser.getCreatedDate()).isEqualTo(savedUser.getLastModifiedDate());
        
        // Обновление пользователя
        savedUser.setName("John Smith");
        User updatedUser = userRepository.save(savedUser);
        
        // Проверка, что createdDate не изменился, а lastModifiedDate обновился
        assertThat(updatedUser.getCreatedDate()).isEqualTo(savedUser.getCreatedDate());
        assertThat(updatedUser.getLastModifiedDate()).isAfter(savedUser.getLastModifiedDate());
    }
    
    @Test
    void testCreatedByAndLastModifiedBy() {
        // Установка текущего аудитора
        AuditorContext.setCurrentAuditor("testUser");
        
        try {
            // Создание пользователя
            User user = new User();
            user.setName("Jane Doe");
            user.setEmail("jane@example.com");
            
            User savedUser = userRepository.save(user);
            
            // Проверка, что аудитор установлен
            assertThat(savedUser.getCreatedBy()).isEqualTo("testUser");
            assertThat(savedUser.getLastModifiedBy()).isEqualTo("testUser");
            
            // Обновление пользователя
            savedUser.setName("Jane Smith");
            User updatedUser = userRepository.save(savedUser);
            
            // Проверка, что createdBy не изменился, а lastModifiedBy обновился
            assertThat(updatedUser.getCreatedBy()).isEqualTo("testUser");
            assertThat(updatedUser.getLastModifiedBy()).isEqualTo("testUser");
            
        } finally {
            AuditorContext.clear();
        }
    }
    
    @Test
    void testAuditingWithEmbedded() {
        // Создание отдела с встроенной аудиторской информацией
        Department department = new Department();
        department.setName("IT Department");
        department.setLocation("Building A");
        
        Department savedDepartment = departmentRepository.save(department);
        
        // Проверка, что аудиторская информация установлена
        assertThat(savedDepartment.getAuditInfo()).isNotNull();
        assertThat(savedDepartment.getAuditInfo().getCreatedDate()).isNotNull();
        assertThat(savedDepartment.getAuditInfo().getLastModifiedDate()).isNotNull();
    }
}
```

### Расширенные тесты

```java
@SpringBootTest
@Transactional
class ExtendedAuditingTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testExtendedAuditingEntity() {
        // Создание пользователя с расширенной аудиторской информацией
        ExtendedUser user = new ExtendedUser();
        user.setName("Test User");
        user.setEmail("test@example.com");
        
        ExtendedUser savedUser = userRepository.save(user);
        
        // Проверка базовых полей аудита
        assertThat(savedUser.getCreatedDate()).isNotNull();
        assertThat(savedUser.getLastModifiedDate()).isNotNull();
        assertThat(savedUser.getVersion()).isEqualTo(0L);
        assertThat(savedUser.isDeleted()).isFalse();
        
        // Проверка soft delete
        savedUser.markAsDeleted("admin");
        ExtendedUser deletedUser = userRepository.save(savedUser);
        
        assertThat(deletedUser.isDeleted()).isTrue();
        assertThat(deletedUser.getDeletedDate()).isNotNull();
        assertThat(deletedUser.getDeletedBy()).isEqualTo("admin");
        
        // Проверка восстановления
        deletedUser.restore();
        ExtendedUser restoredUser = userRepository.save(deletedUser);
        
        assertThat(restoredUser.isDeleted()).isFalse();
        assertThat(restoredUser.getDeletedDate()).isNull();
        assertThat(restoredUser.getDeletedBy()).isNull();
    }
    
    @Test
    void testAuditingWithDifferentTimeZones() {
        // Тест с разными часовыми поясами
        ZoneId utcZone = ZoneOffset.UTC;
        ZoneId localZone = ZoneId.systemDefault();
        
        User user = new User();
        user.setName("Timezone Test User");
        user.setEmail("timezone@example.com");
        
        User savedUser = userRepository.save(user);
        
        LocalDateTime createdDate = savedUser.getCreatedDate();
        
        // Проверка, что дата находится в разумных пределах
        LocalDateTime now = LocalDateTime.now();
        assertThat(createdDate).isBetween(now.minusMinutes(1), now.plusMinutes(1));
    }
    
    @Test
    void testAuditingWithBatchOperations() {
        // Тест аудита при пакетных операциях
        List<User> users = new ArrayList<>();
        
        for (int i = 0; i < 10; i++) {
            User user = new User();
            user.setName("Batch User " + i);
            user.setEmail("batch" + i + "@example.com");
            users.add(user);
        }
        
        List<User> savedUsers = userRepository.saveAll(users);
        
        // Проверка, что все пользователи имеют аудиторскую информацию
        for (User user : savedUsers) {
            assertThat(user.getCreatedDate()).isNotNull();
            assertThat(user.getLastModifiedDate()).isNotNull();
            assertThat(user.getCreatedBy()).isNotNull();
            assertThat(user.getLastModifiedBy()).isNotNull();
        }
    }
}

// Расширенная сущность пользователя
@Entity
@Table(name = "extended_users")
public class ExtendedUser extends ExtendedAuditingEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "email", unique = true, nullable = false)
    private String email;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

## Аннотации @CreatedBy и @LastModifiedBy

### Базовая структура

```java
// Интерфейс для получения текущего аудитора
public interface AuditorAware<T> {
    Optional<T> getCurrentAuditor();
}

// Реализация для строкового аудитора
@Component
public class StringAuditorAware implements AuditorAware<String> {
    
    @Override
    public Optional<String> getCurrentAuditor() {
        // Получение из Spring Security
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication != null && authentication.isAuthenticated() && 
            !authentication.getPrincipal().equals("anonymousUser")) {
            return Optional.of(authentication.getName());
        }
        
        // Получение из HTTP запроса
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder
            .currentRequestAttributes()).getRequest();
        
        String userId = request.getHeader("X-User-Id");
        if (userId != null) {
            return Optional.of(userId);
        }
        
        // Получение из ThreadLocal
        String threadLocalUser = AuditorContext.getCurrentAuditor();
        if (threadLocalUser != null) {
            return Optional.of(threadLocalUser);
        }
        
        return Optional.of("system");
    }
}

// Реализация для числового аудитора
@Component
public class LongAuditorAware implements AuditorAware<Long> {
    
    @Override
    public Optional<Long> getCurrentAuditor() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication != null && authentication.isAuthenticated() && 
            !authentication.getPrincipal().equals("anonymousUser")) {
            
            Object principal = authentication.getPrincipal();
            
            if (principal instanceof UserDetails) {
                // Предполагаем, что у UserDetails есть метод getUserId()
                try {
                    Method getUserIdMethod = principal.getClass().getMethod("getUserId");
                    Object userId = getUserIdMethod.invoke(principal);
                    if (userId instanceof Long) {
                        return Optional.of((Long) userId);
                    }
                } catch (Exception e) {
                    // Игнорируем ошибки
                }
            }
        }
        
        return Optional.of(1L); // Системный пользователь
    }
}
```

### Создание AuditorAware Bean

```java
@Configuration
public class AuditorAwareConfig {
    
    @Bean
    @Primary
    public AuditorAware<String> stringAuditorAware() {
        return new StringAuditorAware();
    }
    
    @Bean
    public AuditorAware<Long> longAuditorAware() {
        return new LongAuditorAware();
    }
    
    @Bean
    public AuditorAware<User> userAuditorAware() {
        return new UserAuditorAware();
    }
}

// Реализация для пользовательского объекта
@Component
public class UserAuditorAware implements AuditorAware<User> {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public Optional<User> getCurrentAuditor() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication != null && authentication.isAuthenticated() && 
            !authentication.getPrincipal().equals("anonymousUser")) {
            
            String username = authentication.getName();
            return userRepository.findByEmail(username);
        }
        
        return Optional.empty();
    }
}

// Сущность с пользовательским аудитором
@Entity
@Table(name = "audited_entities")
@EntityListeners(AuditingEntityListener.class)
public class AuditedEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name")
    private String name;
    
    @CreatedBy
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "created_by_user_id")
    private User createdByUser;
    
    @LastModifiedBy
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "last_modified_by_user_id")
    private User lastModifiedByUser;
    
    @CreatedDate
    @Column(name = "created_date")
    private LocalDateTime createdDate;
    
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private LocalDateTime lastModifiedDate;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public User getCreatedByUser() { return createdByUser; }
    public void setCreatedByUser(User createdByUser) { this.createdByUser = createdByUser; }
    
    public User getLastModifiedByUser() { return lastModifiedByUser; }
    public void setLastModifiedByUser(User lastModifiedByUser) { this.lastModifiedByUser = lastModifiedByUser; }
    
    public LocalDateTime getCreatedDate() { return createdDate; }
    public void setCreatedDate(LocalDateTime createdDate) { this.createdDate = createdDate; }
    
    public LocalDateTime getLastModifiedDate() { return lastModifiedDate; }
    public void setLastModifiedDate(LocalDateTime lastModifiedDate) { this.lastModifiedDate = lastModifiedDate; }
}
```

### Сложные сценарии с AuditorAware

```java
// Реализация с кэшированием
@Component
public class CachedAuditorAware implements AuditorAware<String> {
    
    private final Map<String, String> userCache = new ConcurrentHashMap<>();
    
    @Override
    public Optional<String> getCurrentAuditor() {
        String cacheKey = getCacheKey();
        
        if (userCache.containsKey(cacheKey)) {
            return Optional.of(userCache.get(cacheKey));
        }
        
        String auditor = resolveCurrentAuditor();
        userCache.put(cacheKey, auditor);
        
        return Optional.of(auditor);
    }
    
    private String getCacheKey() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return authentication != null ? authentication.getName() : "anonymous";
    }
    
    private String resolveCurrentAuditor() {
        // Сложная логика разрешения аудитора
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication != null && authentication.isAuthenticated()) {
            Object principal = authentication.getPrincipal();
            
            if (principal instanceof UserDetails) {
                return ((UserDetails) principal).getUsername();
            } else if (principal instanceof String) {
                return (String) principal;
            }
        }
        
        // Получение из HTTP заголовков
        try {
            HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder
                .currentRequestAttributes()).getRequest();
            
            String userId = request.getHeader("X-User-Id");
            if (userId != null) {
                return userId;
            }
            
            String sessionUser = (String) request.getSession().getAttribute("currentUser");
            if (sessionUser != null) {
                return sessionUser;
            }
        } catch (Exception e) {
            // Игнорируем ошибки
        }
        
        return "system";
    }
    
    // Метод для очистки кэша
    public void clearCache() {
        userCache.clear();
    }
}

// Реализация с поддержкой мультитенантности
@Component
public class MultiTenantAuditorAware implements AuditorAware<String> {
    
    @Override
    public Optional<String> getCurrentAuditor() {
        String tenantId = getCurrentTenantId();
        String userId = getCurrentUserId();
        
        if (tenantId != null && userId != null) {
            return Optional.of(tenantId + ":" + userId);
        }
        
        return Optional.of("system");
    }
    
    private String getCurrentTenantId() {
        // Получение tenant ID из различных источников
        HttpServletRequest request = getCurrentRequest();
        
        if (request != null) {
            String tenantId = request.getHeader("X-Tenant-Id");
            if (tenantId != null) {
                return tenantId;
            }
            
            tenantId = (String) request.getSession().getAttribute("tenantId");
            if (tenantId != null) {
                return tenantId;
            }
        }
        
        return "default";
    }
    
    private String getCurrentUserId() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication != null && authentication.isAuthenticated() && 
            !authentication.getPrincipal().equals("anonymousUser")) {
            return authentication.getName();
        }
        
        return "system";
    }
    
    private HttpServletRequest getCurrentRequest() {
        try {
            return ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        } catch (Exception e) {
            return null;
        }
    }
}
```

## Лучшие практики

### 1. Структура аудиторских сущностей

```java
@Service
public class AuditingBestPractices {
    
    @Autowired
    private UserRepository userRepository;
    
    // Хорошо: Использование базового класса для аудита
    public User createUserWithAuditing(String name, String email) {
        User user = new User();
        user.setName(name);
        user.setEmail(email);
        
        // Аудит будет применен автоматически
        return userRepository.save(user);
    }
    
    // Хорошо: Проверка аудиторской информации
    public void validateAuditingInfo(User user) {
        assertThat(user.getCreatedDate()).isNotNull();
        assertThat(user.getLastModifiedDate()).isNotNull();
        assertThat(user.getCreatedBy()).isNotNull();
        assertThat(user.getLastModifiedBy()).isNotNull();
    }
    
    // Плохо: Ручная установка аудиторских полей
    public User createUserBad(String name, String email) {
        User user = new User();
        user.setName(name);
        user.setEmail(email);
        
        // Не делайте так - аудит должен быть автоматическим
        user.setCreatedDate(LocalDateTime.now());
        user.setCreatedBy("manual");
        
        return userRepository.save(user);
    }
}
```

### 2. Оптимизация производительности

```java
@Service
public class AuditingPerformanceService {
    
    @Autowired
    private UserRepository userRepository;
    
    // Мониторинг производительности аудита
    public <T> T monitorAuditingOperation(Supplier<T> operation, String operationName) {
        long startTime = System.currentTimeMillis();
        
        try {
            T result = operation.get();
            
            long endTime = System.currentTimeMillis();
            long duration = endTime - startTime;
            
            logAuditingPerformance(operationName, duration);
            
            return result;
        } catch (Exception e) {
            logAuditingError(operationName, e);
            throw e;
        }
    }
    
    private void logAuditingPerformance(String operationName, long duration) {
        System.out.printf("Auditing operation '%s' выполнена за %dms%n", operationName, duration);
    }
    
    private void logAuditingError(String operationName, Exception e) {
        System.err.printf("Ошибка в auditing operation '%s': %s%n", operationName, e.getMessage());
    }
    
    // Кэширование аудиторской информации
    @Cacheable(value = "auditorInfo", key = "#userId")
    public String getCachedAuditorInfo(String userId) {
        // Получение информации об аудиторе
        return "auditor:" + userId;
    }
}
```

### 3. Тестирование аудита

```java
@SpringBootTest
@Transactional
class AuditingIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testAuditingIntegration() {
        // Подготовка тестового пользователя
        User testUser = createTestUser();
        
        // Создание пользователя с аудитом
        User createdUser = userRepository.save(testUser);
        
        // Проверка аудиторской информации
        assertThat(createdUser.getCreatedDate()).isNotNull();
        assertThat(createdUser.getLastModifiedDate()).isNotNull();
        assertThat(createdUser.getCreatedBy()).isNotNull();
        assertThat(createdUser.getLastModifiedBy()).isNotNull();
        
        // Обновление пользователя
        createdUser.setName("Updated Name");
        User updatedUser = userRepository.save(createdUser);
        
        // Проверка, что createdDate не изменился
        assertThat(updatedUser.getCreatedDate()).isEqualTo(createdUser.getCreatedDate());
        
        // Проверка, что lastModifiedDate обновился
        assertThat(updatedUser.getLastModifiedDate()).isAfter(createdUser.getLastModifiedDate());
    }
    
    private User createTestUser() {
        User user = new User();
        user.setName("Test User");
        user.setEmail("test@example.com");
        return user;
    }
}
```

## Заключение

JPA Auditing предоставляет мощные возможности для автоматического отслеживания изменений в сущностях. Правильное использование аудита позволяет:

- Автоматически отслеживать время создания и обновления записей
- Идентифицировать пользователей, создавших и изменивших записи
- Обеспечивать соответствие требованиям к аудиту и безопасности
- Упрощать отладку и мониторинг изменений

Ключевые моменты для успешного использования:

1. **Используйте базовые классы** для аудиторских сущностей
2. **Настройте @EnableJpaAuditing** правильно
3. **Создайте AuditorAware Bean** для получения текущего аудитора
4. **Тестируйте аудиторскую функциональность** тщательно
5. **Мониторьте производительность** аудиторских операций
6. **Используйте кэширование** для часто запрашиваемой аудиторской информации
