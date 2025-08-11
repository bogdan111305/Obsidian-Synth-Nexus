# Аннотация @Modifying

## Обзор

Аннотация `@Modifying` в Spring Data JPA используется для обозначения методов, которые изменяют данные в базе данных (INSERT, UPDATE, DELETE). Она должна использоваться вместе с аннотацией `@Query` для методов, которые не являются стандартными CRUD операциями.

## Основные возможности

### Типы модифицирующих операций
- **UPDATE запросы** - изменение существующих записей
- **DELETE запросы** - удаление записей
- **Массовые операции** - обработка больших наборов данных
- **Кастомные модификации** - специфичные операции изменения данных

### Структура аннотации

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("JPQL запрос для модификации")
```

## UPDATE запросы

### Базовые UPDATE операции

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Modifying
    @Query("UPDATE User u SET u.active = ?2 WHERE u.id = ?1")
    int updateUserStatus(Long id, boolean active);
    
    @Modifying
    @Query("UPDATE User u SET u.email = ?2 WHERE u.id = ?1")
    int updateUserEmail(Long id, String email);
    
    @Modifying
    @Query("UPDATE User u SET u.lastLoginDate = ?2 WHERE u.id = ?1")
    int updateLastLoginDate(Long id, LocalDateTime lastLoginDate);
}
```

### Массовые UPDATE операции

```java
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.role = ?2")
int updateActiveStatusByRole(boolean active, String role);

@Modifying
@Query("UPDATE User u SET u.lastLoginDate = ?1 WHERE u.lastLoginDate < ?2")
int updateLastLoginForInactiveUsers(LocalDateTime newDate, LocalDateTime threshold);

@Modifying
@Query("UPDATE User u SET u.role = ?1 WHERE u.role = ?2")
int updateRoleForUsers(String newRole, String oldRole);
```

### UPDATE с условиями

```java
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.email LIKE %?2%")
int updateActiveStatusByEmailPattern(boolean active, String emailPattern);

@Modifying
@Query("UPDATE User u SET u.role = ?1 WHERE u.createdDate BETWEEN ?2 AND ?3")
int updateRoleForUsersInDateRange(String role, LocalDateTime start, LocalDateTime end);
```

### UPDATE с именованными параметрами

```java
@Modifying
@Query("UPDATE User u SET u.salary = :salary WHERE u.department.id = :deptId AND u.experience > :minExp")
int updateSalaryByDepartmentAndExperience(@Param("salary") BigDecimal salary, 
                                         @Param("deptId") Long deptId, 
                                         @Param("minExp") Integer minExp);

@Modifying
@Query("UPDATE User u SET u.active = :active WHERE u.id IN :userIds")
int updateActiveStatusForUsers(@Param("active") boolean active, 
                              @Param("userIds") List<Long> userIds);
```

## DELETE запросы

### Базовые DELETE операции

```java
@Modifying
@Query("DELETE FROM User u WHERE u.id = ?1")
int deleteUserById(Long id);

@Modifying
@Query("DELETE FROM User u WHERE u.email = ?1")
int deleteUserByEmail(String email);
```

### Массовые DELETE операции

```java
@Modifying
@Query("DELETE FROM User u WHERE u.role = ?1")
int deleteUsersByRole(String role);

@Modifying
@Query("DELETE FROM User u WHERE u.active = false")
int deleteInactiveUsers();

@Modifying
@Query("DELETE FROM User u WHERE u.lastLoginDate < ?1")
int deleteUsersNotLoggedInSince(LocalDateTime date);
```

### DELETE с JOIN и подзапросами

```java
@Modifying
@Query("DELETE FROM User u WHERE u.id IN (SELECT u.id FROM User u JOIN u.orders o WHERE o.status = ?1)")
int deleteUsersWithOrdersByStatus(String orderStatus);

@Modifying
@Query("DELETE FROM User u WHERE u.department.id IN (SELECT d.id FROM Department d WHERE d.budget < ?1)")
int deleteUsersFromLowBudgetDepartments(BigDecimal minBudget);
```

