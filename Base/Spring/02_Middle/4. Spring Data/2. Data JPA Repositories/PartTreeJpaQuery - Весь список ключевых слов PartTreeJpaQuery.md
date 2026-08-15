# Весь список ключевых слов PartTreeJpaQuery

## Обзор

PartTreeJpaQuery поддерживает множество ключевых слов для создания различных типов запросов. В этой статье представлен полный список всех поддерживаемых ключевых слов с подробными примерами и объяснениями.

## Ключевые слова для префиксов

### Поиск и получение данных
```java
// find - поиск сущностей
List<User> findByEmail(String email);
Optional<User> findByUsername(String username);
User findById(Long id);

// get - получение сущностей
User getById(Long id);
List<User> getByActiveTrue();
Optional<User> getByEmail(String email);

// read - чтение сущностей
List<User> readByEmail(String email);
Optional<User> readByUsername(String username);
User readById(Long id);

// query - запрос сущностей
List<User> queryByEmail(String email);
Optional<User> queryByUsername(String username);
User queryById(Long id);

// search - поиск сущностей
List<User> searchByEmail(String email);
Optional<User> searchByUsername(String username);
User searchById(Long id);
```

### Подсчет и проверка существования
```java
// count - подсчет количества
long count();
long countByEmail(String email);
long countByActiveTrue();
long countByAgeGreaterThan(int age);

// exists - проверка существования
boolean existsByEmail(String email);
boolean existsByUsername(String username);
boolean existsByActiveTrue();
boolean existsByAgeGreaterThan(int age);
```

### Удаление данных
```java
// delete - удаление сущностей
void deleteByEmail(String email);
void deleteByUsername(String username);
void deleteByActiveFalse();
void deleteByAgeGreaterThan(int age);

// remove - удаление сущностей
void removeByEmail(String email);
void removeByUsername(String username);
void removeByActiveFalse();
void removeByAgeGreaterThan(int age);
```

## Ключевые слова для операторов сравнения

### Операторы равенства
```java
// Equals (по умолчанию)
List<User> findByEmail(String email);
// Генерируется: WHERE e.email = :email

// NotEquals
List<User> findByEmailNot(String email);
// Генерируется: WHERE e.email != :email

// EqualsIgnoreCase
List<User> findByEmailIgnoreCase(String email);
// Генерируется: WHERE LOWER(e.email) = LOWER(:email)

// NotEqualsIgnoreCase
List<User> findByEmailNotIgnoreCase(String email);
// Генерируется: WHERE LOWER(e.email) != LOWER(:email)
```

### Операторы для строк
```java
// Like
List<User> findByEmailLike(String emailPattern);
// Генерируется: WHERE e.email LIKE :emailPattern

// NotLike
List<User> findByEmailNotLike(String emailPattern);
// Генерируется: WHERE e.email NOT LIKE :emailPattern

// Containing
List<User> findByEmailContaining(String emailPart);
// Генерируется: WHERE e.email LIKE %:emailPart%

// NotContaining
List<User> findByEmailNotContaining(String emailPart);
// Генерируется: WHERE e.email NOT LIKE %:emailPart%

// StartingWith
List<User> findByEmailStartingWith(String emailPrefix);
// Генерируется: WHERE e.email LIKE :emailPrefix%

// EndingWith
List<User> findByEmailEndingWith(String emailSuffix);
// Генерируется: WHERE e.email LIKE %:emailSuffix

// LikeIgnoreCase
List<User> findByEmailLikeIgnoreCase(String emailPattern);
// Генерируется: WHERE LOWER(e.email) LIKE LOWER(:emailPattern)

// NotLikeIgnoreCase
List<User> findByEmailNotLikeIgnoreCase(String emailPattern);
// Генерируется: WHERE LOWER(e.email) NOT LIKE LOWER(:emailPattern)

// ContainingIgnoreCase
List<User> findByEmailContainingIgnoreCase(String emailPart);
// Генерируется: WHERE LOWER(e.email) LIKE LOWER(%:emailPart%)

// NotContainingIgnoreCase
List<User> findByEmailNotContainingIgnoreCase(String emailPart);
// Генерируется: WHERE LOWER(e.email) NOT LIKE LOWER(%:emailPart%)

// StartingWithIgnoreCase
List<User> findByEmailStartingWithIgnoreCase(String emailPrefix);
// Генерируется: WHERE LOWER(e.email) LIKE LOWER(:emailPrefix%)

// EndingWithIgnoreCase
List<User> findByEmailEndingWithIgnoreCase(String emailSuffix);
// Генерируется: WHERE LOWER(e.email) LIKE LOWER(%:emailSuffix)
```

