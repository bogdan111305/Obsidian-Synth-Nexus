# @Lock & @QueryHints в Spring Data JPA

## Обзор

`@Lock` и `@QueryHints` - это мощные аннотации Spring Data JPA, которые позволяют контролировать блокировки и оптимизировать производительность запросов. `@Lock` используется для управления блокировками на уровне JPA, а `@QueryHints` предоставляет возможность передачи подсказок провайдеру JPA.

## Аннотация @Lock

### Базовая структура

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Lock {
    LockModeType value() default LockModeType.NONE;
}
```

### Типы блокировок

```java
public enum LockModeType {
    NONE,           // Без блокировки
    OPTIMISTIC,     // Оптимистическая блокировка
    OPTIMISTIC_FORCE_INCREMENT, // Оптимистическая блокировка с принудительным инкрементом
    PESSIMISTIC_READ,   // Пессимистическая блокировка для чтения
    PESSIMISTIC_WRITE,  // Пессимистическая блокировка для записи
    PESSIMISTIC_FORCE_INCREMENT // Пессимистическая блокировка с принудительным инкрементом
}
```

### Базовое использование @Lock

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @Version
    private Long version;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    // Геттеры и сеттеры
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Оптимистическая блокировка
    @Lock(LockModeType.OPTIMISTIC)
    Optional<User> findByIdWithOptimisticLock(Long id);
    
    // Пессимистическая блокировка для чтения
    @Lock(LockModeType.PESSIMISTIC_READ)
    Optional<User> findByIdWithPessimisticRead(Long id);
    
    // Пессимистическая блокировка для записи
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<User> findByIdWithPessimisticWrite(Long id);
    
    // Комбинирование с @Query
    @Query("SELECT u FROM User u WHERE u.email = :email")
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<User> findByEmailWithLock(@Param("email") String email);
    
    // Блокировка с пагинацией
    @Lock(LockModeType.PESSIMISTIC_READ)
    Page<User> findByActiveTrue(Pageable pageable);
}
```

## Демонстрация пессимистических блокировок

### Сценарий с конкурентным доступом

```java
@Service
@Transactional
public class PessimisticLockDemoService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EntityManager entityManager;
    
    // Метод без блокировки (может привести к проблемам)
    public void updateUserWithoutLock(Long userId, String newName) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
        
        // Имитация длительной операции
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        user.setName(newName);
        userRepository.save(user);
    }
    
    // Метод с пессимистической блокировкой
    public void updateUserWithPessimisticLock(Long userId, String newName) {
        User user = userRepository.findByIdWithPessimisticWrite(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
        
        // Имитация длительной операции
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        user.setName(newName);
        userRepository.save(user);
    }
    
    // Метод с оптимистической блокировкой
    public void updateUserWithOptimisticLock(Long userId, String newName) {
        User user = userRepository.findByIdWithOptimisticLock(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
        
        // Имитация длительной операции
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        user.setName(newName);
        userRepository.save(user);
    }
    
    // Метод с ручной блокировкой через EntityManager
    public void updateUserWithManualLock(Long userId, String newName) {
        User user = entityManager.find(User.class, userId, LockModeType.PESSIMISTIC_WRITE);
        if (user == null) {
            throw new EntityNotFoundException("User not found");
        }
        
        // Имитация длительной операции
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        user.setName(newName);
        entityManager.merge(user);
    }
}
```

### Тестирование конкурентного доступа