### DELETE с именованными параметрами

```java
@Modifying
@Query("DELETE FROM User u WHERE u.role = :role AND u.createdDate < :date")
int deleteUsersCreatedBeforeDate(@Param("role") String role, 
                                @Param("date") LocalDateTime date);

@Modifying
@Query("DELETE FROM User u WHERE u.id IN :userIds")
int deleteUsersByIds(@Param("userIds") List<Long> userIds);
```

## Свойства @Modifying

### clearAutomatically

```java
@Modifying(clearAutomatically = true)
@Query("UPDATE User u SET u.active = ?2 WHERE u.id = ?1")
int updateUserStatus(Long id, boolean active);
```

**clearAutomatically = true** - автоматически очищает кэш первого уровня (Persistence Context) после выполнения операции. Это важно, поскольку модифицирующие запросы выполняются на уровне базы данных и не обновляют сущности в кэше.

### flushAutomatically

```java
@Modifying(flushAutomatically = true)
@Query("UPDATE User u SET u.email = ?2 WHERE u.id = ?1")
int updateUserEmail(Long id, String email);
```

**flushAutomatically = true** - автоматически сбрасывает все изменения из Persistence Context в базу данных перед выполнением операции. Это гарантирует, что модифицирующий запрос увидит все предыдущие изменения.

### Комбинирование свойств

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE User u SET u.lastModified = :date WHERE u.department.id = :deptId")
int updateLastModifiedByDepartment(@Param("date") LocalDateTime date, 
                                  @Param("deptId") Long deptId);
```

## Возвращаемые значения

### Количество затронутых строк

```java
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.role = ?2")
int updateActiveStatusByRole(boolean active, String role);

@Modifying
@Query("DELETE FROM User u WHERE u.role = ?1")
int deleteUsersByRole(String role);
```

### void методы

```java
@Modifying
@Query("UPDATE User u SET u.lastLoginDate = ?2 WHERE u.id = ?1")
void updateLastLoginDate(Long id, LocalDateTime lastLoginDate);

@Modifying
@Query("DELETE FROM User u WHERE u.active = false")
void cleanupInactiveUsers();
```

## Транзакции

### Настройка транзакций

```java
@Modifying
@Transactional
@Query("UPDATE User u SET u.active = ?2 WHERE u.id = ?1")
int updateUserStatus(Long id, boolean active);

@Modifying
@Transactional(rollbackFor = Exception.class)
@Query("DELETE FROM User u WHERE u.role = ?1")
int deleteUsersByRole(String role);
```

### Уровни изоляции

```java
@Modifying
@Transactional(isolation = Isolation.READ_COMMITTED)
@Query("UPDATE User u SET u.role = ?1 WHERE u.id = ?2")
int updateUserRole(String role, Long id);

@Modifying
@Transactional(isolation = Isolation.SERIALIZABLE)
@Query("UPDATE User u SET u.balance = u.balance - ?1 WHERE u.id = ?2")
int deductBalance(BigDecimal amount, Long userId);
```

### Timeout и readonly настройки

```java
@Modifying
@Transactional(timeout = 30, readOnly = false)
@Query("UPDATE User u SET u.archived = true WHERE u.lastLoginDate < ?1")
int archiveOldUsers(LocalDateTime cutoffDate);
```

## Обработка исключений

### Проверка результата

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public void updateUserStatus(Long id, boolean active) {
        int updatedRows = userRepository.updateUserStatus(id, active);
        
        if (updatedRows == 0) {
            throw new UserNotFoundException("User with id " + id + " not found");
        }
    }
    
    public void updateUsersInBatch(List<Long> userIds, boolean active) {
        int expectedUpdates = userIds.size();
        int actualUpdates = userRepository.updateActiveStatusForUsers(active, userIds);
        
        if (actualUpdates != expectedUpdates) {
            log.warn("Expected to update {} users, but actually updated {}", 
                    expectedUpdates, actualUpdates);
        }
    }
}
```