### Операторы для чисел
```java
// GreaterThan
List<User> findByAgeGreaterThan(int age);
// Генерируется: WHERE e.age > :age

// LessThan
List<User> findByAgeLessThan(int age);
// Генерируется: WHERE e.age < :age

// GreaterThanEqual
List<User> findByAgeGreaterThanEqual(int age);
// Генерируется: WHERE e.age >= :age

// LessThanEqual
List<User> findByAgeLessThanEqual(int age);
// Генерируется: WHERE e.age <= :age

// Between
List<User> findByAgeBetween(int minAge, int maxAge);
// Генерируется: WHERE e.age BETWEEN :minAge AND :maxAge
```

### Операторы для дат
```java
// GreaterThan
List<User> findByCreatedAtGreaterThan(LocalDateTime date);
// Генерируется: WHERE e.createdAt > :date

// LessThan
List<User> findByCreatedAtLessThan(LocalDateTime date);
// Генерируется: WHERE e.createdAt < :date

// GreaterThanEqual
List<User> findByCreatedAtGreaterThanEqual(LocalDateTime date);
// Генерируется: WHERE e.createdAt >= :date

// LessThanEqual
List<User> findByCreatedAtLessThanEqual(LocalDateTime date);
// Генерируется: WHERE e.createdAt <= :date

// Between
List<User> findByCreatedAtBetween(LocalDateTime startDate, LocalDateTime endDate);
// Генерируется: WHERE e.createdAt BETWEEN :startDate AND :endDate
```

### Операторы для коллекций
```java
// In
List<User> findByRoleIn(List<Role> roles);
// Генерируется: WHERE e.role IN :roles

// NotIn
List<User> findByRoleNotIn(List<Role> roles);
// Генерируется: WHERE e.role NOT IN :roles

// InIgnoreCase
List<User> findByRoleInIgnoreCase(List<Role> roles);
// Генерируется: WHERE LOWER(e.role) IN LOWER(:roles)

// NotInIgnoreCase
List<User> findByRoleNotInIgnoreCase(List<Role> roles);
// Генерируется: WHERE LOWER(e.role) NOT IN LOWER(:roles)
```

### Операторы для null значений
```java
// IsNull
List<User> findByEmailIsNull();
// Генерируется: WHERE e.email IS NULL

// IsNotNull
List<User> findByEmailIsNotNull();
// Генерируется: WHERE e.email IS NOT NULL
```

### Операторы для булевых значений
```java
// True
List<User> findByActiveTrue();
// Генерируется: WHERE e.active = true

// False
List<User> findByActiveFalse();
// Генерируется: WHERE e.active = false
```

## Ключевые слова для логических операторов

### Оператор AND
```java
// Простой AND
List<User> findByEmailAndUsername(String email, String username);
// Генерируется: WHERE e.email = :email AND e.username = :username

// Множественный AND
List<User> findByEmailAndUsernameAndActiveTrue(String email, String username);
// Генерируется: WHERE e.email = :email AND e.username = :username AND e.active = true

// AND с различными операторами
List<User> findByEmailAndAgeGreaterThan(String email, int age);
// Генерируется: WHERE e.email = :email AND e.age > :age

// AND с вложенными свойствами
List<User> findByEmailAndAddressCity(String email, String city);
// Генерируется: WHERE e.email = :email AND e.address.city = :city
```

### Оператор OR
```java
// Простой OR
List<User> findByEmailOrUsername(String email, String username);
// Генерируется: WHERE e.email = :email OR e.username = :username

// OR с различными операторами
List<User> findByEmailOrAgeGreaterThan(String email, int age);
// Генерируется: WHERE e.email = :email OR e.age > :age

// OR с вложенными свойствами
List<User> findByEmailOrAddressCity(String email, String city);
// Генерируется: WHERE e.email = :email OR e.address.city = :city
```

## Ключевые слова для сортировки

### Сортировка по одному полю
```java
// Ascending
List<User> findAllByOrderByNameAsc();
// Генерируется: ORDER BY e.name ASC

// Descending
List<User> findAllByOrderByNameDesc();
// Генерируется: ORDER BY e.name DESC

// Сортировка с условием
List<User> findByActiveTrueOrderByEmailAsc();
// Генерируется: WHERE e.active = true ORDER BY e.email ASC
```

