# Аннотация @Query

## Обзор

Аннотация `@Query` в Spring Data JPA позволяет определять собственные JPQL или нативные SQL запросы для методов репозитория. Это дает полный контроль над выполняемыми запросами.

## Основные возможности

### Типы запросов
- JPQL (Java Persistence Query Language)
- Нативные SQL запросы
- Именованные параметры
- Позиционные параметры
- Поддержка пагинации и сортировки

### Структура аннотации

```java
@Query(value = "JPQL или SQL запрос", 
       countQuery = "запрос для подсчета", 
       nativeQuery = false)
```

## JPQL запросы

### Базовые запросы
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE u.email = ?1")
    Optional<User> findByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.active = true")
    List<User> findActiveUsers();
    
    @Query("SELECT COUNT(u) FROM User u WHERE u.role = ?1")
    long countByRole(String role);
}
```

### Запросы с условиями
```java
@Query("SELECT u FROM User u WHERE u.email LIKE %?1% AND u.active = ?2")
List<User> findByEmailContainingAndActive(String email, boolean active);

@Query("SELECT u FROM User u WHERE u.createdDate BETWEEN ?1 AND ?2")
List<User> findByCreatedDateBetween(LocalDateTime start, LocalDateTime end);

@Query("SELECT u FROM User u WHERE u.role IN ?1")
List<User> findByRoles(List<String> roles);
```

### Запросы с JOIN
```java
@Query("SELECT u FROM User u JOIN u.orders o WHERE o.status = ?1")
List<User> findByOrderStatus(String status);

@Query("SELECT u FROM User u LEFT JOIN u.profile p WHERE p.city = ?1")
List<User> findByProfileCity(String city);

@Query("SELECT u FROM User u JOIN u.roles r WHERE r.name = ?1")
List<User> findByRoleName(String roleName);
```

## Нативные SQL запросы

### Включение нативных запросов
```java
@Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
Optional<User> findByEmailNative(String email);

@Query(value = "SELECT * FROM users WHERE active = ?1", nativeQuery = true)
List<User> findActiveUsersNative(boolean active);
```

### Сложные нативные запросы
```java
@Query(value = """
    SELECT u.*, COUNT(o.id) as order_count 
    FROM users u 
    LEFT JOIN orders o ON u.id = o.user_id 
    WHERE u.active = ?1 
    GROUP BY u.id 
    HAVING COUNT(o.id) > ?2
    """, nativeQuery = true)
List<User> findActiveUsersWithOrderCount(boolean active, int minOrders);
```

## Именованные параметры

### Использование @Param
```java
@Query("SELECT u FROM User u WHERE u.email = :email AND u.active = :active")
Optional<User> findByEmailAndActive(@Param("email") String email, 
                                   @Param("active") boolean active);

@Query("SELECT u FROM User u WHERE u.role = :role AND u.createdDate > :date")
List<User> findByRoleAndCreatedAfter(@Param("role") String role, 
                                    @Param("date") LocalDateTime date);
```

### Сложные именованные параметры
```java
@Query("""
    SELECT u FROM User u 
    WHERE (:email IS NULL OR u.email LIKE %:email%) 
    AND (:role IS NULL OR u.role = :role) 
    AND (:active IS NULL OR u.active = :active)
    """)
List<User> findUsersWithOptionalFilters(@Param("email") String email,
                                       @Param("role") String role,
                                       @Param("active") Boolean active);
```

## Пагинация и сортировка

### Запросы с Pageable
```java
@Query("SELECT u FROM User u WHERE u.role = ?1")
Page<User> findByRole(String role, Pageable pageable);

@Query("SELECT u FROM User u WHERE u.active = ?1")
Slice<User> findActiveUsers(boolean active, Pageable pageable);
```

### Запросы с Sort
```java
@Query("SELECT u FROM User u WHERE u.role = ?1")
List<User> findByRole(String role, Sort sort);

