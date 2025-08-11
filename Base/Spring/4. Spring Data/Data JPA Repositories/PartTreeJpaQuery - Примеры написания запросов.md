# Примеры написания запросов

## Обзор

В Spring Data JPA запросы могут быть написаны различными способами. PartTreeJpaQuery позволяет создавать запросы автоматически на основе анализа имени метода. В этой статье рассмотрим подробные примеры написания различных типов запросов.

## Базовые принципы

### Структура имени метода
```
[префикс][By][поле][оператор][And/Or][поле][оператор]...
```

### Поддерживаемые префиксы
- `find` - поиск сущностей
- `get` - получение сущностей
- `read` - чтение сущностей
- `query` - запрос сущностей
- `search` - поиск сущностей
- `count` - подсчет количества
- `exists` - проверка существования
- `delete` - удаление сущностей
- `remove` - удаление сущностей

### Поддерживаемые операторы
- `Equals` (по умолчанию)
- `NotEquals`
- `Like`
- `NotLike`
- `Containing`
- `NotContaining`
- `StartingWith`
- `EndingWith`
- `Between`
- `LessThan`
- `GreaterThan`
- `LessThanEqual`
- `GreaterThanEqual`
- `In`
- `NotIn`
- `IsNull`
- `IsNotNull`
- `True`
- `False`
- `IgnoreCase`

## Примеры запросов

### Простые запросы поиска

#### Поиск по одному полю
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по email
    List<User> findByEmail(String email);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email
    
    // Поиск по username
    Optional<User> findByUsername(String username);
    // Генерируется: SELECT e FROM User e WHERE e.username = :username
    
    // Поиск по ID
    User findById(Long id);
    // Генерируется: SELECT e FROM User e WHERE e.id = :id
    
    // Поиск по имени
    List<User> findByName(String name);
    // Генерируется: SELECT e FROM User e WHERE e.name = :name
}
```

#### Поиск с различными операторами
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск с LIKE
    List<User> findByEmailLike(String emailPattern);
    // Генерируется: SELECT e FROM User e WHERE e.email LIKE :emailPattern
    
    // Поиск с CONTAINING
    List<User> findByEmailContaining(String emailPart);
    // Генерируется: SELECT e FROM User e WHERE e.email LIKE %:emailPart%
    
    // Поиск с STARTING WITH
    List<User> findByEmailStartingWith(String emailPrefix);
    // Генерируется: SELECT e FROM User e WHERE e.email LIKE :emailPrefix%
    
    // Поиск с ENDING WITH
    List<User> findByEmailEndingWith(String emailSuffix);
    // Генерируется: SELECT e FROM User e WHERE e.email LIKE %:emailSuffix
    
    // Поиск с игнорированием регистра
    List<User> findByEmailIgnoreCase(String email);
    // Генерируется: SELECT e FROM User e WHERE LOWER(e.email) = LOWER(:email)
}
```

### Сложные запросы с множественными условиями

#### Запросы с AND
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по email И username
    User findByEmailAndUsername(String email, String username);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email AND e.username = :username
    
    // Поиск по email И username И active
    List<User> findByEmailAndUsernameAndActiveTrue(String email, String username);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email AND e.username = :username AND e.active = true
    
    // Поиск по имени И возрасту
    List<User> findByNameAndAge(String name, int age);
    // Генерируется: SELECT e FROM User e WHERE e.name = :name AND e.age = :age
    
    // Поиск по email И роли
    List<User> findByEmailAndRole(String email, Role role);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email AND e.role = :role
}
```

#### Запросы с OR
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по email ИЛИ username
    List<User> findByEmailOrUsername(String email, String username);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email OR e.username = :username
    
    // Поиск по имени ИЛИ фамилии
    List<User> findByNameOrLastName(String name, String lastName);
    // Генерируется: SELECT e FROM User e WHERE e.name = :name OR e.lastName = :lastName
    
    // Поиск по email ИЛИ роли
    List<User> findByEmailOrRole(String email, Role role);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email OR e.role = :role
}
```

