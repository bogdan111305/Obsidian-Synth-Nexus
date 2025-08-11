# @Query в Spring Data JPA

## Обзор

Аннотация `@Query` в Spring Data JPA позволяет определять собственные JPQL или нативные SQL запросы для методов репозитория. Это дает полный контроль над выполняемыми запросами и является одним из самых мощных инструментов для создания кастомных запросов.

## Основные возможности

### Типы запросов
- **JPQL** (Java Persistence Query Language) - портабельные запросы
- **Нативные SQL запросы** - специфичные для БД
- **Именованные параметры** - для лучшей читаемости
- **Позиционные параметры** - для простых случаев
- **Поддержка пагинации и сортировки**
- **Хранимые процедуры** - для сложной бизнес-логики

### Структура аннотации

```java
@Query(value = "JPQL или SQL запрос", 
       countQuery = "запрос для подсчета", 
       nativeQuery = false)
```

## StoredProcedureJpaQuery и @Procedure

### Определение хранимой процедуры

```sql
-- Создание хранимой процедуры в MySQL
DELIMITER //
CREATE PROCEDURE GetUsersByDepartment(IN deptId BIGINT)
BEGIN
    SELECT * FROM users WHERE department_id = deptId;
END //
DELIMITER ;

-- Создание хранимой процедуры в PostgreSQL
CREATE OR REPLACE FUNCTION get_users_by_department(dept_id BIGINT)
RETURNS TABLE(id BIGINT, name VARCHAR(255), email VARCHAR(255)) AS $$
BEGIN
    RETURN QUERY SELECT u.id, u.name, u.email 
                 FROM users u 
                 WHERE u.department_id = dept_id;
END;
$$ LANGUAGE plpgsql;
```

### Использование @Procedure

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Вызов хранимой процедуры по имени
    @Procedure("get_users_by_department")
    List<User> getUsersByDepartment(@Param("dept_id") Long deptId);
    
    // Вызов хранимой процедуры с указанием имени метода
    @Procedure(name = "get_users_by_department")
    List<User> findUsersByDepartment(@Param("dept_id") Long deptId);
    
    // Вызов хранимой процедуры с указанием имени процедуры
    @Procedure(procedureName = "get_users_by_department")
    List<User> callGetUsersByDepartment(@Param("dept_id") Long deptId);
}
```

### Хранимые процедуры с выходными параметрами

```sql
-- Процедура с выходным параметром
DELIMITER //
CREATE PROCEDURE GetUserCountByDepartment(
    IN deptId BIGINT,
    OUT userCount INT
)
BEGIN
    SELECT COUNT(*) INTO userCount 
    FROM users 
    WHERE department_id = deptId;
END //
DELIMITER ;
```

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Procedure(name = "GetUserCountByDepartment")
    Integer getUserCountByDepartment(@Param("deptId") Long deptId, 
                                   @Param("userCount") Integer userCount);
}
```

### Хранимые процедуры с множественными результатами

```sql
-- Процедура, возвращающая несколько результатов
DELIMITER //
CREATE PROCEDURE GetDepartmentStats(IN deptId BIGINT)
BEGIN
    SELECT COUNT(*) as user_count FROM users WHERE department_id = deptId;
    SELECT AVG(salary) as avg_salary FROM users WHERE department_id = deptId;
    SELECT MAX(salary) as max_salary FROM users WHERE department_id = deptId;
END //
DELIMITER ;
```

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Procedure(name = "GetDepartmentStats")
    List<Object[]> getDepartmentStats(@Param("deptId") Long deptId);
}
```

## JPQL запросы

### Базовые запросы

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простой JPQL запрос
    @Query("SELECT u FROM User u WHERE u.name = :name")
    List<User> findByName(@Param("name") String name);
    
    // Запрос с несколькими параметрами
    @Query("SELECT u FROM User u WHERE u.age > :minAge AND u.department.name = :deptName")
    List<User> findByAgeAndDepartment(@Param("minAge") int minAge, 
                                     @Param("deptName") String deptName);
    
    // Запрос с сортировкой
    @Query("SELECT u FROM User u WHERE u.active = true ORDER BY u.name ASC")
    List<User> findActiveUsersOrdered();
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
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Запрос с JOIN
    @Query("SELECT u FROM User u JOIN u.department d WHERE d.name = :deptName")
    List<User> findByDepartmentName(@Param("deptName") String deptName);
    
    // Запрос с LEFT JOIN
    @Query("SELECT u FROM User u LEFT JOIN u.roles r WHERE r.name = :roleName")
    List<User> findByRoleName(@Param("roleName") String roleName);
    
    // Запрос с подзапросом
    @Query("SELECT u FROM User u WHERE u.salary > (SELECT AVG(salary) FROM User)")
    List<User> findUsersWithAboveAverageSalary();
    
    // Запрос с EXISTS
    @Query("SELECT u FROM User u WHERE EXISTS (SELECT 1 FROM u.roles r WHERE r.name = :roleName)")
    List<User> findUsersWithRole(@Param("roleName") String roleName);
}
```

