# DAO & Repository. Практика

## Создание UserRepository и UserService

### Модель User

```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "email", unique = true, nullable = false)
    private String email;
    
    @Column(name = "first_name")
    private String firstName;
    
    @Column(name = "last_name")
    private String lastName;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "role")
    private Role role;
    
    @Column(name = "active")
    private boolean active = true;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
    
    // Конструкторы
    public User() {}
    
    public User(String email, String firstName, String lastName, Role role) {
        this.email = email;
        this.firstName = firstName;
        this.lastName = lastName;
        this.role = role;
    }
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    
    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
    
    public Role getRole() { return role; }
    public void setRole(Role role) { this.role = role; }
    
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
    
    public List<Order> getOrders() { return orders; }
    public void setOrders(List<Order> orders) { this.orders = orders; }
    
    // Методы
    public void addOrder(Order order) {
        orders.add(order);
        order.setUser(this);
    }
    
    public void removeOrder(Order order) {
        orders.remove(order);
        order.setUser(null);
    }
    
    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", email='" + email + '\'' +
                ", firstName='" + firstName + '\'' +
                ", lastName='" + lastName + '\'' +
                ", role=" + role +
                ", active=" + active +
                '}';
    }
}
```

### Enum Role

```java
public enum Role {
    USER,
    ADMIN,
    MODERATOR
}
```

### Интерфейс UserRepository

```java
public interface UserRepository {
    User save(User user);
    User findById(Long id);
    List<User> findAll();
    void delete(Long id);
    void delete(User user);
    boolean exists(Long id);
    long count();
    
    // Специализированные методы
    User findByEmail(String email);
    List<User> findByRole(Role role);
    List<User> findActiveUsers();
    List<User> findByFirstNameAndLastName(String firstName, String lastName);
    List<User> findByEmailContaining(String emailPart);
    List<User> findByRoleAndActive(Role role, boolean active);
}
```

### Реализация UserRepository

```java
@Repository
@Transactional
public class UserRepositoryImpl implements UserRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public User save(User user) {
        if (user.getId() == null) {
            entityManager.persist(user);
            return user;
        } else {
            return entityManager.merge(user);
        }
    }
    
    @Override
    public User findById(Long id) {
        return entityManager.find(User.class, id);
    }
    
    @Override
    public List<User> findAll() {
        return entityManager.createQuery("from User", User.class)
                          .getResultList();
    }
    
    @Override
    public void delete(Long id) {
        User user = findById(id);
        if (user != null) {
            entityManager.remove(user);
        }
    }
    
    @Override
    public void delete(User user) {
        entityManager.remove(user);
    }
    
    @Override
    public boolean exists(Long id) {
        return findById(id) != null;
    }
    
    @Override
    public long count() {
        return (Long) entityManager.createQuery("select count(*) from User")
                                 .getSingleResult();
    }
    
    @Override
    public User findByEmail(String email) {
        return entityManager.createQuery(
            "from User where email = :email", User.class)
            .setParameter("email", email)
            .getResultStream()
            .findFirst()
            .orElse(null);
    }
    
    @Override
    public List<User> findByRole(Role role) {
        return entityManager.createQuery(
            "from User where role = :role", User.class)
            .setParameter("role", role)
            .getResultList();
    }
    
    @Override
    public List<User> findActiveUsers() {
        return entityManager.createQuery(
            "from User where active = true", User.class)
            .getResultList();
    }
    
    @Override
    public List<User> findByFirstNameAndLastName(String firstName, String lastName) {
        return entityManager.createQuery(
            "from User where firstName = :firstName and lastName = :lastName", User.class)
            .setParameter("firstName", firstName)
            .setParameter("lastName", lastName)
            .getResultList();
    }
    
    @Override
    public List<User> findByEmailContaining(String emailPart) {
        return entityManager.createQuery(
            "from User where email like :emailPart", User.class)
            .setParameter("emailPart", "%" + emailPart + "%")
            .getResultList();
    }
    
    @Override
    public List<User> findByRoleAndActive(Role role, boolean active) {
        return entityManager.createQuery(
            "from User where role = :role and active = :active", User.class)
            .setParameter("role", role)
            .setParameter("active", active)
            .getResultList();
    }
}
```