#### Комбинированные запросы
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Сложный запрос с AND и OR
    List<User> findByEmailAndUsernameOrRole(String email, String username, Role role);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email AND e.username = :username OR e.role = :role
    
    // Поиск по email И (username ИЛИ role)
    List<User> findByEmailAndUsernameOrEmailAndRole(String email, String username, Role role);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email AND e.username = :username OR e.email = :email AND e.role = :role
}
```

### Запросы с операторами сравнения

#### Числовые сравнения
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по возрасту больше
    List<User> findByAgeGreaterThan(int age);
    // Генерируется: SELECT e FROM User e WHERE e.age > :age
    
    // Поиск по возрасту меньше
    List<User> findByAgeLessThan(int age);
    // Генерируется: SELECT e FROM User e WHERE e.age < :age
    
    // Поиск по возрасту больше или равно
    List<User> findByAgeGreaterThanEqual(int age);
    // Генерируется: SELECT e FROM User e WHERE e.age >= :age
    
    // Поиск по возрасту меньше или равно
    List<User> findByAgeLessThanEqual(int age);
    // Генерируется: SELECT e FROM User e WHERE e.age <= :age
    
    // Поиск по возрасту между
    List<User> findByAgeBetween(int minAge, int maxAge);
    // Генерируется: SELECT e FROM User e WHERE e.age BETWEEN :minAge AND :maxAge
}
```

#### Сравнения дат
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по дате создания после
    List<User> findByCreatedAtGreaterThan(LocalDateTime date);
    // Генерируется: SELECT e FROM User e WHERE e.createdAt > :date
    
    // Поиск по дате создания до
    List<User> findByCreatedAtLessThan(LocalDateTime date);
    // Генерируется: SELECT e FROM User e WHERE e.createdAt < :date
    
    // Поиск по дате создания между
    List<User> findByCreatedAtBetween(LocalDateTime startDate, LocalDateTime endDate);
    // Генерируется: SELECT e FROM User e WHERE e.createdAt BETWEEN :startDate AND :endDate
    
    // Поиск по дате создания сегодня
    List<User> findByCreatedAtToday();
    // Генерируется: SELECT e FROM User e WHERE DATE(e.createdAt) = CURRENT_DATE
}
```

### Запросы с коллекциями

#### Операторы IN и NOT IN
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по ролям в списке
    List<User> findByRoleIn(List<Role> roles);
    // Генерируется: SELECT e FROM User e WHERE e.role IN :roles
    
    // Поиск по ролям не в списке
    List<User> findByRoleNotIn(List<Role> roles);
    // Генерируется: SELECT e FROM User e WHERE e.role NOT IN :roles
    
    // Поиск по ID в списке
    List<User> findByIdIn(List<Long> ids);
    // Генерируется: SELECT e FROM User e WHERE e.id IN :ids
    
    // Поиск по email в списке
    List<User> findByEmailIn(List<String> emails);
    // Генерируется: SELECT e FROM User e WHERE e.email IN :emails
}
```

### Запросы с null проверками

#### Проверки на null
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск пользователей без email
    List<User> findByEmailIsNull();
    // Генерируется: SELECT e FROM User e WHERE e.email IS NULL
    
    // Поиск пользователей с email
    List<User> findByEmailIsNotNull();
    // Генерируется: SELECT e FROM User e WHERE e.email IS NOT NULL
    
    // Поиск пользователей без телефона
    List<User> findByPhoneIsNull();
    // Генерируется: SELECT e FROM User e WHERE e.phone IS NULL
    
    // Поиск пользователей с телефоном
    List<User> findByPhoneIsNotNull();
    // Генерируется: SELECT e FROM User e WHERE e.phone IS NOT NULL
}
```

### Запросы с булевыми значениями

#### Булевые операторы
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск активных пользователей
    List<User> findByActiveTrue();
    // Генерируется: SELECT e FROM User e WHERE e.active = true
    
    // Поиск неактивных пользователей
    List<User> findByActiveFalse();
    // Генерируется: SELECT e FROM User e WHERE e.active = false
    
    // Поиск подтвержденных пользователей
    List<User> findByVerifiedTrue();
    // Генерируется: SELECT e FROM User e WHERE e.verified = true
    
    // Поиск неподтвержденных пользователей
    List<User> findByVerifiedFalse();
    // Генерируется: SELECT e FROM User e WHERE e.verified = false
}
```

### Запросы с сортировкой