@Query("SELECT u FROM User u")
List<User> findAllUsers(Sort sort);
```

## Count запросы

### Кастомные count запросы
```java
@Query(value = "SELECT u FROM User u WHERE u.role = ?1",
       countQuery = "SELECT COUNT(u) FROM User u WHERE u.role = ?1")
Page<User> findByRole(String role, Pageable pageable);

@Query(value = """
    SELECT u.* FROM users u 
    JOIN user_roles ur ON u.id = ur.user_id 
    WHERE ur.role_name = ?1
    """,
    countQuery = """
    SELECT COUNT(*) FROM users u 
    JOIN user_roles ur ON u.id = ur.user_id 
    WHERE ur.role_name = ?1
    """,
    nativeQuery = true)
Page<User> findByRoleNative(String role, Pageable pageable);
```

## Проекции

### Class-based проекции
```java
@Query("SELECT new com.example.dto.UserSummary(u.id, u.email, u.role) " +
       "FROM User u WHERE u.active = ?1")
List<UserSummary> findUserSummaries(boolean active);
```

### Interface-based проекции
```java
@Query("SELECT u.email as email, u.role as role FROM User u WHERE u.active = ?1")
List<UserProjection> findUserProjections(boolean active);

public interface UserProjection {
    String getEmail();
    String getRole();
}
```

## SpEL в @Query

### Использование SpEL
```java
@Query("SELECT u FROM User u WHERE u.role = ?#{@securityService.getCurrentUserRole()}")
List<User> findUsersByCurrentUserRole();

@Query("SELECT u FROM User u WHERE u.createdDate > ?#{T(java.time.LocalDateTime).now().minusDays(7)}")
List<User> findUsersCreatedLastWeek();
```

### Кастомные SpEL функции
```java
@Query("SELECT u FROM User u WHERE u.email LIKE %?#{@emailService.getDomain()}")
List<User> findUsersByCurrentDomain();
```

## Обработка результатов

### Типы возвращаемых значений
```java
@Query("SELECT u FROM User u WHERE u.id = ?1")
Optional<User> findById(Long id);

@Query("SELECT u.email FROM User u WHERE u.role = ?1")
List<String> findEmailsByRole(String role);

@Query("SELECT COUNT(u) FROM User u WHERE u.active = ?1")
long countActiveUsers(boolean active);

@Query("SELECT u.role FROM User u WHERE u.id = ?1")
String findRoleById(Long id);
```

## Лучшие практики

### Производительность
```java
// Хорошо - используйте индексы
@Query("SELECT u FROM User u WHERE u.email = ?1")
Optional<User> findByEmail(String email);

// Избегайте N+1 проблем
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = ?1")
Optional<User> findByIdWithOrders(Long id);
```

### Безопасность
```java
// Используйте параметризованные запросы
@Query("SELECT u FROM User u WHERE u.email = ?1")
Optional<User> findByEmail(String email);

// Избегайте конкатенации строк
// ПЛОХО: @Query("SELECT u FROM User u WHERE u.email = '" + email + "'")
```

### Читаемость
```java
// Используйте многострочные строки для сложных запросов
@Query("""
    SELECT u FROM User u 
    JOIN u.orders o 
    WHERE u.active = ?1 
    AND o.status = ?2 
    AND o.createdDate > ?3
    """)
List<User> findActiveUsersWithRecentOrders(boolean active, String status, LocalDateTime date);
```

## Отладка и логирование

### Включение логирования
```properties
logging.level.org.springframework.data.jpa.repository.query=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### Проверка сгенерированных запросов
```java
@Component
public class QueryDebugger {
    
    @EventListener(ApplicationReadyEvent.class)
    public void debugQueries() {
        // Логирование сгенерированных запросов
    }
}
```

## Лучшие практики

1. **Используйте JPQL** для переносимости между БД
2. **Применяйте именованные параметры** для сложных запросов
3. **Используйте многострочные строки** для читаемости
4. **Оптимизируйте запросы** с помощью JOIN FETCH
5. **Тестируйте запросы** в unit-тестах
6. **Мониторьте производительность** сложных запросов 