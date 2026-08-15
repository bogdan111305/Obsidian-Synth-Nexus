# DAO & Repository. JSR 303. Bean Validation

## JSR 303 Bean Validation Specification

### Обзор JSR 303

JSR 303 (Bean Validation) - это спецификация Java для валидации объектов. Она предоставляет стандартный способ валидации JavaBeans и их свойств через аннотации.

### Основные возможности

- **Декларативная валидация** через аннотации
- **Кастомные валидаторы** для специфических требований
- **Группы валидации** для разных сценариев
- **Интеграция** с JPA/Hibernate
- **Каскадная валидация** для вложенных объектов

### Жизненный цикл валидации

1. **Создание** объекта
2. **Установка** значений свойств
3. **Валидация** через аннотации
4. **Обработка** результатов валидации

## Подключение hibernate-validator зависимости

### Maven

```xml
<dependencies>
    <!-- Hibernate Validator -->
    <dependency>
        <groupId>org.hibernate.validator</groupId>
        <artifactId>hibernate-validator</artifactId>
        <version>6.2.5.Final</version>
    </dependency>
    
    <!-- Expression Language для сообщений об ошибках -->
    <dependency>
        <groupId>org.glassfish</groupId>
        <artifactId>jakarta.el</artifactId>
        <version>3.0.4</version>
    </dependency>
</dependencies>
```

### Gradle

```gradle
dependencies {
    implementation 'org.hibernate.validator:hibernate-validator:6.2.5.Final'
    implementation 'org.glassfish:jakarta.el:3.0.4'
}
```

### Spring Boot

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

## Аннотации JSR 303 Specification

### Базовые аннотации валидации

#### @NotNull
```java
@Entity
public class User {
    
    @NotNull(message = "Email cannot be null")
    @Column(name = "email", unique = true)
    private String email;
    
    @NotNull(message = "First name is required")
    @Column(name = "first_name")
    private String firstName;
}
```

#### @Size
```java
@Entity
public class User {
    
    @Size(min = 2, max = 50, message = "First name must be between 2 and 50 characters")
    @Column(name = "first_name")
    private String firstName;
    
    @Size(max = 255, message = "Email cannot exceed 255 characters")
    @Column(name = "email")
    private String email;
}
```

#### @Min, @Max
```java
@Entity
public class Product {
    
    @Min(value = 0, message = "Price cannot be negative")
    @Column(name = "price")
    private BigDecimal price;
    
    @Max(value = 1000, message = "Quantity cannot exceed 1000")
    @Column(name = "quantity")
    private Integer quantity;
}
```

#### @Email
```java
@Entity
public class User {
    
    @Email(message = "Invalid email format")
    @Column(name = "email", unique = true)
    private String email;
}
```

#### @Pattern
```java
@Entity
public class User {
    
    @Pattern(regexp = "^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,6}$", 
             message = "Invalid email format")
    @Column(name = "email")
    private String email;
    
    @Pattern(regexp = "^\\+?[1-9]\\d{1,14}$", 
             message = "Invalid phone number format")
    @Column(name = "phone")
    private String phone;
}
```

#### @Past, @Future
```java
@Entity
public class User {
    
    @Past(message = "Birth date must be in the past")
    @Column(name = "birth_date")
    private LocalDate birthDate;
    
    @Future(message = "Expiration date must be in the future")
    @Column(name = "expiration_date")
    private LocalDateTime expirationDate;
}
```

### Сложные аннотации

#### @Valid
```java
@Entity
public class Order {
    
    @Valid
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();
    
    @Valid
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "shipping_address_id")
    private Address shippingAddress;
}
```

#### @AssertTrue, @AssertFalse
```java
@Entity
public class User {
    
    @AssertTrue(message = "Terms must be accepted")
    @Column(name = "terms_accepted")
    private boolean termsAccepted;
    
    @AssertFalse(message = "User cannot be blocked")
    @Column(name = "blocked")
    private boolean blocked;
}
```

#### @DecimalMin, @DecimalMax
```java
@Entity
public class Product {
    
    @DecimalMin(value = "0.01", message = "Price must be at least 0.01")
    @DecimalMax(value = "999999.99", message = "Price cannot exceed 999999.99")
    @Column(name = "price")
    private BigDecimal price;
}
```