#### Сортировка по одному полю
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск всех пользователей с сортировкой по имени
    List<User> findAllByOrderByNameAsc();
    // Генерируется: SELECT e FROM User e ORDER BY e.name ASC
    
    // Поиск всех пользователей с сортировкой по имени (по убыванию)
    List<User> findAllByOrderByNameDesc();
    // Генерируется: SELECT e FROM User e ORDER BY e.name DESC
    
    // Поиск активных пользователей с сортировкой по email
    List<User> findByActiveTrueOrderByEmailAsc();
    // Генерируется: SELECT e FROM User e WHERE e.active = true ORDER BY e.email ASC
}
```

#### Сортировка по нескольким полям
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск с сортировкой по имени и фамилии
    List<User> findAllByOrderByNameAscLastNameAsc();
    // Генерируется: SELECT e FROM User e ORDER BY e.name ASC, e.lastName ASC
    
    // Поиск с сортировкой по имени (по возрастанию) и возрасту (по убыванию)
    List<User> findAllByOrderByNameAscAgeDesc();
    // Генерируется: SELECT e FROM User e ORDER BY e.name ASC, e.age DESC
    
    // Поиск активных пользователей с сортировкой по дате создания
    List<User> findByActiveTrueOrderByCreatedAtDesc();
    // Генерируется: SELECT e FROM User e WHERE e.active = true ORDER BY e.createdAt DESC
}
```

### Запросы с ограничением результатов

#### Ограничение количества результатов
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск первых 10 пользователей
    List<User> findTop10ByOrderByCreatedAtDesc();
    // Генерируется: SELECT e FROM User e ORDER BY e.createdAt DESC LIMIT 10
    
    // Поиск первых 5 активных пользователей
    List<User> findTop5ByActiveTrueOrderByCreatedAtDesc();
    // Генерируется: SELECT e FROM User e WHERE e.active = true ORDER BY e.createdAt DESC LIMIT 5
    
    // Поиск первого пользователя по email
    User findFirstByEmail(String email);
    // Генерируется: SELECT e FROM User e WHERE e.email = :email LIMIT 1
    
    // Поиск первого активного пользователя
    User findFirstByActiveTrue();
    // Генерируется: SELECT e FROM User e WHERE e.active = true LIMIT 1
}
```

### Запросы подсчета

#### Подсчет количества
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Подсчет всех пользователей
    long count();
    // Генерируется: SELECT COUNT(e) FROM User e
    
    // Подсчет активных пользователей
    long countByActiveTrue();
    // Генерируется: SELECT COUNT(e) FROM User e WHERE e.active = true
    
    // Подсчет пользователей по email
    long countByEmail(String email);
    // Генерируется: SELECT COUNT(e) FROM User e WHERE e.email = :email
    
    // Подсчет пользователей по роли
    long countByRole(Role role);
    // Генерируется: SELECT COUNT(e) FROM User e WHERE e.role = :role
    
    // Подсчет пользователей старше определенного возраста
    long countByAgeGreaterThan(int age);
    // Генерируется: SELECT COUNT(e) FROM User e WHERE e.age > :age
}
```

### Запросы проверки существования

#### Проверка существования
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Проверка существования пользователя по email
    boolean existsByEmail(String email);
    // Генерируется: SELECT COUNT(e) > 0 FROM User e WHERE e.email = :email
    
    // Проверка существования пользователя по username
    boolean existsByUsername(String username);
    // Генерируется: SELECT COUNT(e) > 0 FROM User e WHERE e.username = :username
    
    // Проверка существования активных пользователей
    boolean existsByActiveTrue();
    // Генерируется: SELECT COUNT(e) > 0 FROM User e WHERE e.active = true
    
    // Проверка существования пользователей по роли
    boolean existsByRole(Role role);
    // Генерируется: SELECT COUNT(e) > 0 FROM User e WHERE e.role = :role
}
```

### Запросы удаления

#### Удаление сущностей
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Удаление пользователя по email
    void deleteByEmail(String email);
    // Генерируется: DELETE FROM User e WHERE e.email = :email
    
    // Удаление пользователя по username
    void deleteByUsername(String username);
    // Генерируется: DELETE FROM User e WHERE e.username = :username
    
    // Удаление неактивных пользователей
    void deleteByActiveFalse();
    // Генерируется: DELETE FROM User e WHERE e.active = false
    
    // Удаление пользователей по роли
    void deleteByRole(Role role);
    // Генерируется: DELETE FROM User e WHERE e.role = :role
    
    // Удаление пользователей старше определенного возраста
    void deleteByAgeGreaterThan(int age);
    // Генерируется: DELETE FROM User e WHERE e.age > :age
}
```

## Сложные примеры

