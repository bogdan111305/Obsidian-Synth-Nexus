# Подключение Querydsl

## Обзор

Querydsl - это библиотека для создания типобезопасных запросов в Java. Она позволяет создавать SQL-подобные запросы с помощью Java API, что обеспечивает компиляционную проверку типов и лучшую читаемость кода.

## Основные возможности

### Преимущества Querydsl
- Типобезопасные запросы
- Компиляционная проверка
- Лучшая читаемость кода
- Поддержка различных БД
- Интеграция со Spring Data JPA

### Поддерживаемые модули
- JPA/Hibernate
- SQL
- MongoDB
- Lucene/Solr
- Collections

## Настройка Maven

### Базовые зависимости
```xml
<dependencies>
    <!-- Querydsl JPA -->
    <dependency>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-jpa</artifactId>
        <version>5.0.0</version>
        <classifier>jakarta</classifier>
    </dependency>
    
    <!-- Querydsl APT для генерации Q-классов -->
    <dependency>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-apt</artifactId>
        <version>5.0.0</version>
        <classifier>jakarta</classifier>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

### Конфигурация плагина для генерации Q-классов
```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.mysema.maven</groupId>
            <artifactId>apt-maven-plugin</artifactId>
            <version>1.1.3</version>
            <executions>
                <execution>
                    <goals>
                        <goal>process</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>target/generated-sources/java</outputDirectory>
                        <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## Настройка Gradle

### Базовые зависимости
```gradle
dependencies {
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jakarta'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api:2.1.1'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api:3.1.0'
}

// Конфигурация для генерации Q-классов
def querydslDir = "$buildDir/generated/querydsl"

sourceSets {
    main {
        java {
            srcDirs += [ querydslDir ]
        }
    }
}

configurations {
    querydsl.extendsFrom compileClasspath
}

tasks.withType(JavaCompile) {
    options.annotationProcessorGeneratedSourcesDirectory = file(querydslDir)
}
```

## Конфигурация Spring Boot

### Автоконфигурация
```java
@Configuration
@EnableJpaRepositories(basePackages = "com.example.repository")
public class QuerydslConfig {
    
    @Bean
    public JPAQueryFactory jpaQueryFactory(EntityManager entityManager) {
        return new JPAQueryFactory(entityManager);
    }
}
```

### Интеграция с Spring Data JPA
```java
@Configuration
public class QuerydslJpaConfig {
    
    @Bean
    public QuerydslPredicateExecutorCustomizer querydslPredicateExecutorCustomizer() {
        return new QuerydslPredicateExecutorCustomizer() {
            @Override
            public void customize(QuerydslPredicateExecutor<?> executor) {
                // Кастомизация Querydsl
            }
        };
    }
}
```

## Создание сущностей

### Базовая сущность
```java
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "email", nullable = false, unique = true)
    private String email;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "active", nullable = false)
    private boolean active = true;
    
    @Column(name = "role", nullable = false)
    private String role;
    
    @Column(name = "created_date", nullable = false)
    private LocalDateTime createdDate;
    
    // Геттеры и сеттеры
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
    
    public String getEmail() {
        return email;
    }
    
    public void setEmail(String email) {
        this.email = email;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public boolean isActive() {
        return active;
    }
    
    public void setActive(boolean active) {
        this.active = active;
    }
    
    public String getRole() {
        return role;
    }
    
    public void setRole(String role) {
        this.role = role;
    }
    
    public LocalDateTime getCreatedDate() {
        return createdDate;
    }
    
    public void setCreatedDate(LocalDateTime createdDate) {
        this.createdDate = createdDate;
    }
}
```

### Связанная сущность
```java
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", nullable = false, unique = true)
    private String orderNumber;
    
    @Column(name = "status", nullable = false)
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "total_amount", nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @Column(name = "created_date", nullable = false)
    private LocalDateTime createdDate;
    
    // Геттеры и сеттеры
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
    
    public String getOrderNumber() {
        return orderNumber;
    }
    
    public void setOrderNumber(String orderNumber) {
        this.orderNumber = orderNumber;
    }
    
    public OrderStatus getStatus() {
        return status;
    }
    
    public void setStatus(OrderStatus status) {
        this.status = status;
    }
    
    public BigDecimal getTotalAmount() {
        return totalAmount;
    }
    
    public void setTotalAmount(BigDecimal totalAmount) {
        this.totalAmount = totalAmount;
    }
    
    public User getUser() {
        return user;
    }
    
    public void setUser(User user) {
        this.user = user;
    }
    
    public LocalDateTime getCreatedDate() {
        return createdDate;
    }
    
    public void setCreatedDate(LocalDateTime createdDate) {
        this.createdDate = createdDate;
    }
}
```

## Генерация Q-классов

### Автоматическая генерация
После настройки Maven/Gradle плагина, Q-классы будут генерироваться автоматически при компиляции.

### Пример сгенерированного Q-класса
```java
@Generated("com.querydsl.codegen.EntitySerializer")
public class QUser extends EntityPathBase<User> {
    
    public static final QUser user = new QUser("user");
    
    public final StringPath email = createString("email");
    public final StringPath name = createString("name");
    public final BooleanPath active = createBoolean("active");
    public final StringPath role = createString("role");
    public final DateTimePath<LocalDateTime> createdDate = createDateTime("createdDate", LocalDateTime.class);
    public final NumberPath<Long> id = createNumber("id", Long.class);
    
    public QUser(String variable) {
        super(User.class, PathMetadataFactory.forVariable(variable));
    }
    
    public QUser(Path<? extends User> path) {
        super(path.getType(), path.getMetadata());
    }
    
    public QUser(PathMetadata metadata) {
        super(User.class, metadata);
    }
}
```