### Сортировка по нескольким полям
```java
// Множественная сортировка
List<User> findAllByOrderByNameAscLastNameAsc();
// Генерируется: ORDER BY e.name ASC, e.lastName ASC

// Смешанная сортировка
List<User> findAllByOrderByNameAscAgeDesc();
// Генерируется: ORDER BY e.name ASC, e.age DESC

// Сортировка с условием
List<User> findByActiveTrueOrderByCreatedAtDesc();
// Генерируется: WHERE e.active = true ORDER BY e.createdAt DESC
```

## Ключевые слова для ограничения результатов

### Ограничение количества
```java
// Top
List<User> findTop10ByOrderByCreatedAtDesc();
// Генерируется: ORDER BY e.createdAt DESC LIMIT 10

// First
User findFirstByEmail(String email);
// Генерируется: WHERE e.email = :email LIMIT 1

// Top с условием
List<User> findTop5ByActiveTrueOrderByCreatedAtDesc();
// Генерируется: WHERE e.active = true ORDER BY e.createdAt DESC LIMIT 5

// First с условием
User findFirstByActiveTrue();
// Генерируется: WHERE e.active = true LIMIT 1
```

## Ключевые слова для вложенных свойств

### Доступ к вложенным свойствам
```java
// Простое вложенное свойство
List<User> findByAddressCity(String city);
// Генерируется: WHERE e.address.city = :city

// Вложенное свойство с операторами
List<User> findByAddressCityContaining(String city);
// Генерируется: WHERE e.address.city LIKE %:city%

// Вложенное свойство с сравнениями
List<User> findByAddressZipCodeGreaterThan(String zipCode);
// Генерируется: WHERE e.address.zipCode > :zipCode

// Множественные вложенные свойства
List<User> findByAddressCityAndAddressStreet(String city, String street);
// Генерируется: WHERE e.address.city = :city AND e.address.street = :street
```

### Доступ к коллекциям вложенных объектов
```java
// Коллекция с простым условием
List<User> findByOrdersStatus(OrderStatus status);
// Генерируется: JOIN e.orders o WHERE o.status = :status

// Коллекция с операторами сравнения
List<User> findByOrdersTotalGreaterThan(BigDecimal amount);
// Генерируется: JOIN e.orders o WHERE o.total > :amount

// Коллекция с датами
List<User> findByOrdersCreatedAtBetween(LocalDateTime startDate, LocalDateTime endDate);
// Генерируется: JOIN e.orders o WHERE o.createdAt BETWEEN :startDate AND :endDate

// Коллекция с множественными условиями
List<User> findByOrdersStatusAndOrdersTotalGreaterThan(OrderStatus status, BigDecimal amount);
// Генерируется: JOIN e.orders o WHERE o.status = :status AND o.total > :amount
```

## Ключевые слова для специальных случаев

### Игнорирование регистра
```java
// IgnoreCase для строк
List<User> findByEmailIgnoreCase(String email);
List<User> findByEmailLikeIgnoreCase(String emailPattern);
List<User> findByEmailContainingIgnoreCase(String emailPart);
List<User> findByEmailStartingWithIgnoreCase(String emailPrefix);
List<User> findByEmailEndingWithIgnoreCase(String emailSuffix);

// IgnoreCase для коллекций
List<User> findByRoleInIgnoreCase(List<Role> roles);
List<User> findByRoleNotInIgnoreCase(List<Role> roles);
```

### Специальные операторы для дат
```java
// Сегодня
List<User> findByCreatedAtToday();
// Генерируется: WHERE DATE(e.createdAt) = CURRENT_DATE

// Вчера
List<User> findByCreatedAtYesterday();
// Генерируется: WHERE DATE(e.createdAt) = CURRENT_DATE - 1

// На этой неделе
List<User> findByCreatedAtThisWeek();
// Генерируется: WHERE YEARWEEK(e.createdAt) = YEARWEEK(CURRENT_DATE)

// В этом месяце
List<User> findByCreatedAtThisMonth();
// Генерируется: WHERE YEAR(e.createdAt) = YEAR(CURRENT_DATE) AND MONTH(e.createdAt) = MONTH(CURRENT_DATE)
```

## Ключевые слова для агрегатных функций

### Подсчет
```java
// Общий подсчет
long count();
// Генерируется: SELECT COUNT(e) FROM User e

// Подсчет с условием
long countByEmail(String email);
// Генерируется: SELECT COUNT(e) FROM User e WHERE e.email = :email

// Подсчет с операторами
long countByAgeGreaterThan(int age);
// Генерируется: SELECT COUNT(e) FROM User e WHERE e.age > :age

// Подсчет с вложенными свойствами
long countByAddressCity(String city);
// Генерируется: SELECT COUNT(e) FROM User e WHERE e.address.city = :city
```