### Логирование операций

```java
@Modifying
@Query("UPDATE User u SET u.active = ?2 WHERE u.id = ?1")
int updateUserStatus(Long id, boolean active);

@Service
public class UserService {
    
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    
    public void updateUserStatus(Long id, boolean active) {
        logger.info("Updating user {} status to {}", id, active);
        
        try {
            int updatedRows = userRepository.updateUserStatus(id, active);
            logger.info("Successfully updated {} users", updatedRows);
        } catch (Exception e) {
            logger.error("Failed to update user {} status", id, e);
            throw e;
        }
    }
}
```

## Производительность

### Массовые операции

```java
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.role = ?2")
int updateActiveStatusByRole(boolean active, String role);

// Использование в сервисе
@Service
public class UserService {
    
    @Transactional
    public void deactivateUsersByRole(String role) {
        int updatedCount = userRepository.updateActiveStatusByRole(false, role);
        log.info("Deactivated {} users with role {}", updatedCount, role);
    }
}
```

### Батчевые операции

```java
@Modifying
@Query("UPDATE User u SET u.lastLoginDate = ?1 WHERE u.id = ?2")
int updateLastLoginDate(LocalDateTime date, Long id);

@Service
public class UserService {
    
    @Transactional
    public void updateLastLoginForUsers(List<Long> userIds) {
        LocalDateTime now = LocalDateTime.now();
        
        for (Long userId : userIds) {
            userRepository.updateLastLoginDate(now, userId);
        }
    }
    
    // Более эффективный подход
    @Transactional
    public void updateLastLoginForUsersBatch(List<Long> userIds) {
        LocalDateTime now = LocalDateTime.now();
        userRepository.updateLastLoginForUsers(now, userIds);
    }
}
```

### Оптимизация с использованием индексов

```java
// Хорошо - используется индекс по email
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.email = ?2")
int updateUserStatusByEmail(boolean active, String email);

// Лучше - используется первичный ключ
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.id IN ?2")
int updateUserStatusByIds(boolean active, List<Long> ids);
```

## Native SQL для модификации

### Native UPDATE запросы

```java
@Modifying
@Query(value = "UPDATE users SET salary = salary * 1.1 WHERE department_id = ?1", 
       nativeQuery = true)
int increaseSalaryByDepartment(Long departmentId);

@Modifying
@Query(value = "UPDATE users SET last_modified = CURRENT_TIMESTAMP WHERE active = true", 
       nativeQuery = true)
int updateLastModifiedForActiveUsers();
```

### Native DELETE запросы

```java
@Modifying
@Query(value = "DELETE FROM users WHERE created_date < ?1 AND role = 'TEMP'", 
       nativeQuery = true)
int deleteTempUsersOlderThan(LocalDateTime date);

@Modifying
@Query(value = "DELETE FROM user_sessions WHERE expires_at < NOW()", 
       nativeQuery = true)
int deleteExpiredSessions();
```

### Сложные native операции

```java
@Modifying
@Query(value = """
    UPDATE users u 
    SET u.rank = (
        SELECT COUNT(*) + 1 
        FROM users u2 
        WHERE u2.department_id = u.department_id 
        AND u2.salary > u.salary
    )
    WHERE u.department_id = ?1
    """, nativeQuery = true)
int updateUserRanksByDepartment(Long departmentId);
```

## Лучшие практики

### Безопасность

```java
// ✅ Хорошо - используйте параметризованные запросы
@Modifying
@Query("UPDATE User u SET u.active = :active WHERE u.id = :id")
int updateUserStatus(@Param("active") boolean active, @Param("id") Long id);

// ❌ Плохо - избегайте конкатенации строк
// @Query("UPDATE User u SET u.active = " + active + " WHERE u.id = " + id)
```

### Производительность