## Аннотация @Valid для проверки вложенных объектов

### Каскадная валидация

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotNull(message = "Email is required")
    @Email(message = "Invalid email format")
    @Column(name = "email", unique = true)
    private String email;
    
    @Valid
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
    
    @Valid
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "profile_id")
    private UserProfile profile;
    
    @Valid
    @Embedded
    private Address address;
}
```

### Вложенные объекты

```java
@Embeddable
public class Address {
    
    @NotNull(message = "Street is required")
    @Size(min = 5, max = 100, message = "Street must be between 5 and 100 characters")
    @Column(name = "street")
    private String street;
    
    @NotNull(message = "City is required")
    @Size(min = 2, max = 50, message = "City must be between 2 and 50 characters")
    @Column(name = "city")
    private String city;
    
    @NotNull(message = "Postal code is required")
    @Pattern(regexp = "^\\d{5}(-\\d{4})?$", message = "Invalid postal code format")
    @Column(name = "postal_code")
    private String postalCode;
    
    @NotNull(message = "Country is required")
    @Size(min = 2, max = 50, message = "Country must be between 2 and 50 characters")
    @Column(name = "country")
    private String country;
    
    // Геттеры и сеттеры
}
```

### Коллекции с валидацией

```java
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotNull(message = "Order date is required")
    @PastOrPresent(message = "Order date cannot be in the future")
    @Column(name = "order_date")
    private LocalDateTime orderDate;
    
    @Valid
    @NotEmpty(message = "Order must contain at least one item")
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();
    
    @Valid
    @NotNull(message = "Shipping address is required")
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "shipping_address_id")
    private Address shippingAddress;
}
```

## Создание объекта Validator

### Создание ValidatorFactory

```java
@Component
public class ValidationService {
    
    private final Validator validator;
    
    public ValidationService() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        this.validator = factory.getValidator();
    }
    
    public <T> Set<ConstraintViolation<T>> validate(T object) {
        return validator.validate(object);
    }
    
    public <T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups) {
        return validator.validate(object, groups);
    }
    
    public <T> boolean isValid(T object) {
        return validator.validate(object).isEmpty();
    }
    
    public <T> void validateAndThrow(T object) {
        Set<ConstraintViolation<T>> violations = validator.validate(object);
        if (!violations.isEmpty()) {
            throw new ValidationException("Validation failed", violations);
        }
    }
}
```

### Кастомный Validator

```java
@Component
public class CustomValidator {
    
    private final Validator validator;
    
    public CustomValidator() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        this.validator = factory.getValidator();
    }
    
    public ValidationResult validate(Object object) {
        Set<ConstraintViolation<Object>> violations = validator.validate(object);
        
        if (violations.isEmpty()) {
            return ValidationResult.success();
        } else {
            List<String> errors = violations.stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.toList());
            return ValidationResult.failure(errors);
        }
    }
    
    public ValidationResult validateWithGroups(Object object, Class<?>... groups) {
        Set<ConstraintViolation<Object>> violations = validator.validate(object, groups);
        
        if (violations.isEmpty()) {
            return ValidationResult.success();
        } else {
            List<String> errors = violations.stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.toList());
            return ValidationResult.failure(errors);
        }
    }
}
```

### Класс ValidationResult

```java
public class ValidationResult {
    
    private final boolean valid;
    private final List<String> errors;
    
    private ValidationResult(boolean valid, List<String> errors) {
        this.valid = valid;
        this.errors = errors;
    }
    
    public static ValidationResult success() {
        return new ValidationResult(true, Collections.emptyList());
    }
    
    public static ValidationResult failure(List<String> errors) {
        return new ValidationResult(false, errors);
    }
    
    public boolean isValid() {
        return valid;
    }
    
    public List<String> getErrors() {
        return errors;
    }
    