### Запросы с агрегациями

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Подсчет количества
    @Query("SELECT COUNT(u) FROM User u WHERE u.department.id = :deptId")
    long countByDepartment(@Param("deptId") Long deptId);
    
    // Среднее значение
    @Query("SELECT AVG(u.salary) FROM User u WHERE u.department.id = :deptId")
    Double getAverageSalaryByDepartment(@Param("deptId") Long deptId);
    
    // Максимальное значение
    @Query("SELECT MAX(u.salary) FROM User u WHERE u.department.id = :deptId")
    Double getMaxSalaryByDepartment(@Param("deptId") Long deptId);
    
    // Группировка с агрегацией
    @Query("SELECT u.department.name, COUNT(u), AVG(u.salary) " +
           "FROM User u " +
           "GROUP BY u.department.name " +
           "HAVING COUNT(u) > :minCount")
    List<Object[]> getDepartmentStats(@Param("minCount") long minCount);
}
```

## Усовершенствованный оператор LIKE

### Различные варианты LIKE

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Точное совпадение
    @Query("SELECT u FROM User u WHERE u.name = :name")
    List<User> findByNameExact(@Param("name") String name);
    
    // Начинается с
    @Query("SELECT u FROM User u WHERE u.name LIKE :name%")
    List<User> findByNameStartingWith(@Param("name") String name);
    
    // Заканчивается на
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name")
    List<User> findByNameEndingWith(@Param("name") String name);
    
    // Содержит
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name%")
    List<User> findByNameContaining(@Param("name") String name);
    
    // Не содержит
    @Query("SELECT u FROM User u WHERE u.name NOT LIKE %:name%")
    List<User> findByNameNotContaining(@Param("name") String name);
    
    // С использованием ESCAPE для специальных символов
    @Query("SELECT u FROM User u WHERE u.name LIKE :pattern ESCAPE '\\'")
    List<User> findByNameWithPattern(@Param("pattern") String pattern);
}
```

### Поиск по нескольким полям

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по нескольким полям с OR
    @Query("SELECT u FROM User u WHERE " +
           "u.name LIKE %:searchTerm% OR " +
           "u.email LIKE %:searchTerm% OR " +
           "u.department.name LIKE %:searchTerm%")
    List<User> searchUsers(@Param("searchTerm") String searchTerm);
    
    // Поиск по нескольким полям с AND
    @Query("SELECT u FROM User u WHERE " +
           "u.name LIKE %:name% AND " +
           "u.email LIKE %:email%")
    List<User> findByNameAndEmail(@Param("name") String name, @Param("email") String email);
    
    // Поиск с игнорированием регистра
    @Query("SELECT u FROM User u WHERE LOWER(u.name) LIKE LOWER(CONCAT('%', :name, '%'))")
    List<User> findByNameIgnoreCase(@Param("name") String name);
}
```

### Поиск с использованием регулярных выражений

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // MySQL REGEXP
    @Query(value = "SELECT * FROM users WHERE name REGEXP :pattern", nativeQuery = true)
    List<User> findByNameRegex(@Param("pattern") String pattern);
    
    // PostgreSQL ~
    @Query(value = "SELECT * FROM users WHERE name ~ :pattern", nativeQuery = true)
    List<User> findByNameRegexPostgres(@Param("pattern") String pattern);
    
    // Oracle REGEXP_LIKE
    @Query(value = "SELECT * FROM users WHERE REGEXP_LIKE(name, :pattern)", nativeQuery = true)
    List<User> findByNameRegexOracle(@Param("pattern") String pattern);
}
```

## Нативные SQL запросы

### Базовые нативные запросы

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Простой native query
    @Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
    List<User> findByNameNative(@Param("name") String name);
    
    // Native query с JOIN
    @Query(value = "SELECT u.* FROM users u " +
                   "JOIN departments d ON u.department_id = d.id " +
                   "WHERE d.name = :deptName", nativeQuery = true)
    List<User> findByDepartmentNameNative(@Param("deptName") String deptName);
    
    // Native query с агрегацией
    @Query(value = "SELECT d.name, COUNT(u.id) as user_count " +
                   "FROM departments d " +
                   "LEFT JOIN users u ON d.id = u.department_id " +
                   "GROUP BY d.id, d.name", nativeQuery = true)
    List<Object[]> getDepartmentUserCounts();
}
```

### Сложные нативные запросы

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Native query с оконными функциями
    @Query(value = "SELECT u.*, " +
                   "ROW_NUMBER() OVER (ORDER BY u.salary DESC) as salary_rank " +
                   "FROM users u", nativeQuery = true)
    List<Object[]> findUsersWithSalaryRank();
    
    // Native query с подзапросом
    @Query(value = "SELECT * FROM users u " +
                   "WHERE u.salary > (SELECT AVG(salary) FROM users)", nativeQuery = true)
    List<User> findUsersWithAboveAverageSalaryNative();
    
    // Native query с CASE
    @Query(value = "SELECT u.*, " +
                   "CASE WHEN u.salary > 100000 THEN 'HIGH' " +
                   "     WHEN u.salary > 50000 THEN 'MEDIUM' " +
                   "     ELSE 'LOW' END as salary_level " +
                   "FROM users u", nativeQuery = true)
    List<Object[]> findUsersWithSalaryLevel();
}
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

### Native query с пагинацией

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Native query с пагинацией
    @Query(value = "SELECT * FROM users WHERE department_id = :deptId",
           countQuery = "SELECT COUNT(*) FROM users WHERE department_id = :deptId",
           nativeQuery = true)
    Page<User> findByDepartmentNative(@Param("deptId") Long deptId, Pageable pageable);
    
    // Native query с сортировкой
    @Query(value = "SELECT * FROM users ORDER BY name ASC", nativeQuery = true)
    List<User> findAllOrderedByNameNative();
}
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

// Класс проекции
public class UserSummary {
    private String name;
    private String email;
    
    public UserSummary(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    // getters and setters
}
```