## Метод delete

### Реализация метода delete

```java
@Override
public void delete(Long id) {
    User user = findById(id);
    if (user != null) {
        entityManager.remove(user);
    }
}
```

### Улучшенная версия с возвратом результата

```java
@Override
public boolean delete(Long id) {
    User user = findById(id);
    if (user != null) {
        entityManager.remove(user);
        return true;
    }
    return false;
}
```

### Версия с проверкой существования

```java
@Override
public void delete(Long id) {
    if (!exists(id)) {
        throw new EntityNotFoundException("User with ID " + id + " not found");
    }
    
    User user = findById(id);
    entityManager.remove(user);
}
```

### Версия с каскадным удалением

```java
@Override
public void deleteWithOrders(Long id) {
    User user = findById(id);
    if (user != null) {
        // Удаляем все заказы пользователя
        for (Order order : user.getOrders()) {
            entityManager.remove(order);
        }
        entityManager.remove(user);
    }
}
```

## Метод findById

### Базовая реализация

```java
@Override
public User findById(Long id) {
    return entityManager.find(User.class, id);
}
```

### С обработкой исключений

```java
@Override
public User findById(Long id) {
    if (id == null) {
        throw new IllegalArgumentException("ID cannot be null");
    }
    
    try {
        return entityManager.find(User.class, id);
    } catch (Exception e) {
        throw new DataAccessException("Error finding user by ID: " + id, e);
    }
}
```

### С возвратом Optional

```java
@Override
public Optional<User> findById(Long id) {
    if (id == null) {
        return Optional.empty();
    }
    
    User user = entityManager.find(User.class, id);
    return Optional.ofNullable(user);
}
```

### С загрузкой ассоциаций

```java
@Override
public User findByIdWithOrders(Long id) {
    return entityManager.createQuery(
        "from User u left join fetch u.orders where u.id = :id", User.class)
        .setParameter("id", id)
        .getResultStream()
        .findFirst()
        .orElse(null);
}
```

## Тестирование метода findById

### Unit тест с Mockito

```java
@ExtendWith(MockitoExtension.class)
class UserRepositoryImplTest {
    
    @Mock
    private EntityManager entityManager;
    
    @InjectMocks
    private UserRepositoryImpl userRepository;
    
    @Test
    void findById_ShouldReturnUser_WhenUserExists() {
        // Arrange
        Long userId = 1L;
        User expectedUser = new User("test@example.com", "John", "Doe", Role.USER);
        expectedUser.setId(userId);
        
        when(entityManager.find(User.class, userId)).thenReturn(expectedUser);
        
        // Act
        User actualUser = userRepository.findById(userId);
        
        // Assert
        assertThat(actualUser).isNotNull();
        assertThat(actualUser.getId()).isEqualTo(userId);
        assertThat(actualUser.getEmail()).isEqualTo("test@example.com");
        
        verify(entityManager).find(User.class, userId);
    }
    
    @Test
    void findById_ShouldReturnNull_WhenUserDoesNotExist() {
        // Arrange
        Long userId = 999L;
        when(entityManager.find(User.class, userId)).thenReturn(null);
        
        // Act
        User actualUser = userRepository.findById(userId);
        
        // Assert
        assertThat(actualUser).isNull();
        verify(entityManager).find(User.class, userId);
    }
    
    @Test
    void findById_ShouldThrowException_WhenIdIsNull() {
        // Act & Assert
        assertThatThrownBy(() -> userRepository.findById(null))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("ID cannot be null");
    }
}
```

### Интеграционный тест

```java
@SpringBootTest
@Transactional
class UserRepositoryIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void findById_ShouldReturnUser_WhenUserExists() {
        // Arrange
        User user = new User("test@example.com", "John", "Doe", Role.USER);
        User savedUser = userRepository.save(user);
        
        // Act
        User foundUser = userRepository.findById(savedUser.getId());
        
        // Assert
        assertThat(foundUser).isNotNull();
        assertThat(foundUser.getId()).isEqualTo(savedUser.getId());
        assertThat(foundUser.getEmail()).isEqualTo("test@example.com");
    }
    
    @Test
    void findById_ShouldReturnNull_WhenUserDoesNotExist() {
        // Act
        User foundUser = userRepository.findById(999L);
        
        // Assert
        assertThat(foundUser).isNull();
    }
}
```

