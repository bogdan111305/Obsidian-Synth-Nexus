# Интерфейс Repository

## Обзор

Интерфейс `Repository` является базовым интерфейсом в Spring Data JPA, который предоставляет фундаментальную функциональность для работы с сущностями. Это маркерный интерфейс, который не содержит методов, но служит основой для всей иерархии репозиториев.

## Иерархия интерфейсов Spring Data

```
Repository (базовый маркерный интерфейс)
    ↓
CrudRepository<T, ID> (добавляет CRUD операции)
    ↓
PagingAndSortingRepository<T, ID> (добавляет пагинацию и сортировку)
    ↓
JpaRepository<T, ID> (специфичный для JPA с дополнительными методами)
```

## Детальный анализ каждого интерфейса

### Repository<T, ID>
Базовый маркерный интерфейс без методов:
```java
@Indexed
public interface Repository<T, ID> {
    // Пустой интерфейс - маркер
}
```

### CrudRepository<T, ID>
Добавляет базовые CRUD операции:

#### Методы сохранения
```java
<S extends T> S save(S entity);
<S extends T> Iterable<S> saveAll(Iterable<S> entities);
```

#### Методы поиска
```java
Optional<T> findById(ID id);
Iterable<T> findAll();
Iterable<T> findAllById(Iterable<ID> ids);
boolean existsById(ID id);
long count();
```

#### Методы удаления
```java
void deleteById(ID id);
void delete(T entity);
void deleteAllById(Iterable<? extends ID> ids);
void deleteAll(Iterable<? extends T> entities);
void deleteAll();
```

### PagingAndSortingRepository<T, ID>
Добавляет поддержку пагинации и сортировки:
```java
Iterable<T> findAll(Sort sort);
Page<T> findAll(Pageable pageable);
```

### JpaRepository<T, ID>
Специфичный для JPA интерфейс с дополнительными методами:

#### Дополнительные методы поиска
```java
List<T> findAll();
List<T> findAll(Sort sort);
List<T> findAllById(Iterable<ID> ids);
```

#### Методы сохранения с возвратом результата
```java
<S extends T> List<S> saveAllAndFlush(Iterable<S> entities);
<S extends T> S saveAndFlush(S entity);
```

#### Методы удаления
```java
void deleteAllInBatch(Iterable<T> entities);
void deleteAllByIdInBatch(Iterable<ID> ids);
```

#### Методы flush
```java
void flush();
```

## Примеры использования

### Базовый репозиторий
```java
@Repository
public interface UserRepository extends Repository<User, Long> {
    // Только кастомные методы - базовые CRUD методы недоступны
    List<User> findByEmail(String email);
}
```

### CRUD репозиторий
```java
@Repository
public interface UserRepository extends CrudRepository<User, Long> {
    // Доступны все CRUD методы + кастомные
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
}
```

### JPA репозиторий
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // Доступны все методы JpaRepository + кастомные
    List<User> findByEmail(String email);
    Page<User> findByActiveTrue(Pageable pageable);
}
```

## Настройка репозиториев

### Базовая конфигурация
```java
@Configuration
@EnableJpaRepositories(basePackages = "com.example.repository")
public class JpaConfig {
    // конфигурация
}
```

### Расширенная конфигурация
```java
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.repository",
    entityManagerFactoryRef = "entityManagerFactory",
    transactionManagerRef = "transactionManager",
    repositoryImplementationPostfix = "Impl",
    repositoryBaseClass = CustomRepositoryImpl.class
)
public class JpaConfig {
    // детальная конфигурация
}
```

### Конфигурация через свойства
```yaml
spring:
  data:
    jpa:
      repositories:
        enabled: true
        bootstrap-mode: default
        repository-base-package: com.example.repository
        repository-implementation-postfix: Impl
```

## Типы ID

### Примитивные типы
```java
public interface UserRepository extends JpaRepository<User, Long> { }
public interface ProductRepository extends JpaRepository<Product, Integer> { }
public interface OrderRepository extends JpaRepository<Order, String> { }
```

### Составные ключи
```java
@Embeddable
public class OrderId implements Serializable {
    private Long customerId;
    private Long orderNumber;
    // equals, hashCode, конструкторы
}

@Entity
public class Order {
    @EmbeddedId
    private OrderId id;
    // остальные поля
}