```java
@SpringBootTest
@Transactional
class PessimisticLockTest {
    
    @Autowired
    private PessimisticLockDemoService lockService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testConcurrentAccessWithoutLock() throws InterruptedException {
        // Создание тестового пользователя
        User user = new User();
        user.setName("Original Name");
        user.setEmail("test@example.com");
        User savedUser = userRepository.save(user);
        
        // Запуск двух потоков без блокировки
        CountDownLatch latch = new CountDownLatch(2);
        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger failureCount = new AtomicInteger(0);
        
        Thread thread1 = new Thread(() -> {
            try {
                lockService.updateUserWithoutLock(savedUser.getId(), "Name from Thread 1");
                successCount.incrementAndGet();
            } catch (Exception e) {
                failureCount.incrementAndGet();
                System.err.println("Thread 1 failed: " + e.getMessage());
            } finally {
                latch.countDown();
            }
        });
        
        Thread thread2 = new Thread(() -> {
            try {
                lockService.updateUserWithoutLock(savedUser.getId(), "Name from Thread 2");
                successCount.incrementAndGet();
            } catch (Exception e) {
                failureCount.incrementAndGet();
                System.err.println("Thread 2 failed: " + e.getMessage());
            } finally {
                latch.countDown();
            }
        });
        
        thread1.start();
        thread2.start();
        latch.await();
        
        // Проверка результатов
        User updatedUser = userRepository.findById(savedUser.getId()).orElse(null);
        assertThat(updatedUser).isNotNull();
        System.out.println("Final name: " + updatedUser.getName());
        System.out.println("Success count: " + successCount.get());
        System.out.println("Failure count: " + failureCount.get());
    }
    
    @Test
    void testConcurrentAccessWithPessimisticLock() throws InterruptedException {
        // Создание тестового пользователя
        User user = new User();
        user.setName("Original Name");
        user.setEmail("test@example.com");
        User savedUser = userRepository.save(user);
        
        // Запуск двух потоков с пессимистической блокировкой
        CountDownLatch latch = new CountDownLatch(2);
        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger failureCount = new AtomicInteger(0);
        
        Thread thread1 = new Thread(() -> {
            try {
                lockService.updateUserWithPessimisticLock(savedUser.getId(), "Name from Thread 1");
                successCount.incrementAndGet();
            } catch (Exception e) {
                failureCount.incrementAndGet();
                System.err.println("Thread 1 failed: " + e.getMessage());
            } finally {
                latch.countDown();
            }
        });
        
        Thread thread2 = new Thread(() -> {
            try {
                lockService.updateUserWithPessimisticLock(savedUser.getId(), "Name from Thread 2");
                successCount.incrementAndGet();
            } catch (Exception e) {
                failureCount.incrementAndGet();
                System.err.println("Thread 2 failed: " + e.getMessage());
            } finally {
                latch.countDown();
            }
        });
        
        thread1.start();
        thread2.start();
        latch.await();
        
        // Проверка результатов
        User updatedUser = userRepository.findById(savedUser.getId()).orElse(null);
        assertThat(updatedUser).isNotNull();
        System.out.println("Final name: " + updatedUser.getName());
        System.out.println("Success count: " + successCount.get());
        System.out.println("Failure count: " + failureCount.get());
    }
}
```

### Сложные сценарии блокировок

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Блокировка с условием
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<User> findActiveUserByEmailWithLock(@Param("email") String email);
    
    // Блокировка с JOIN
    @Query("SELECT u FROM User u JOIN u.department d WHERE d.name = :deptName")
    @Lock(LockModeType.PESSIMISTIC_READ)
    List<User> findUsersByDepartmentWithLock(@Param("deptName") String deptName);
    
    // Блокировка с пагинацией
    @Lock(LockModeType.PESSIMISTIC_READ)
    Page<User> findByDepartmentName(String departmentName, Pageable pageable);
    
    // Оптимистическая блокировка с версией
    @Lock(LockModeType.OPTIMISTIC)
    @Query("SELECT u FROM User u WHERE u.id = :id")
    Optional<User> findByIdWithOptimisticLock(@Param("id") Long id);
}