## Метод findById with Properties

### Реализация с Properties

```java
@Override
public User findByIdWithProperties(Long id, String... properties) {
    if (properties == null || properties.length == 0) {
        return findById(id);
    }
    
    StringBuilder hql = new StringBuilder("from User u where u.id = :id");
    
    for (String property : properties) {
        hql.append(" and u.").append(property).append(" is not null");
    }
    
    return entityManager.createQuery(hql.toString(), User.class)
                       .setParameter("id", id)
                       .getResultStream()
                       .findFirst()
                       .orElse(null);
}
```

### Использование

```java
// Найти пользователя с заполненными firstName и lastName
User user = userRepository.findByIdWithProperties(1L, "firstName", "lastName");

// Найти активного пользователя с email
User user = userRepository.findByIdWithProperties(1L, "email", "active");
```

## Метод findById with Mapper

### Интерфейс Mapper

```java
@FunctionalInterface
public interface UserMapper<T> {
    T map(User user);
}
```

### Реализация с Mapper

```java
@Override
public <T> T findByIdWithMapper(Long id, UserMapper<T> mapper) {
    User user = findById(id);
    return user != null ? mapper.map(user) : null;
}
```

### Использование Mapper

```java
// Получить только email пользователя
String email = userRepository.findByIdWithMapper(1L, User::getEmail);

// Получить полное имя пользователя
String fullName = userRepository.findByIdWithMapper(1L, 
    user -> user.getFirstName() + " " + user.getLastName());

// Получить DTO
UserDto userDto = userRepository.findByIdWithMapper(1L, 
    user -> new UserDto(user.getId(), user.getEmail(), user.getFirstName(), user.getLastName()));
```

## Метод create

### Базовая реализация

```java
@Override
public User create(User user) {
    if (user.getId() != null) {
        throw new IllegalArgumentException("New user cannot have an ID");
    }
    
    entityManager.persist(user);
    return user;
}
```

### С валидацией

```java
@Override
public User create(User user) {
    if (user == null) {
        throw new IllegalArgumentException("User cannot be null");
    }
    
    if (user.getId() != null) {
        throw new IllegalArgumentException("New user cannot have an ID");
    }
    
    if (user.getEmail() == null || user.getEmail().trim().isEmpty()) {
        throw new IllegalArgumentException("Email is required");
    }
    
    // Проверяем уникальность email
    if (findByEmail(user.getEmail()) != null) {
        throw new IllegalArgumentException("User with email " + user.getEmail() + " already exists");
    }
    
    entityManager.persist(user);
    return user;
}
```

### С установкой значений по умолчанию

```java
@Override
public User create(User user) {
    if (user.getId() != null) {
        throw new IllegalArgumentException("New user cannot have an ID");
    }
    
    // Устанавливаем значения по умолчанию
    if (user.getRole() == null) {
        user.setRole(Role.USER);
    }
    
    if (user.getActive() == null) {
        user.setActive(true);
    }
    
    entityManager.persist(user);
    return user;
}
```

## Тестирование метода create

### Unit тест

```java
@Test
void create_ShouldPersistUser_WhenValidUserProvided() {
    // Arrange
    User user = new User("test@example.com", "John", "Doe", Role.USER);
    
    // Act
    User createdUser = userRepository.create(user);
    
    // Assert
    assertThat(createdUser).isNotNull();
    assertThat(createdUser.getId()).isNotNull();
    assertThat(createdUser.getEmail()).isEqualTo("test@example.com");
    
    verify(entityManager).persist(user);
}

@Test
void create_ShouldThrowException_WhenUserHasId() {
    // Arrange
    User user = new User("test@example.com", "John", "Doe", Role.USER);
    user.setId(1L);
    
    // Act & Assert
    assertThatThrownBy(() -> userRepository.create(user))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("New user cannot have an ID");
}

@Test
void create_ShouldThrowException_WhenEmailAlreadyExists() {
    // Arrange
    User existingUser = new User("test@example.com", "John", "Doe", Role.USER);
    User newUser = new User("test@example.com", "Jane", "Doe", Role.USER);
    
    when(userRepository.findByEmail("test@example.com")).thenReturn(existingUser);
    
    // Act & Assert
    assertThatThrownBy(() -> userRepository.create(newUser))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("User with email test@example.com already exists");
}
```