    public void throwIfInvalid() {
        if (!valid) {
            throw new ValidationException("Validation failed: " + String.join(", ", errors));
        }
    }
}
```

## Validation groups

### Определение групп валидации

```java
public interface CreateGroup {}
public interface UpdateGroup {}
public interface AdminGroup {}
public interface UserGroup {}
```

### Использование групп в сущностях

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotNull(groups = {CreateGroup.class, UpdateGroup.class}, 
             message = "Email is required")
    @Email(groups = {CreateGroup.class, UpdateGroup.class}, 
           message = "Invalid email format")
    @Column(name = "email", unique = true)
    private String email;
    
    @NotNull(groups = CreateGroup.class, message = "Password is required")
    @Size(groups = CreateGroup.class, min = 8, max = 100, 
          message = "Password must be between 8 and 100 characters")
    @Column(name = "password")
    private String password;
    
    @NotNull(groups = {CreateGroup.class, UpdateGroup.class}, 
             message = "First name is required")
    @Size(groups = {CreateGroup.class, UpdateGroup.class}, min = 2, max = 50, 
          message = "First name must be between 2 and 50 characters")
    @Column(name = "first_name")
    private String firstName;
    
    @NotNull(groups = AdminGroup.class, message = "Role is required for admin operations")
    @Enumerated(EnumType.STRING)
    @Column(name = "role")
    private Role role;
    
    @AssertTrue(groups = CreateGroup.class, message = "Terms must be accepted")
    @Column(name = "terms_accepted")
    private boolean termsAccepted;
}
```

### Валидация с группами в Repository

```java
@Repository
@Transactional
public class UserRepositoryImpl implements UserRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    private final Validator validator;
    
    public UserRepositoryImpl() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        this.validator = factory.getValidator();
    }
    
    @Override
    public User create(User user) {
        // Валидация для создания
        Set<ConstraintViolation<User>> violations = validator.validate(user, CreateGroup.class);
        if (!violations.isEmpty()) {
            throw new ValidationException("Validation failed for user creation", violations);
        }
        
        entityManager.persist(user);
        return user;
    }
    
    @Override
    public User update(User user) {
        // Валидация для обновления
        Set<ConstraintViolation<User>> violations = validator.validate(user, UpdateGroup.class);
        if (!violations.isEmpty()) {
            throw new ValidationException("Validation failed for user update", violations);
        }
        
        return entityManager.merge(user);
    }
    
    @Override
    public User createAdmin(User user) {
        // Валидация для админских операций
        Set<ConstraintViolation<User>> violations = validator.validate(user, 
            CreateGroup.class, AdminGroup.class);
        if (!violations.isEmpty()) {
            throw new ValidationException("Validation failed for admin user creation", violations);
        }
        
        entityManager.persist(user);
        return user;
    }
}
```

### Валидация в Service слое

```java
@Service
@Transactional
public class UserService {
    
    private final UserRepository userRepository;
    private final Validator validator;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        this.validator = factory.getValidator();
    }
    
    public User createUser(User user) {
        // Валидация перед сохранением
        validateUser(user, CreateGroup.class);
        return userRepository.create(user);
    }
    
    public User updateUser(User user) {
        // Валидация перед обновлением
        validateUser(user, UpdateGroup.class);
        return userRepository.update(user);
    }
    
    public User createAdminUser(User user) {
        // Валидация для админских операций
        validateUser(user, CreateGroup.class, AdminGroup.class);
        return userRepository.createAdmin(user);
    }
    
    private void validateUser(User user, Class<?>... groups) {
        Set<ConstraintViolation<User>> violations = validator.validate(user, groups);
        if (!violations.isEmpty()) {
            List<String> errors = violations.stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.toList());
            throw new ValidationException("User validation failed: " + String.join(", ", errors));
        }
    }
}
```

### Кастомные группы валидации

```java
public interface UserValidation {
    interface Create {}
    interface Update {}
    interface Admin {}
    interface Profile {}
}
```

```java
@Entity
public class User {
    
    @NotNull(groups = {UserValidation.Create.class, UserValidation.Update.class})
    @Email(groups = {UserValidation.Create.class, UserValidation.Update.class})
    private String email;
    
    @NotNull(groups = UserValidation.Create.class)
    @Size(groups = UserValidation.Create.class, min = 8)
    private String password;
    
    @NotNull(groups = UserValidation.Admin.class)
    private Role role;
    
    @Valid
    @NotNull(groups = UserValidation.Profile.class)
    private UserProfile profile;
}
```

## Интеграция с Spring

### Конфигурация Spring