@Service
@Transactional
public class AdvancedLockService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EntityManager entityManager;
    
    // Метод с таймаутом для блокировки
    public User findUserWithTimeout(Long userId, int timeoutSeconds) {
        Map<String, Object> properties = new HashMap<>();
        properties.put("javax.persistence.lock.timeout", timeoutSeconds * 1000);
        
        return entityManager.find(User.class, userId, LockModeType.PESSIMISTIC_WRITE, properties);
    }
    
    // Метод с условной блокировкой
    public User findUserWithConditionalLock(Long userId, boolean useLock) {
        if (useLock) {
            return userRepository.findByIdWithPessimisticWrite(userId)
                .orElseThrow(() -> new EntityNotFoundException("User not found"));
        } else {
            return userRepository.findById(userId)
                .orElseThrow(() -> new EntityNotFoundException("User not found"));
        }
    }
    
    // Метод с блокировкой нескольких сущностей
    public void updateUserAndDepartment(Long userId, String userName, String deptName) {
        User user = userRepository.findByIdWithPessimisticWrite(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
        
        Department dept = entityManager.find(Department.class, user.getDepartment().getId(), 
            LockModeType.PESSIMISTIC_WRITE);
        
        user.setName(userName);
        dept.setName(deptName);
        
        userRepository.save(user);
        entityManager.merge(dept);
    }
}
```

## Аннотация @QueryHints

### Базовая структура

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface QueryHints {
    QueryHint[] value();
}

@Target({})
@Retention(RetentionPolicy.RUNTIME)
public @interface QueryHint {
    String name();
    String value();
}
```

### Основные типы QueryHints

```java
// Hibernate QueryHints
public static final String HINT_FETCH_SIZE = "org.hibernate.fetchSize";
public static final String HINT_CACHEABLE = "org.hibernate.cacheable";
public static final String HINT_CACHE_REGION = "org.hibernate.cacheRegion";
public static final String HINT_CACHE_MODE = "org.hibernate.cacheMode";
public static final String HINT_COMMENT = "org.hibernate.comment";
public static final String HINT_TIMEOUT_SEC = "org.hibernate.timeout";
public static final String HINT_FLUSH_MODE = "org.hibernate.flushMode";
public static final String HINT_READ_ONLY = "org.hibernate.readOnly";

// JPA QueryHints
public static final String HINT_JAVA_TIMEOUT = "javax.persistence.query.timeout";
public static final String HINT_LOCK_TIMEOUT = "javax.persistence.lock.timeout";
```

### Базовое использование @QueryHints

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простой QueryHint
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "50")
    })
    List<User> findByActiveTrue();
    
    // Множественные QueryHints
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "100"),
        @QueryHint(name = HINT_CACHEABLE, value = "true"),
        @QueryHint(name = HINT_COMMENT, value = "Find active users")
    })
    List<User> findByDepartmentName(String departmentName);
    
    // QueryHints с кэшированием
    @QueryHints(value = {
        @QueryHint(name = HINT_CACHEABLE, value = "true"),
        @QueryHint(name = HINT_CACHE_REGION, value = "userCache")
    })
    Optional<User> findById(Long id);
    
    // QueryHints с таймаутом
    @QueryHints(value = {
        @QueryHint(name = HINT_TIMEOUT_SEC, value = "30")
    })
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmailWithTimeout(@Param("email") String email);
    
    // QueryHints с пагинацией
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "25"),
        @QueryHint(name = HINT_READ_ONLY, value = "true")
    })
    Page<User> findByActiveTrue(Pageable pageable);
}
```

### Сложные сценарии с QueryHints

```java
@Repository
public interface AdvancedUserRepository extends JpaRepository<User, Long> {
    
    // Оптимизация для больших результатов
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "1000"),
        @QueryHint(name = HINT_CACHEABLE, value = "true"),
        @QueryHint(name = HINT_CACHE_REGION, value = "largeResultCache")
    })
    @Query("SELECT u FROM User u WHERE u.department.name = :deptName")
    List<User> findUsersByDepartmentOptimized(@Param("deptName") String deptName);
    
    // Оптимизация для отчетов
    @QueryHints(value = {
        @QueryHint(name = HINT_READ_ONLY, value = "true"),
        @QueryHint(name = HINT_FLUSH_MODE, value = "MANUAL"),
        @QueryHint(name = HINT_COMMENT, value = "Report query - read only")
    })
    @Query("SELECT u FROM User u JOIN u.department d " +
           "WHERE d.name = :deptName AND u.active = true")
    List<User> findActiveUsersForReport(@Param("deptName") String deptName);
    
    // Оптимизация для экспорта
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "500"),
        @QueryHint(name = HINT_READ_ONLY, value = "true"),
        @QueryHint(name = HINT_TIMEOUT_SEC, value = "300")
    })
    @Query("SELECT u FROM User u WHERE u.active = true")
    Stream<User> findActiveUsersForExport();
    
    // Оптимизация с блокировкой
    @QueryHints(value = {
        @QueryHint(name = HINT_LOCK_TIMEOUT, value = "5000"),
        @QueryHint(name = HINT_COMMENT, value = "Locked query")
    })
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT u FROM User u WHERE u.id = :id")
    Optional<User> findByIdWithLockAndHints(@Param("id") Long id);
}
```

### Динамические QueryHints

```java
@Service
public class DynamicQueryHintsService {
    