### Native query с проекциями

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Native query с проекцией на интерфейс
    @Query(value = "SELECT u.name as name, u.email as email FROM users u WHERE u.department_id = :deptId",
           nativeQuery = true)
    List<UserProjection> findUserProjectionsByDepartment(@Param("deptId") Long deptId);
    
    // Native query с проекцией на класс
    @Query(value = "SELECT u.name as name, u.email as email FROM users u WHERE u.department_id = :deptId",
           nativeQuery = true)
    List<UserSummary> findUserSummariesByDepartment(@Param("deptId") Long deptId);
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

## Оптимизация производительности

### Использование JOIN FETCH

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Избегает N+1 проблему
    @Query("SELECT u FROM User u JOIN FETCH u.department JOIN FETCH u.roles")
    List<User> findAllWithDepartmentAndRoles();
    
    // Условный JOIN FETCH
    @Query("SELECT u FROM User u " +
           "LEFT JOIN FETCH u.department " +
           "LEFT JOIN FETCH u.roles " +
           "WHERE u.active = :active")
    List<User> findUsersByActiveStatus(@Param("active") boolean active);
}
```

### Использование @QueryHints

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "50"),
        @QueryHint(name = HINT_CACHEABLE, value = "true"),
        @QueryHint(name = HINT_COMMENT, value = "User search query")
    })
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name%")
    List<User> findByNameContaining(@Param("name") String name);
}
```

### Использование пагинации

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Пагинированный поиск
    @Query("SELECT u FROM User u WHERE u.department.id = :deptId")
    Page<User> findByDepartment(@Param("deptId") Long deptId, Pageable pageable);
    
    // Slice для больших наборов данных
    @Query("SELECT u FROM User u WHERE u.active = true")
    Slice<User> findActiveUsers(Pageable pageable);
}
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

### 1. Именование параметров

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // ✅ Хорошие имена параметров
    @Query("SELECT u FROM User u WHERE u.name = :name AND u.age = :age")
    List<User> findByNameAndAge(@Param("name") String name, @Param("age") int age);
    
    // ❌ Плохие имена параметров
    @Query("SELECT u FROM User u WHERE u.name = :n AND u.age = :a")
    List<User> findByNameAndAge(@Param("n") String name, @Param("a") int age);
}
```

### 2. Использование Optional

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // ✅ Использование Optional для одиночных результатов
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmail(@Param("email") String email);
    
    // ✅ List для множественных результатов
    @Query("SELECT u FROM User u WHERE u.department.id = :deptId")
    List<User> findByDepartment(@Param("deptId") Long deptId);
}
```

### 3. Безопасность от SQL инъекций

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // ✅ Безопасно - параметры экранируются автоматически
    @Query("SELECT u FROM User u WHERE u.name = :name")
    List<User> findByName(@Param("name") String name);
    
    // ❌ Опасно - не используйте конкатенацию строк
    // @Query("SELECT u FROM User u WHERE u.name = '" + name + "'")
    // List<User> findByNameUnsafe(String name);
}
```

### 4. Читаемость

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

## Заключение

`@Query` предоставляет мощный и гибкий способ создания кастомных запросов в Spring Data JPA. Понимание различных типов запросов, оптимизации производительности и лучших практик позволяет создавать эффективные и безопасные запросы для работы с базой данных.

### Основные рекомендации:

1. **Используйте JPQL** для переносимости между БД
2. **Применяйте именованные параметры** для сложных запросов  
3. **Используйте многострочные строки** для читаемости
4. **Оптимизируйте запросы** с помощью JOIN FETCH
5. **Тестируйте запросы** в unit-тестах
6. **Мониторьте производительность** сложных запросов
7. **Используйте хранимые процедуры** для сложной бизнес-логики
8. **Применяйте проекции** для оптимизации передачи данных