### Интеграционный тест

```java
@Test
void create_ShouldPersistUser_WhenValidUserProvided() {
    // Arrange
    User user = new User("test@example.com", "John", "Doe", Role.USER);
    
    // Act
    User createdUser = userRepository.create(user);
    
    // Assert
    assertThat(createdUser).isNotNull();
    assertThat(createdUser.getId()).isNotNull();
    
    // Проверяем, что пользователь действительно сохранен
    User foundUser = userRepository.findById(createdUser.getId());
    assertThat(foundUser).isNotNull();
    assertThat(foundUser.getEmail()).isEqualTo("test@example.com");
}
```

## Custom ByteBuddy TransactionInterceptor

### Создание TransactionInterceptor

```java
public class TransactionInterceptor {
    
    private final EntityManager entityManager;
    
    public TransactionInterceptor(EntityManager entityManager) {
        this.entityManager = entityManager;
    }
    
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        EntityTransaction transaction = entityManager.getTransaction();
        
        try {
            transaction.begin();
            Object result = proxy.invokeSuper(obj, args);
            transaction.commit();
            return result;
        } catch (Exception e) {
            if (transaction.isActive()) {
                transaction.rollback();
            }
            throw e;
        }
    }
}
```

### Создание прокси с ByteBuddy

```java
@Component
public class RepositoryProxyFactory {
    
    private final EntityManager entityManager;
    
    public RepositoryProxyFactory(EntityManager entityManager) {
        this.entityManager = entityManager;
    }
    
    public <T> T createProxy(Class<T> repositoryClass, T implementation) {
        return new ByteBuddy()
            .subclass(repositoryClass)
            .method(ElementMatchers.any())
            .intercept(MethodDelegation.to(new TransactionInterceptor(entityManager)))
            .make()
            .load(getClass().getClassLoader())
            .getLoaded()
            .getDeclaredConstructor()
            .newInstance();
    }
}
```

### Использование прокси

```java
@Configuration
public class RepositoryConfig {
    
    @Bean
    public UserRepository userRepository(EntityManager entityManager, 
                                      RepositoryProxyFactory proxyFactory) {
        UserRepositoryImpl implementation = new UserRepositoryImpl(entityManager);
        return proxyFactory.createProxy(UserRepository.class, implementation);
    }
}
```

## Custom ByteBuddy Proxy

### Создание кастомного прокси

```java
public class LoggingInterceptor {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingInterceptor.class);
    
    public static Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        String methodName = method.getName();
        String className = obj.getClass().getSimpleName();
        
        log.debug("Entering method: {}.{}", className, methodName);
        
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = proxy.invokeSuper(obj, args);
            long duration = System.currentTimeMillis() - startTime;
            log.debug("Exiting method: {}.{} ({}ms)", className, methodName, duration);
            return result;
        } catch (Exception e) {
            log.error("Exception in method: {}.{}", className, methodName, e);
            throw e;
        }
    }
}
```

### Создание прокси с логированием

```java
@Component
public class LoggingProxyFactory {
    
    public <T> T createLoggingProxy(Class<T> targetClass, T implementation) {
        return new ByteBuddy()
            .subclass(targetClass)
            .method(ElementMatchers.any())
            .intercept(MethodDelegation.to(LoggingInterceptor.class))
            .make()
            .load(getClass().getClassLoader())
            .getLoaded()
            .getDeclaredConstructor()
            .newInstance();
    }
}
```

### Создание прокси с кэшированием