    @Autowired
    private EntityManager entityManager;
    
    public List<User> findUsersWithDynamicHints(Map<String, String> hints) {
        String jpql = "SELECT u FROM User u WHERE u.active = true";
        Query query = entityManager.createQuery(jpql);
        
        // Применение динамических подсказок
        hints.forEach(query::setHint);
        
        return query.getResultList();
    }
    
    public List<User> findUsersWithConditionalHints(boolean useCache, int fetchSize) {
        Map<String, String> hints = new HashMap<>();
        
        if (useCache) {
            hints.put(HINT_CACHEABLE, "true");
            hints.put(HINT_CACHE_REGION, "userCache");
        }
        
        hints.put(HINT_FETCH_SIZE, String.valueOf(fetchSize));
        
        return findUsersWithDynamicHints(hints);
    }
    
    public List<User> findUsersForExport() {
        Map<String, String> hints = new HashMap<>();
        hints.put(HINT_FETCH_SIZE, "1000");
        hints.put(HINT_READ_ONLY, "true");
        hints.put(HINT_TIMEOUT_SEC, "300");
        hints.put(HINT_COMMENT, "Export query");
        
        return findUsersWithDynamicHints(hints);
    }
}
```

### Оптимизация производительности с QueryHints

```java
@Repository
public interface OptimizedUserRepository extends JpaRepository<User, Long> {
    
    // Оптимизация для списков
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "50"),
        @QueryHint(name = HINT_CACHEABLE, value = "true")
    })
    @EntityGraph(attributePaths = {"department"})
    Page<User> findByActiveTrueOptimized(Pageable pageable);
    
    // Оптимизация для детального просмотра
    @QueryHints(value = {
        @QueryHint(name = HINT_CACHEABLE, value = "true"),
        @QueryHint(name = HINT_CACHE_REGION, value = "userDetailsCache")
    })
    @EntityGraph(attributePaths = {
        "department",
        "roles",
        "orders"
    })
    Optional<User> findByIdWithDetails(Long id);
    
    // Оптимизация для поиска
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "25"),
        @QueryHint(name = HINT_COMMENT, value = "Search query")
    })
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name% OR u.email LIKE %:email%")
    List<User> searchUsers(@Param("name") String name, @Param("email") String email);
}
```

## Комбинирование @Lock и @QueryHints

```java
@Repository
public interface CombinedUserRepository extends JpaRepository<User, Long> {
    
    // Комбинирование блокировки и подсказок
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(value = {
        @QueryHint(name = HINT_LOCK_TIMEOUT, value = "10000"),
        @QueryHint(name = HINT_COMMENT, value = "Pessimistic lock with timeout")
    })
    Optional<User> findByIdWithLockAndTimeout(Long id);
    
    // Оптимистическая блокировка с подсказками
    @Lock(LockModeType.OPTIMISTIC)
    @QueryHints(value = {
        @QueryHint(name = HINT_CACHEABLE, value = "true"),
        @QueryHint(name = HINT_COMMENT, value = "Optimistic lock with cache")
    })
    Optional<User> findByIdWithOptimisticLockAndCache(Long id);
    
    // Блокировка с пагинацией и подсказками
    @Lock(LockModeType.PESSIMISTIC_READ)
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "25"),
        @QueryHint(name = HINT_READ_ONLY, value = "true")
    })
    Page<User> findByDepartmentNameWithLock(String departmentName, Pageable pageable);
}
```

## Лучшие практики

### 1. Выбор правильной блокировки

```java
@Service
public class LockBestPractices {
    
    @Autowired
    private UserRepository userRepository;
    
