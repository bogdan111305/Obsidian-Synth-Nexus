# Аннотация @Modifying

## Обзор

Аннотация `@Modifying` в Spring Data JPA используется для обозначения методов, которые изменяют данные в базе данных (INSERT, UPDATE, DELETE). Она должна использоваться вместе с аннотацией `@Query` для методов, которые не являются стандартными CRUD операциями.

## Основные возможности

### Типы модифицирующих операций
- UPDATE запросы
- DELETE запросы
- Массовые операции
- Кастомные модификации

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

### DELETE с JOIN
```java
@Modifying
@Query("DELETE FROM User u WHERE u.id IN (SELECT u.id FROM User u JOIN u.orders o WHERE o.status = ?1)")
int deleteUsersWithOrdersByStatus(String orderStatus);
```

## Свойства @Modifying

### clearAutomatically
```java
@Modifying(clearAutomatically = true)
@Query("UPDATE User u SET u.active = ?2 WHERE u.id = ?1")
int updateUserStatus(Long id, boolean active);
```

**clearAutomatically = true** - автоматически очищает кэш первого уровня после выполнения операции.

### flushAutomatically
```java
@Modifying(flushAutomatically = true)
@Query("UPDATE User u SET u.email = ?2 WHERE u.id = ?1")
int updateUserEmail(Long id, String email);
```

**flushAutomatically = true** - автоматически сбрасывает изменения в базу данных перед выполнением операции.

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
        int updatedRows = userRepository.updateUserStatus(id, active);
        logger.info("Updated {} users with id {}", updatedRows, id);
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
}
```

## Лучшие практики

### Безопасность
```java
// Хорошо - используйте параметризованные запросы
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.id = ?2")
int updateUserStatus(boolean active, Long id);

// Плохо - избегайте конкатенации строк
// @Query("UPDATE User u SET u.active = " + active + " WHERE u.id = " + id)
```

### Производительность
```java
// Используйте индексы для условий WHERE
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.email = ?2")
int updateUserStatusByEmail(boolean active, String email);

// Избегайте сканирования всей таблицы
@Modifying
@Query("UPDATE User u SET u.active = ?1 WHERE u.id IN ?2")
int updateUserStatusByIds(boolean active, List<Long> ids);
```

### Транзакционность
```java
@Modifying
@Transactional
@Query("UPDATE User u SET u.active = ?2 WHERE u.id = ?1")
int updateUserStatus(Long id, boolean active);

// Используйте правильные уровни изоляции
@Modifying
@Transactional(isolation = Isolation.SERIALIZABLE)
@Query("UPDATE User u SET u.balance = u.balance - ?1 WHERE u.id = ?2")
int deductBalance(BigDecimal amount, Long userId);
```

## Отладка и логирование

### Включение логирования
```properties
logging.level.org.springframework.data.jpa.repository.query=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### Мониторинг производительности
```java
@Component
public class ModifyingQueryMonitor {
    
    private final Map<String, Long> executionTimes = new ConcurrentHashMap<>();
    
    public void monitorExecution(String methodName, Supplier<Integer> execution) {
        long startTime = System.currentTimeMillis();
        try {
            int result = execution.get();
            long executionTime = System.currentTimeMillis() - startTime;
            executionTimes.merge(methodName, executionTime, Long::sum);
            
            log.info("Method {} affected {} rows in {} ms", methodName, result, executionTime);
        } catch (Exception e) {
            log.error("Error executing method {}", methodName, e);
            throw e;
        }
    }
}
```

## Лучшие практики

1. **Всегда используйте @Transactional** для модифицирующих операций
2. **Проверяйте возвращаемые значения** для валидации операций
3. **Используйте clearAutomatically** для очистки кэша
4. **Применяйте flushAutomatically** для синхронизации с БД
5. **Логируйте массовые операции** для мониторинга
6. **Тестируйте модифицирующие операции** в unit-тестах
7. **Используйте правильные уровни изоляции** для критических операций 