## Создание репозитория

### Базовый репозиторий с Querydsl
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long>, 
                                     QuerydslPredicateExecutor<User> {
    
    // Стандартные методы Spring Data JPA
    List<User> findByActive(boolean active);
    Optional<User> findByEmail(String email);
    
    // Кастомные методы с Querydsl
    @Query("SELECT u FROM User u WHERE u.active = ?1")
    List<User> findActiveUsers(boolean active);
}
```

### Кастомный репозиторий с Querydsl
```java
public interface UserRepositoryCustom {
    List<User> findUsersByComplexCriteria(String email, String role, boolean active);
    Page<User> findUsersWithPagination(String searchTerm, Pageable pageable);
}

@Repository
public interface UserRepository extends JpaRepository<User, Long>, 
                                     QuerydslPredicateExecutor<User>,
                                     UserRepositoryCustom {
}
```

## Реализация кастомного репозитория

### Реализация с Querydsl
```java
public class UserRepositoryImpl implements UserRepositoryCustom {
    
    private final JPAQueryFactory queryFactory;
    
    public UserRepositoryImpl(EntityManager entityManager) {
        this.queryFactory = new JPAQueryFactory(entityManager);
    }
    
    @Override
    public List<User> findUsersByComplexCriteria(String email, String role, boolean active) {
        QUser user = QUser.user;
        
        return queryFactory
            .selectFrom(user)
            .where(
                user.email.containsIgnoreCase(email),
                user.role.eq(role),
                user.active.eq(active)
            )
            .orderBy(user.name.asc())
            .fetch();
    }
    
    @Override
    public Page<User> findUsersWithPagination(String searchTerm, Pageable pageable) {
        QUser user = QUser.user;
        
        JPAQuery<User> query = queryFactory
            .selectFrom(user)
            .where(
                user.name.containsIgnoreCase(searchTerm)
                .or(user.email.containsIgnoreCase(searchTerm))
            );
        
        long total = query.fetchCount();
        
        List<User> content = query
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .orderBy(user.name.asc())
            .fetch();
        
        return new PageImpl<>(content, pageable, total);
    }
}
```

## Конфигурация IDE

### IntelliJ IDEA
1. Откройте Settings/Preferences
2. Перейдите в Build, Execution, Deployment > Compiler > Annotation Processors
3. Включите "Enable annotation processing"
4. Укажите "Production sources directory" как "target/generated-sources/java"

### Eclipse
1. Правый клик на проекте > Properties
2. Java Compiler > Annotation Processing
3. Включите "Enable annotation processing"
4. Укажите "Generated source directory" как "target/generated-sources/java"

## Проверка подключения

### Тестовый класс
```java
@SpringBootTest
class QuerydslIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testQuerydslIntegration() {
        // Проверка, что Q-классы доступны
        QUser user = QUser.user;
        assertNotNull(user);
        assertNotNull(user.email);
        assertNotNull(user.name);
        
        // Проверка базовой функциональности
        List<User> users = userRepository.findAll();
        assertNotNull(users);
    }
}
```

### Проверка генерации Q-классов
```java
@Component
public class QuerydslVerification {
    
    public void verifyQClasses() {
        try {
            // Проверка доступности Q-классов
            QUser user = QUser.user;
            QOrder order = QOrder.order;
            
            System.out.println("Q-классы успешно сгенерированы:");
            System.out.println("QUser: " + user.getClass().getName());
            System.out.println("QOrder: " + order.getClass().getName());
            
        } catch (Exception e) {
            System.err.println("Ошибка при доступе к Q-классам: " + e.getMessage());
        }
    }
}
```

## Лучшие практики

### Структура проекта
```
src/
├── main/
│   ├── java/
│   │   └── com/example/
│   │       ├── entity/
│   │       ├── repository/
│   │       └── service/
│   └── resources/
└── test/
    └── java/
        └── com/example/

target/
└── generated-sources/
    └── java/
        └── com/example/
            └── entity/
                ├── QUser.java
                └── QOrder.java
```

### Конфигурация для разных профилей
```yaml
# application-dev.yml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true

# application-prod.yml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
```

## Отладка и логирование

### Включение логирования
```properties
# Логирование Querydsl
logging.level.com.querydsl=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### Проверка сгенерированных запросов
```java
@Component
public class QuerydslDebugger {
    
    public void debugQuery(JPAQuery<?> query) {
        System.out.println("Generated SQL: " + query.toString());
        System.out.println("Parameters: " + query.getMetadata().getParams());
    }
}
```

## Лучшие практики

1. **Настройте автоматическую генерацию Q-классов** в Maven/Gradle
2. **Используйте JPAQueryFactory** для создания запросов
3. **Интегрируйте с Spring Data JPA** для лучшей совместимости
4. **Настройте IDE** для работы с сгенерированными классами
5. **Тестируйте интеграцию** в unit-тестах
6. **Мониторьте производительность** сложных запросов
7. **Документируйте кастомные репозитории** для понимания их назначения 