    // Хорошо: Оптимистическая блокировка для коротких операций
    public void updateUserOptimistic(Long userId, String newName) {
        User user = userRepository.findByIdWithOptimisticLock(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
        
        user.setName(newName);
        userRepository.save(user);
    }
    
    // Хорошо: Пессимистическая блокировка для длительных операций
    public void updateUserPessimistic(Long userId, String newName) {
        User user = userRepository.findByIdWithPessimisticWrite(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
        
        // Длительная операция
        performComplexCalculation(user);
        
        user.setName(newName);
        userRepository.save(user);
    }
    
    // Плохо: Пессимистическая блокировка для простых операций
    public void updateUserPessimisticBad(Long userId, String newName) {
        User user = userRepository.findByIdWithPessimisticWrite(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
        
        // Простая операция - избыточная блокировка
        user.setName(newName);
        userRepository.save(user);
    }
    
    private void performComplexCalculation(User user) {
        // Имитация сложной операции
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 2. Оптимизация с QueryHints

```java
@Service
public class QueryHintsBestPractices {
    
    @Autowired
    private UserRepository userRepository;
    
    // Хорошо: Оптимизация для списков
    public Page<User> getUsersForList(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByActiveTrueOptimized(pageable);
    }
    
    // Хорошо: Оптимизация для детального просмотра
    public User getUserForDetails(Long userId) {
        return userRepository.findByIdWithDetails(userId).orElse(null);
    }
    
    // Хорошо: Оптимизация для экспорта
    public List<User> getUsersForExport() {
        return userRepository.findActiveUsersForExport().collect(Collectors.toList());
    }
    
    // Плохо: Избыточные подсказки
    public List<User> getUsersBad() {
        // Избыточные подсказки для простого запроса
        return userRepository.findByActiveTrue(); // Уже оптимизировано
    }
}
```

### 3. Обработка исключений

```java
@Service
public class LockExceptionHandler {
    
    @Autowired
    private UserRepository userRepository;
    
    public void updateUserWithExceptionHandling(Long userId, String newName) {
        try {
            User user = userRepository.findByIdWithPessimisticWrite(userId)
                .orElseThrow(() -> new EntityNotFoundException("User not found"));
            
            user.setName(newName);
            userRepository.save(user);
            
        } catch (PessimisticLockException e) {
            // Обработка ошибки блокировки
            System.err.println("Lock timeout: " + e.getMessage());
            throw new RuntimeException("User is currently being updated by another process", e);
            
        } catch (OptimisticLockException e) {
            // Обработка ошибки оптимистической блокировки
            System.err.println("Optimistic lock failed: " + e.getMessage());
            throw new RuntimeException("User was modified by another process", e);
            
        } catch (Exception e) {
            // Общая обработка ошибок
            System.err.println("Unexpected error: " + e.getMessage());
            throw new RuntimeException("Failed to update user", e);
        }
    }
}
```

### 4. Мониторинг производительности

```java
@Service
public class LockPerformanceService {
    
    @Autowired
    private UserRepository userRepository;
    
    public <T> T monitorLockQuery(Supplier<T> querySupplier, String queryName) {
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
        System.out.printf("Lock query '%s' выполнен за %dms%n", queryName, duration);
    }
    
    private void logQueryError(String queryName, Exception e) {
        System.err.printf("Ошибка в lock query '%s': %s%n", queryName, e.getMessage());
    }
}
```

## Заключение

`@Lock` и `@QueryHints` предоставляют мощные возможности для управления блокировками и оптимизации производительности в Spring Data JPA. Правильное использование этих аннотаций позволяет:

- Контролировать конкурентный доступ к данным
- Оптимизировать производительность запросов
- Избежать конфликтов при одновременном доступе
- Создавать масштабируемые приложения

Ключевые моменты для успешного использования:

1. **Выбирайте правильный тип блокировки** в зависимости от сценария
2. **Используйте QueryHints** для оптимизации производительности
3. **Обрабатывайте исключения** блокировок
4. **Мониторьте производительность** и оптимизируйте при необходимости
5. **Избегайте избыточных блокировок** для простых операций