```java
public class CachingInterceptor {
    
    private final Map<String, Object> cache = new ConcurrentHashMap<>();
    
    public static Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        String cacheKey = generateCacheKey(method, args);
        
        // Проверяем кэш только для методов чтения
        if (isReadMethod(method)) {
            Object cachedResult = cache.get(cacheKey);
            if (cachedResult != null) {
                return cachedResult;
            }
        }
        
        Object result = proxy.invokeSuper(obj, args);
        
        // Кэшируем результат для методов чтения
        if (isReadMethod(method)) {
            cache.put(cacheKey, result);
        }
        
        return result;
    }
    
    private static String generateCacheKey(Method method, Object[] args) {
        StringBuilder key = new StringBuilder(method.getDeclaringClass().getSimpleName())
            .append(".")
            .append(method.getName());
        
        if (args != null) {
            for (Object arg : args) {
                key.append(".").append(arg);
            }
        }
        
        return key.toString();
    }
    
    private static boolean isReadMethod(Method method) {
        String methodName = method.getName();
        return methodName.startsWith("find") || 
               methodName.startsWith("get") || 
               methodName.startsWith("count") ||
               methodName.startsWith("exists");
    }
}
```

### Комбинированный прокси

```java
@Component
public class AdvancedProxyFactory {
    
    public <T> T createAdvancedProxy(Class<T> targetClass, T implementation) {
        return new ByteBuddy()
            .subclass(targetClass)
            .method(ElementMatchers.any())
            .intercept(MethodDelegation.to(LoggingInterceptor.class)
                .andThen(MethodDelegation.to(CachingInterceptor.class)))
            .make()
            .load(getClass().getClassLoader())
            .getLoaded()
            .getDeclaredConstructor()
            .newInstance();
    }
}
```

## Заключение

Практическое применение DAO/Repository паттерна включает:

- **Создание** специализированных репозиториев
- **Тестирование** всех методов
- **Использование** мапперов и прокси
- **Оптимизацию** производительности
- **Обеспечение** надежности и поддерживаемости кода

Правильная реализация всех компонентов обеспечивает создание масштабируемых и тестируемых приложений.

---

## Senior-insights: Spring Data JPA паттерны

### Почему @Repository часто не нужен с JpaRepository

`JpaRepository` уже наследует `Repository` — маркерный интерфейс Spring Data. Spring Data автоматически зарегистрирует реализацию при сканировании компонентов.

```java
// @Repository НЕ нужен — Spring Data сам найдёт и зарегистрирует
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByActiveTrue();
}

// @Repository нужен только для ручных реализаций с EntityManager,
// чтобы Spring преобразовывал JPA-исключения в DataAccessException
@Repository
public class CustomUserRepositoryImpl {
    @PersistenceContext
    private EntityManager em;
    // ...
}
```

> [!INFO]
> `@Repository` обеспечивает трансляцию JPA-исключений в Spring `DataAccessException`. В `JpaRepository` это уже встроено через `SimpleJpaRepository`.

### Specification pattern для динамических фильтров

`Specification<T>` — паттерн для type-safe динамических WHERE-условий. Базируется на Criteria API.

```java
// Интерфейс репозитория
public interface UserRepository extends JpaRepository<User, Long>,
                                        JpaSpecificationExecutor<User> {
}

// Класс с Specification-ами (статические фабричные методы)
public class UserSpecifications {

    public static Specification<User> hasEmail(String email) {
        return (root, query, cb) ->
            email == null ? null : cb.equal(root.get(User_.email), email);
    }

    public static Specification<User> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get(User_.active));
    }

    public static Specification<User> hasRole(Role role) {
        return (root, query, cb) ->
            role == null ? null : cb.equal(root.get(User_.role), role);
    }

    public static Specification<User> registeredAfter(LocalDate date) {
        return (root, query, cb) ->
            date == null ? null : cb.greaterThanOrEqualTo(root.get(User_.createdAt), date);
    }
}

// Использование: комбинируем условия динамически
@Service
public class UserService {

    public List<User> searchUsers(UserSearchRequest request) {
        Specification<User> spec = Specification.where(UserSpecifications.isActive())
            .and(UserSpecifications.hasEmail(request.getEmail()))
            .and(UserSpecifications.hasRole(request.getRole()))
            .and(UserSpecifications.registeredAfter(request.getRegisteredAfter()));

        return userRepository.findAll(spec);
    }
}
```