### Запросы с вложенными свойствами
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по адресу города
    List<User> findByAddressCity(String city);
    // Генерируется: SELECT e FROM User e WHERE e.address.city = :city
    
    // Поиск по адресу улицы
    List<User> findByAddressStreet(String street);
    // Генерируется: SELECT e FROM User e WHERE e.address.street = :street
    
    // Поиск по адресу города и улицы
    List<User> findByAddressCityAndAddressStreet(String city, String street);
    // Генерируется: SELECT e FROM User e WHERE e.address.city = :city AND e.address.street = :street
    
    // Поиск по профилю роли
    List<User> findByProfileRole(Role role);
    // Генерируется: SELECT e FROM User e WHERE e.profile.role = :role
}
```

### Запросы с коллекциями вложенных объектов
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск по заказам статуса
    List<User> findByOrdersStatus(OrderStatus status);
    // Генерируется: SELECT e FROM User e JOIN e.orders o WHERE o.status = :status
    
    // Поиск по заказам даты
    List<User> findByOrdersCreatedAtBetween(LocalDateTime startDate, LocalDateTime endDate);
    // Генерируется: SELECT e FROM User e JOIN e.orders o WHERE o.createdAt BETWEEN :startDate AND :endDate
    
    // Поиск по заказам суммы
    List<User> findByOrdersTotalGreaterThan(BigDecimal amount);
    // Генерируется: SELECT e FROM User e JOIN e.orders o WHERE o.total > :amount
}
```

### Запросы с агрегатными функциями
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Поиск пользователей с максимальным возрастом
    List<User> findByAgeOrderByAgeDesc();
    // Генерируется: SELECT e FROM User e ORDER BY e.age DESC
    
    // Поиск пользователей с минимальным возрастом
    List<User> findByAgeOrderByAgeAsc();
    // Генерируется: SELECT e FROM User e ORDER BY e.age ASC
    
    // Поиск пользователей по дате создания (последние)
    List<User> findByCreatedAtOrderByCreatedAtDesc();
    // Генерируется: SELECT e FROM User e ORDER BY e.createdAt DESC
}
```

## Лучшие практики

### 1. Именование методов
```java
// Правильно - четкие и понятные имена
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
    List<User> findByActiveTrue();
    long countByActiveTrue();
    boolean existsByEmail(String email);
}

// Неправильно - неясные имена
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> find(String email); // Неясно, что ищется
    User get(String username); // Неясно, что получается
    List<User> findActive(); // Неясно, что означает "active"
}
```

### 2. Оптимизация производительности
```java
// Используйте индексы для часто используемых полей
@Entity
@Table(indexes = {
    @Index(name = "idx_user_email", columnList = "email"),
    @Index(name = "idx_user_username", columnList = "username"),
    @Index(name = "idx_user_active", columnList = "active"),
    @Index(name = "idx_user_created_at", columnList = "created_at")
})
public class User {
    // поля
}

// Используйте ограничения для больших результатов
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findTop100ByActiveTrueOrderByCreatedAtDesc();
    List<User> findTop50ByRoleOrderByNameAsc(Role role);
}
```

### 3. Валидация параметров
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> findByEmail(String email) {
        if (email == null || email.trim().isEmpty()) {
            throw new IllegalArgumentException("Email cannot be null or empty");
        }
        return userRepository.findByEmail(email);
    }
    
    public List<User> findByAgeBetween(int minAge, int maxAge) {
        if (minAge < 0 || maxAge < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
        if (minAge > maxAge) {
            throw new IllegalArgumentException("Min age cannot be greater than max age");
        }
        return userRepository.findByAgeBetween(minAge, maxAge);
    }
}
```

### 4. Обработка результатов
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public Optional<User> findByUsername(String username) {
        try {
            return userRepository.findByUsername(username);
        } catch (Exception e) {
            log.error("Error finding user by username: " + username, e);
            return Optional.empty();
        }
    }
    
    public List<User> findByActiveTrue() {
        try {
            return userRepository.findByActiveTrue();
        } catch (Exception e) {
            log.error("Error finding active users", e);
            return Collections.emptyList();
        }
    }
    
    public long countActiveUsers() {
        try {
            return userRepository.countByActiveTrue();
        } catch (Exception e) {
            log.error("Error counting active users", e);
            return 0L;
        }
    }
}
```

## Заключение

PartTreeJpaQuery предоставляет мощный и гибкий способ создания запросов в Spring Data JPA. Правильное использование конвенций именования позволяет создавать эффективные и читаемые запросы без написания SQL или JPQL вручную.

Ключевые принципы:
- Используйте четкие и понятные имена методов
- Следуйте конвенциям именования Spring Data JPA
- Оптимизируйте производительность через индексы и ограничения
- Добавляйте валидацию параметров в сервисном слое
- Правильно обрабатывайте результаты и исключения