```java
// ✅ Используйте индексы для условий WHERE
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.email = ?2")
int updateUserStatusByEmail(boolean active, String email);

// ✅ Избегайте сканирования всей таблицы
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.id IN ?2")
int updateUserStatusByIds(boolean active, List<Long> ids);

// ✅ Используйте LIMIT для больших операций
@Modifying
@Query(value = "UPDATE users SET processed = true WHERE processed = false LIMIT 1000", 
       nativeQuery = true)
int processUsersInBatch();
```

### Транзакционность

```java
@Modifying
@Transactional
@Query("UPDATE User u SET u.active = ?2 WHERE u.id = ?1")
int updateUserStatus(Long id, boolean active);

// Используйте правильные уровни изоляции для критических операций
@Modifying
@Transactional(isolation = Isolation.SERIALIZABLE)
@Query("UPDATE User u SET u.balance = u.balance - ?1 WHERE u.id = ?2 AND u.balance >= ?1")
int deductBalanceIfSufficient(BigDecimal amount, Long userId);
```

### Мониторинг и логирование

```java
@Service
public class UserModificationService {
    
    private final UserRepository userRepository;
    private final MeterRegistry meterRegistry;
    
    @Transactional
    public void massUpdateUserStatus(String role, boolean active) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            int updatedCount = userRepository.updateActiveStatusByRole(active, role);
            
            meterRegistry.counter("user.status.updates", "role", role)
                        .increment(updatedCount);
            
            log.info("Updated status for {} users with role {}", updatedCount, role);
        } finally {
            sample.stop(Timer.builder("user.modification.duration")
                             .tag("operation", "status_update")
                             .register(meterRegistry));
        }
    }
}
```

## Отладка и логирование

### Включение логирования

```properties
# Логирование SQL запросов
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE

# Логирование Spring Data JPA
logging.level.org.springframework.data.jpa.repository.query=DEBUG
logging.level.org.springframework.transaction=DEBUG
```

### Мониторинг производительности

```java
@Component
public class ModifyingQueryMonitor {
    
    private final Map<String, AtomicLong> executionTimes = new ConcurrentHashMap<>();
    private final Map<String, AtomicLong> executionCounts = new ConcurrentHashMap<>();
    
    public int monitorExecution(String methodName, Supplier<Integer> execution) {
        long startTime = System.currentTimeMillis();
        
        try {
            int result = execution.get();
            long executionTime = System.currentTimeMillis() - startTime;
            
            executionTimes.computeIfAbsent(methodName, k -> new AtomicLong(0))
                         .addAndGet(executionTime);
            executionCounts.computeIfAbsent(methodName, k -> new AtomicLong(0))
                          .incrementAndGet();
            
            log.info("Method {} affected {} rows in {} ms", methodName, result, executionTime);
            
            return result;
        } catch (Exception e) {
            log.error("Error executing modifying method {}", methodName, e);
            throw e;
        }
    }
    
    @Scheduled(fixedRate = 60000) // Каждую минуту
    public void logStatistics() {
        executionTimes.forEach((method, totalTime) -> {
            long count = executionCounts.get(method).get();
            long avgTime = count > 0 ? totalTime.get() / count : 0;
            
            log.info("Method {} - Executions: {}, Total time: {} ms, Avg time: {} ms", 
                    method, count, totalTime.get(), avgTime);
        });
    }
}
```

## Заключение

Аннотация `@Modifying` является мощным инструментом для выполнения операций изменения данных в Spring Data JPA. Правильное использование её свойств и следование лучшим практикам обеспечивает эффективную и безопасную работу с базой данных.

### Основные рекомендации:

1. **Всегда используйте @Transactional** для модифицирующих операций
2. **Проверяйте возвращаемые значения** для валидации операций
3. **Используйте clearAutomatically** для очистки кэша при необходимости
4. **Применяйте flushAutomatically** для синхронизации с БД
5. **Логируйте массовые операции** для мониторинга
6. **Тестируйте модифицирующие операции** в unit-тестах
7. **Используйте правильные уровни изоляции** для критических операций
8. **Мониторьте производительность** больших операций
9. **Применяйте именованные параметры** для сложных запросов
10. **Используйте индексы** для оптимизации условий WHERE