### Кастомная реализация через фрагменты (fragment interfaces)

Spring Data позволяет добавить кастомные методы через паттерн "фрагмент интерфейса":

```java
// 1. Интерфейс фрагмента
public interface UserRepositoryCustom {
    List<User> findUsersWithComplexCriteria(ComplexFilter filter);
    void bulkDeactivate(List<Long> ids);
}

// 2. Реализация фрагмента (имя: {репозиторий}Impl)
@Repository
public class UserRepositoryCustomImpl implements UserRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<User> findUsersWithComplexCriteria(ComplexFilter filter) {
        // Сложный Criteria API запрос
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<User> cq = cb.createQuery(User.class);
        Root<User> root = cq.from(User.class);

        List<Predicate> predicates = new ArrayList<>();
        if (filter.getMinAge() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get(User_.age), filter.getMinAge()));
        }
        if (filter.getDepartment() != null) {
            Join<User, Department> dept = root.join(User_.department);
            predicates.add(cb.equal(dept.get(Department_.name), filter.getDepartment()));
        }

        cq.where(predicates.toArray(new Predicate[0]));
        return em.createQuery(cq).getResultList();
    }

    @Override
    @Transactional
    public void bulkDeactivate(List<Long> ids) {
        em.createQuery("UPDATE User u SET u.active = false WHERE u.id IN :ids")
          .setParameter("ids", ids)
          .executeUpdate();
    }
}

// 3. Финальный репозиторий наследует оба интерфейса
public interface UserRepository extends JpaRepository<User, Long>,
                                        JpaSpecificationExecutor<User>,
                                        UserRepositoryCustom {
    // Стандартные методы Spring Data + кастомные из UserRepositoryCustom
}
```

### Проблема N+1 в Spring Data: @EntityGraph на методе репозитория

Spring Data по умолчанию использует `FetchType` из маппинга. Для LAZY-ассоциаций при запросе списка — классический N+1.

```java
@Entity
public class User {
    @ManyToOne(fetch = FetchType.LAZY) // LAZY по умолчанию
    private Department department;
}

// ПРОБЛЕМА: N+1
List<User> users = userRepository.findAll();
for (User u : users) {
    u.getDepartment().getName(); // N дополнительных SELECT!
}
```

**Решение 1: @EntityGraph на методе репозитория**

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Загружаем department вместе с user (JOIN FETCH)
    @EntityGraph(attributePaths = {"department"})
    List<User> findAll();

    // Для конкретного метода
    @EntityGraph(attributePaths = {"department", "orders"})
    @Query("SELECT u FROM User u WHERE u.active = true")
    List<User> findActiveUsersWithDetails();

    // Использовать именованный EntityGraph
    @EntityGraph(value = "User.withDepartmentAndOrders")
    Optional<User> findById(Long id);
}

// Именованный EntityGraph на entity:
@Entity
@NamedEntityGraph(
    name = "User.withDepartmentAndOrders",
    attributeNodes = {
        @NamedAttributeNode("department"),
        @NamedAttributeNode(value = "orders", subgraph = "order-items")
    },
    subgraphs = @NamedSubgraph(
        name = "order-items",
        attributeNodes = @NamedAttributeNode("items")
    )
)
public class User { ... }
```

> [!WARNING]
> `@EntityGraph` с несколькими коллекциями (`orders` + `items`) может вызвать **MultipleBagFetchException** или декартово произведение. Безопаснее использовать `Set` вместо `List` для нескольких eager-коллекций, или разбить на несколько запросов.

**Решение 2: @Query с JOIN FETCH**

```java
@Query("SELECT DISTINCT u FROM User u LEFT JOIN FETCH u.department LEFT JOIN FETCH u.orders")
List<User> findAllWithAssociations();
// DISTINCT нужен чтобы убрать дублирование из-за JOIN
```

---

**Связанные файлы:**
- [[DAO & Repository. CRUD]] — базовые операции
- [[Entity Graphs]] — подробнее об EntityGraph
- [[Общая информация и базовые варианты решения]] — N+1 проблема 