```java
@Configuration
@EnableTransactionManagement
public class ValidationConfig {
    
    @Bean
    public Validator validator() {
        return Validation.buildDefaultValidatorFactory().getValidator();
    }
    
    @Bean
    public LocalValidatorFactoryBean localValidatorFactoryBean() {
        LocalValidatorFactoryBean factory = new LocalValidatorFactoryBean();
        factory.setValidationMessageSource(messageSource());
        return factory;
    }
    
    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource messageSource = 
            new ReloadableResourceBundleMessageSource();
        messageSource.setBasename("classpath:messages");
        messageSource.setDefaultEncoding("UTF-8");
        return messageSource;
    }
}
```

### Использование в контроллерах

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final UserService userService;
    private final Validator validator;
    
    public UserController(UserService userService, Validator validator) {
        this.userService = userService;
        this.validator = validator;
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        User createdUser = userService.createUser(user);
        return ResponseEntity.ok(createdUser);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, 
                                         @Valid @RequestBody User user) {
        user.setId(id);
        User updatedUser = userService.updateUser(user);
        return ResponseEntity.ok(updatedUser);
    }
    
    @PostMapping("/admin")
    public ResponseEntity<User> createAdminUser(@Valid @RequestBody User user) {
        User createdUser = userService.createAdminUser(user);
        return ResponseEntity.ok(createdUser);
    }
}
```

### Обработка ошибок валидации

```java
@ControllerAdvice
public class ValidationExceptionHandler {
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(ValidationException ex) {
        ErrorResponse error = new ErrorResponse();
        error.setMessage("Validation failed");
        error.setErrors(extractValidationErrors(ex));
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleMethodArgumentNotValidException(
            MethodArgumentNotValidException ex) {
        ErrorResponse error = new ErrorResponse();
        error.setMessage("Validation failed");
        error.setErrors(extractBindingErrors(ex.getBindingResult()));
        return ResponseEntity.badRequest().body(error);
    }
    
    private List<String> extractValidationErrors(ValidationException ex) {
        if (ex.getConstraintViolations() != null) {
            return ex.getConstraintViolations().stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.toList());
        }
        return Collections.singletonList(ex.getMessage());
    }
    
    private List<String> extractBindingErrors(BindingResult bindingResult) {
        return bindingResult.getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.toList());
    }
}
```

## Лучшие практики

### 1. Использование сообщений об ошибках

```java
@Entity
public class User {
    
    @NotNull(message = "{user.email.required}")
    @Email(message = "{user.email.invalid}")
    private String email;
    
    @Size(min = 8, max = 100, message = "{user.password.length}")
    private String password;
}
```

### 2. Кастомные валидаторы

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
public @interface StrongPassword {
    String message() default "Password must be strong";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {
    
    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null) {
            return false;
        }
        
        // Проверяем наличие цифр
        boolean hasDigit = password.matches(".*\\d.*");
        // Проверяем наличие букв в верхнем регистре
        boolean hasUpper = password.matches(".*[A-Z].*");
        // Проверяем наличие букв в нижнем регистре
        boolean hasLower = password.matches(".*[a-z].*");
        // Проверяем наличие специальных символов
        boolean hasSpecial = password.matches(".*[!@#$%^&*()_+\\-=\\[\\]{};':\"\\\\|,.<>\\/?].*");
        
        return hasDigit && hasUpper && hasLower && hasSpecial;
    }
}
```

### 3. Валидация в Repository

```java
@Repository
public class UserRepositoryImpl implements UserRepository {
    
    private final Validator validator;
    
    @Override
    public User save(User user) {
        // Валидация перед сохранением
        Set<ConstraintViolation<User>> violations = validator.validate(user);
        if (!violations.isEmpty()) {
            throw new ValidationException("User validation failed", violations);
        }
        
        if (user.getId() == null) {
            entityManager.persist(user);
            return user;
        } else {
            return entityManager.merge(user);
        }
    }
}
```

## Заключение

JSR 303 Bean Validation предоставляет мощный инструмент для валидации данных в Java-приложениях:

- **Декларативная валидация** через аннотации
- **Гибкость** через кастомные валидаторы
- **Интеграция** с JPA/Hibernate
- **Группы валидации** для разных сценариев
- **Каскадная валидация** для сложных объектов

Правильное использование валидации обеспечивает целостность данных и улучшает качество приложения. 