public interface OrderRepository extends JpaRepository<Order, OrderId> { }
```

### UUID ключи
```java
@Entity
public class Document {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private UUID id;
    // остальные поля
}

public interface DocumentRepository extends JpaRepository<Document, UUID> { }
```

## Лучшие практики

### 1. Выбор правильного базового интерфейса
- **Repository** - только для кастомных методов без CRUD
- **CrudRepository** - для базовых CRUD операций
- **JpaRepository** - для полного функционала JPA

### 2. Именование
```java
// Правильно
public interface UserRepository extends JpaRepository<User, Long> { }
public interface ProductRepository extends JpaRepository<Product, Long> { }

// Неправильно
public interface UserRepo extends JpaRepository<User, Long> { }
public interface ProductDAO extends JpaRepository<Product, Long> { }
```

### 3. Пакетная структура
```
com.example
├── entity
│   ├── User.java
│   └── Product.java
├── repository
│   ├── UserRepository.java
│   └── ProductRepository.java
└── service
    ├── UserService.java
    └── ProductService.java
```

### 4. Аннотации
```java
@Repository  // Явное указание компонента
public interface UserRepository extends JpaRepository<User, Long> {
    // методы
}
```

### 5. Типизация
```java
// Всегда указывайте типы
public interface UserRepository extends JpaRepository<User, Long> { }
public interface ProductRepository extends JpaRepository<Product, Long> { }

// Избегайте сырых типов
public interface UserRepository extends JpaRepository { } // Неправильно
```

## Производительность

### Ленивая загрузка репозиториев
```yaml
spring:
  data:
    jpa:
      repositories:
        bootstrap-mode: lazy
```

### Кэширование метаданных
```java
@Configuration
public class OptimizedJpaConfig {
    
    @Bean
    public RepositoryFactorySupport repositoryFactorySupport(EntityManager entityManager) {
        JpaRepositoryFactory factory = new JpaRepositoryFactory(entityManager);
        factory.setRepositoryBaseClass(CustomRepositoryImpl.class);
        return factory;
    }
}
```

## Тестирование

### Unit тестирование
```java
@ExtendWith(MockitoExtension.class)
class UserRepositoryTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Test
    void testFindById() {
        // given
        User user = new User();
        user.setId(1L);
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));
        
        // when
        Optional<User> result = userRepository.findById(1L);
        
        // then
        assertTrue(result.isPresent());
        assertEquals(1L, result.get().getId());
    }
}
```

### Интеграционное тестирование
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testSaveAndFind() {
        // given
        User user = new User();
        user.setEmail("test@example.com");
        
        // when
        User savedUser = userRepository.save(user);
        Optional<User> foundUser = userRepository.findById(savedUser.getId());
        
        // then
        assertTrue(foundUser.isPresent());
        assertEquals("test@example.com", foundUser.get().getEmail());
    }
}
```

## Отладка и мониторинг

### Логирование SQL запросов
```properties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
logging.level.org.springframework.data.jpa=DEBUG
```

### Мониторинг производительности
```java
@Component
public class RepositoryPerformanceMonitor {
    
    @EventListener(ApplicationReadyEvent.class)
    public void monitorRepositoryPerformance() {
        // Логика мониторинга
    }
}
```

## Ограничения и особенности

### 1. Только интерфейсы
Репозитории должны быть интерфейсами, не классами:
```java
// Правильно
public interface UserRepository extends JpaRepository<User, Long> { }

// Неправильно
public class UserRepository extends JpaRepository<User, Long> { }
```

### 2. Типизация
Всегда указывайте типы сущности и ID:
```java
// Правильно
public interface UserRepository extends JpaRepository<User, Long> { }

// Неправильно
public interface UserRepository extends JpaRepository { }
```

### 3. Исключения
Spring Data JPA выбрасывает специфичные исключения:
- `EmptyResultDataAccessException` - когда ожидается один результат, но ничего не найдено
- `IncorrectResultSizeDataAccessException` - когда найдено больше результатов, чем ожидается
- `DataIntegrityViolationException` - при нарушении целостности данных

## Заключение

Интерфейс `Repository` является фундаментом Spring Data JPA. Правильный выбор базового интерфейса и следование лучшим практикам обеспечивает эффективную и надежную работу с данными в Spring приложениях.