### Проверка существования
```java
// Простая проверка
boolean existsByEmail(String email);
// Генерируется: SELECT COUNT(e) > 0 FROM User e WHERE e.email = :email

// Проверка с операторами
boolean existsByAgeGreaterThan(int age);
// Генерируется: SELECT COUNT(e) > 0 FROM User e WHERE e.age > :age

// Проверка с вложенными свойствами
boolean existsByAddressCity(String city);
// Генерируется: SELECT COUNT(e) > 0 FROM User e WHERE e.address.city = :city
```

## Ключевые слова для удаления

### Удаление с условиями
```java
// Простое удаление
void deleteByEmail(String email);
// Генерируется: DELETE FROM User e WHERE e.email = :email

// Удаление с операторами
void deleteByAgeGreaterThan(int age);
// Генерируется: DELETE FROM User e WHERE e.age > :age

// Удаление с вложенными свойствами
void deleteByAddressCity(String city);
// Генерируется: DELETE FROM User e WHERE e.address.city = :city

// Удаление с множественными условиями
void deleteByEmailAndActiveFalse(String email);
// Генерируется: DELETE FROM User e WHERE e.email = :email AND e.active = false
```

## Комбинированные ключевые слова

### Сложные запросы
```java
// Комбинация префиксов и операторов
List<User> findTop10ByActiveTrueOrderByCreatedAtDesc();
// Генерируется: WHERE e.active = true ORDER BY e.createdAt DESC LIMIT 10

// Комбинация операторов сравнения
List<User> findByAgeBetweenAndActiveTrue(int minAge, int maxAge);
// Генерируется: WHERE e.age BETWEEN :minAge AND :maxAge AND e.active = true

// Комбинация строковых операторов
List<User> findByEmailContainingAndUsernameStartingWith(String emailPart, String usernamePrefix);
// Генерируется: WHERE e.email LIKE %:emailPart% AND e.username LIKE :usernamePrefix%

// Комбинация с вложенными свойствами
List<User> findByAddressCityAndAgeGreaterThan(String city, int age);
// Генерируется: WHERE e.address.city = :city AND e.age > :age

// Комбинация с коллекциями
List<User> findByOrdersStatusAndEmailContaining(OrderStatus status, String emailPart);
// Генерируется: JOIN e.orders o WHERE o.status = :status AND e.email LIKE %:emailPart%
```

## Специальные случаи

### Обработка пустых коллекций
```java
// Проверка на пустую коллекцию
List<User> findByOrdersIsEmpty();
// Генерируется: WHERE e.orders IS EMPTY

// Проверка на непустую коллекцию
List<User> findByOrdersIsNotEmpty();
// Генерируется: WHERE e.orders IS NOT EMPTY
```

### Обработка размеров коллекций
```java
// Поиск по размеру коллекции
List<User> findByOrdersSize(int size);
// Генерируется: WHERE SIZE(e.orders) = :size

// Поиск по размеру коллекции с операторами
List<User> findByOrdersSizeGreaterThan(int size);
// Генерируется: WHERE SIZE(e.orders) > :size
```

## Лучшие практики использования ключевых слов

### 1. Правильное именование
```java
// Правильно - четкие и понятные имена
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmailContaining(String emailPart);
    Optional<User> findByUsernameIgnoreCase(String username);
    List<User> findByAgeBetweenAndActiveTrue(int minAge, int maxAge);
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
// Используйте ограничения для больших результатов
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findTop100ByActiveTrueOrderByCreatedAtDesc();
    List<User> findTop50ByRoleOrderByNameAsc(Role role);
    User findFirstByEmail(String email);
}

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
```

### 3. Валидация параметров
```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> findByEmailContaining(String emailPart) {
        if (emailPart == null || emailPart.trim().isEmpty()) {
            throw new IllegalArgumentException("Email part cannot be null or empty");
        }
        return userRepository.findByEmailContaining(emailPart);
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

## Заключение

PartTreeJpaQuery поддерживает богатый набор ключевых слов для создания различных типов запросов. Правильное использование этих ключевых слов позволяет создавать эффективные и читаемые запросы без написания SQL или JPQL вручную.

Ключевые принципы:
- Используйте четкие и понятные имена методов
- Следуйте конвенциям именования Spring Data JPA
- Оптимизируйте производительность через индексы и ограничения
- Добавляйте валидацию параметров в сервисном слое
- Правильно обрабатывайте результаты и исключения
