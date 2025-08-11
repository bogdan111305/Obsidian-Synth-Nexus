# Querydsl в Spring Data JPA

## Обзор

Querydsl - это библиотека для создания типобезопасных запросов в Java. Она предоставляет альтернативу Criteria API с более читаемым и поддерживаемым кодом. Querydsl особенно полезен для сложных динамических запросов.

## Подключение Querydsl

### Зависимости

```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <version>5.0.0</version>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <version>5.0.0</version>
    <scope>provided</scope>
</dependency>
```

### Конфигурация

```java
@Configuration
public class QuerydslConfig {
    
    @Bean
    public JPAQueryFactory jpaQueryFactory(EntityManager entityManager) {
        return new JPAQueryFactory(entityManager);
    }
    
    @Bean
    public QuerydslPredicateExecutorCustomizer querydslPredicateExecutorCustomizer() {
        return config -> config.setDefaultBinding(QuerydslBindings.EMPTY);
    }
}

// Плагин для Maven (добавить в pom.xml)
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
```

## Создание QPredicates

### Базовые Q-классы

```java
// Автоматически генерируемые Q-классы
@Generated("com.querydsl.codegen.EntitySerializer")
public class QUser extends EntityPathBase<User> {
    
    public static final QUser user = new QUser("user");
    
    public final NumberPath<Long> id = createNumber("id", Long.class);
    public final StringPath name = createString("name");
    public final StringPath email = createString("email");
    public final BooleanPath active = createBoolean("active");
    public final NumberPath<Integer> age = createNumber("age", Integer.class);
    public final NumberPath<java.math.BigDecimal> salary = createNumber("salary", java.math.BigDecimal.class);
    
    public final QDepartment department = new QDepartment(this, "department");
    
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

@Generated("com.querydsl.codegen.EntitySerializer")
public class QDepartment extends EntityPathBase<Department> {
    
    public static final QDepartment department = new QDepartment("department");
    
    public final NumberPath<Long> id = createNumber("id", Long.class);
    public final StringPath name = createString("name");
    public final StringPath location = createString("location");
    
    public final ListPath<User, QUser> users = this.<User, QUser>createList("users", User.class, QUser.class, PathInits.DIRECT2);
    
    public QDepartment(String variable) {
        super(Department.class, PathMetadataFactory.forVariable(variable));
    }
    
    public QDepartment(Path<? extends Department> path) {
        super(path.getType(), path.getMetadata());
    }
    
    public QDepartment(PathMetadata metadata) {
        super(Department.class, metadata);
    }
}
```

### Создание предикатов

```java
@Service
public class QuerydslPredicateService {
    
    @Autowired
    private JPAQueryFactory queryFactory;
    
    // Простые предикаты
    public List<User> findUsersByName(String name) {
        return queryFactory
            .selectFrom(QUser.user)
            .where(QUser.user.name.eq(name))
            .fetch();
    }
    
    public List<User> findActiveUsers() {
        return queryFactory
            .selectFrom(QUser.user)
            .where(QUser.user.active.isTrue())
            .fetch();
    }
    
    public List<User> findUsersByAgeRange(int minAge, int maxAge) {
        return queryFactory
            .selectFrom(QUser.user)
            .where(QUser.user.age.between(minAge, maxAge))
            .fetch();
    }
    
    // Сложные предикаты
    public List<User> findUsersByComplexCriteria(UserSearchCriteria criteria) {
        QUser user = QUser.user;
        QDepartment department = QDepartment.department;
        
        BooleanBuilder predicate = new BooleanBuilder();
        
        if (criteria.getName() != null && !criteria.getName().trim().isEmpty()) {
            predicate.and(user.name.containsIgnoreCase(criteria.getName()));
        }
        
        if (criteria.getMinAge() != null) {
            predicate.and(user.age.goe(criteria.getMinAge()));
        }
        
        if (criteria.getMaxAge() != null) {
            predicate.and(user.age.loe(criteria.getMaxAge()));
        }
        
        if (criteria.getDepartmentName() != null && !criteria.getDepartmentName().trim().isEmpty()) {
            predicate.and(department.name.containsIgnoreCase(criteria.getDepartmentName()));
        }
        
        if (criteria.getMinSalary() != null) {
            predicate.and(user.salary.goe(criteria.getMinSalary()));
        }
        
        if (criteria.getMaxSalary() != null) {
            predicate.and(user.salary.loe(criteria.getMaxSalary()));
        }
        
        if (criteria.getActive() != null) {
            predicate.and(user.active.eq(criteria.getActive()));
        }
        
        return queryFactory
            .selectFrom(user)
            .leftJoin(user.department, department)
            .where(predicate)
            .fetch();
    }
    
    // Предикаты с агрегацией
    public List<DepartmentStatistics> getDepartmentStatistics() {
        QUser user = QUser.user;
        QDepartment department = QDepartment.department;
        
        return queryFactory
            .select(Projections.constructor(DepartmentStatistics.class,
                department.name,
                user.count(),
                user.salary.avg(),
                user.salary.max(),
                user.salary.min()))
            .from(user)
            .leftJoin(user.department, department)
            .groupBy(department.id, department.name)
            .fetch();
    }
    
    // Предикаты с подзапросами
    public List<User> findUsersWithHighSalary() {
        QUser user = QUser.user;
        
        return queryFactory
            .selectFrom(user)
            .where(user.salary.gt(
                JPAExpressions
                    .select(user.salary.avg())
                    .from(user)
            ))
            .fetch();
    }
}

// Класс для статистики
@Data
@AllArgsConstructor
public class DepartmentStatistics {
    private String departmentName;
    private Long userCount;
    private Double averageSalary;
    private BigDecimal maxSalary;
    private BigDecimal minSalary;
}
```

## Замена Criteria API на Querydsl

### Сравнение подходов

```java
@Service
public class QuerydslVsCriteriaService {
    
    @Autowired
    private JPAQueryFactory queryFactory;
    
    @Autowired
    private EntityManager entityManager;
    
    // Querydsl подход
    public List<User> findUsersWithQuerydsl(UserSearchCriteria criteria) {
        QUser user = QUser.user;
        QDepartment department = QDepartment.department;
        
        BooleanBuilder predicate = new BooleanBuilder();
        
        if (criteria.getName() != null) {
            predicate.and(user.name.containsIgnoreCase(criteria.getName()));
        }
        
        if (criteria.getMinAge() != null) {
            predicate.and(user.age.goe(criteria.getMinAge()));
        }
        
        if (criteria.getDepartmentName() != null) {
            predicate.and(department.name.containsIgnoreCase(criteria.getDepartmentName()));
        }
        
        return queryFactory
            .selectFrom(user)
            .leftJoin(user.department, department)
            .where(predicate)
            .orderBy(user.name.asc())
            .fetch();
    }
    
    // Criteria API подход (для сравнения)
    public List<User> findUsersWithCriteria(UserSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        Join<User, Department> departmentJoin = root.join("department", JoinType.LEFT);
        
        List<Predicate> predicates = new ArrayList<>();
        
        if (criteria.getName() != null) {
            predicates.add(cb.like(cb.lower(root.get("name")), 
                "%" + criteria.getName().toLowerCase() + "%"));
        }
        
        if (criteria.getMinAge() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("age"), criteria.getMinAge()));
        }
        
        if (criteria.getDepartmentName() != null) {
            predicates.add(cb.like(cb.lower(departmentJoin.get("name")), 
                "%" + criteria.getDepartmentName().toLowerCase() + "%"));
        }
        
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        query.orderBy(cb.asc(root.get("name")));
        
        return entityManager.createQuery(query).getResultList();
    }
    
    // Сложные запросы с Querydsl
    public List<User> findUsersWithComplexQuerydsl(ComplexSearchCriteria criteria) {
        QUser user = QUser.user;
        QDepartment department = QDepartment.department;
        
        BooleanBuilder predicate = new BooleanBuilder();
        
        // Основные условия
        if (criteria.getName() != null) {
            predicate.and(user.name.containsIgnoreCase(criteria.getName()));
        }
        
        // Условия для отдела
        if (criteria.getDepartmentCriteria() != null) {
            DepartmentCriteria deptCriteria = criteria.getDepartmentCriteria();
            
            if (deptCriteria.getName() != null) {
                predicate.and(department.name.containsIgnoreCase(deptCriteria.getName()));
            }
            
            if (deptCriteria.getLocation() != null) {
                predicate.and(department.location.eq(deptCriteria.getLocation()));
            }
        }
        
        // Условия для заказов (если есть)
        if (criteria.getOrderCriteria() != null) {
            OrderCriteria orderCriteria = criteria.getOrderCriteria();
            
            if (orderCriteria.getMinTotal() != null) {
                // Подзапрос для заказов
                predicate.and(JPAExpressions
                    .selectOne()
                    .from(QOrder.order)
                    .where(QOrder.order.user.eq(user)
                        .and(QOrder.order.total.goe(orderCriteria.getMinTotal())))
                    .exists());
            }
        }
        
        return queryFactory
            .selectFrom(user)
            .leftJoin(user.department, department)
            .where(predicate)
            .fetch();
    }
}

// Критерии для сложного поиска
@Data
public class ComplexSearchCriteria {
    private String name;
    private DepartmentCriteria departmentCriteria;
    private OrderCriteria orderCriteria;
}

@Data
public class DepartmentCriteria {
    private String name;
    private String location;
}

@Data
public class OrderCriteria {
    private BigDecimal minTotal;
    private LocalDate orderDateFrom;
}
```

## Класс QuerydslPredicateExecutor

### Интерфейс репозитория

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long>, QuerydslPredicateExecutor<User> {
    
    // Стандартные методы Spring Data JPA
    List<User> findByActiveTrue();
    Optional<User> findByEmail(String email);
    Page<User> findByDepartmentName(String departmentName, Pageable pageable);
    
    // Методы с Querydsl
    default List<User> findUsersWithPredicate(Predicate predicate) {
        return findAll(predicate);
    }
    
    default Page<User> findUsersWithPredicateAndPagination(Predicate predicate, Pageable pageable) {
        return findAll(predicate, pageable);
    }
    
    default List<User> findUsersWithPredicateAndSort(Predicate predicate, Sort sort) {
        return findAll(predicate, sort);
    }
}

// Сервис для работы с Querydsl предикатами
@Service
public class QuerydslPredicateService {
    
    @Autowired
    private UserRepository userRepository;
    
    // Создание предикатов для поиска
    public Predicate createUserSearchPredicate(UserSearchCriteria criteria) {
        QUser user = QUser.user;
        QDepartment department = QDepartment.department;
        
        BooleanBuilder predicate = new BooleanBuilder();
        
        if (criteria.getName() != null && !criteria.getName().trim().isEmpty()) {
            predicate.and(user.name.containsIgnoreCase(criteria.getName()));
        }
        
        if (criteria.getMinAge() != null) {
            predicate.and(user.age.goe(criteria.getMinAge()));
        }
        
        if (criteria.getMaxAge() != null) {
            predicate.and(user.age.loe(criteria.getMaxAge()));
        }
        
        if (criteria.getDepartmentName() != null && !criteria.getDepartmentName().trim().isEmpty()) {
            predicate.and(department.name.containsIgnoreCase(criteria.getDepartmentName()));
        }
        
        if (criteria.getMinSalary() != null) {
            predicate.and(user.salary.goe(criteria.getMinSalary()));
        }
        
        if (criteria.getMaxSalary() != null) {
            predicate.and(user.salary.loe(criteria.getMaxSalary()));
        }
        
        if (criteria.getActive() != null) {
            predicate.and(user.active.eq(criteria.getActive()));
        }
        
        return predicate;
    }
    
    // Поиск пользователей с предикатом
    public List<User> findUsers(UserSearchCriteria criteria) {
        Predicate predicate = createUserSearchPredicate(criteria);
        return userRepository.findUsersWithPredicate(predicate);
    }
    
    // Поиск с пагинацией
    public Page<User> findUsersWithPagination(UserSearchCriteria criteria, Pageable pageable) {
        Predicate predicate = createUserSearchPredicate(criteria);
        return userRepository.findUsersWithPredicateAndPagination(predicate, pageable);
    }
    
    // Поиск с сортировкой
    public List<User> findUsersWithSort(UserSearchCriteria criteria, Sort sort) {
        Predicate predicate = createUserSearchPredicate(criteria);
        return userRepository.findUsersWithPredicateAndSort(predicate, sort);
    }
    
    // Сложные предикаты
    public Predicate createComplexPredicate(ComplexSearchCriteria criteria) {
        QUser user = QUser.user;
        QDepartment department = QDepartment.department;
        
        BooleanBuilder predicate = new BooleanBuilder();
        
        // Основные условия
        if (criteria.getName() != null) {
            predicate.and(user.name.containsIgnoreCase(criteria.getName()));
        }
        
        // Условия для отдела
        if (criteria.getDepartmentCriteria() != null) {
            DepartmentCriteria deptCriteria = criteria.getDepartmentCriteria();
            
            if (deptCriteria.getName() != null) {
                predicate.and(department.name.containsIgnoreCase(deptCriteria.getName()));
            }
            
            if (deptCriteria.getLocation() != null) {
                predicate.and(department.location.eq(deptCriteria.getLocation()));
            }
        }
        
        return predicate;
    }
}
```

### Кастомные предикаты

```java
// Утилитный класс для создания предикатов
public class QuerydslPredicates {
    
    public static BooleanExpression userByName(String name) {
        return name != null ? QUser.user.name.containsIgnoreCase(name) : null;
    }
    
    public static BooleanExpression userByAgeRange(Integer minAge, Integer maxAge) {
        BooleanExpression predicate = null;
        
        if (minAge != null) {
            predicate = QUser.user.age.goe(minAge);
        }
        
        if (maxAge != null) {
            BooleanExpression maxAgePredicate = QUser.user.age.loe(maxAge);
            predicate = predicate != null ? predicate.and(maxAgePredicate) : maxAgePredicate;
        }
        
        return predicate;
    }
    
    public static BooleanExpression userBySalaryRange(BigDecimal minSalary, BigDecimal maxSalary) {
        BooleanExpression predicate = null;
        
        if (minSalary != null) {
            predicate = QUser.user.salary.goe(minSalary);
        }
        
        if (maxSalary != null) {
            BooleanExpression maxSalaryPredicate = QUser.user.salary.loe(maxSalary);
            predicate = predicate != null ? predicate.and(maxSalaryPredicate) : maxSalaryPredicate;
        }
        
        return predicate;
    }
    
    public static BooleanExpression userByDepartment(String departmentName) {
        return departmentName != null ? 
            QDepartment.department.name.containsIgnoreCase(departmentName) : null;
    }
    
    public static BooleanExpression userByActive(Boolean active) {
        return active != null ? QUser.user.active.eq(active) : null;
    }
    
    // Комбинирование предикатов
    public static BooleanExpression combinePredicates(BooleanExpression... predicates) {
        BooleanBuilder builder = new BooleanBuilder();
        
        for (BooleanExpression predicate : predicates) {
            if (predicate != null) {
                builder.and(predicate);
            }
        }
        
        return builder.getValue();
    }
}

// Использование кастомных предикатов
@Service
public class CustomPredicateService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> findUsersWithCustomPredicates(UserSearchCriteria criteria) {
        BooleanExpression predicate = QuerydslPredicates.combinePredicates(
            QuerydslPredicates.userByName(criteria.getName()),
            QuerydslPredicates.userByAgeRange(criteria.getMinAge(), criteria.getMaxAge()),
            QuerydslPredicates.userBySalaryRange(criteria.getMinSalary(), criteria.getMaxSalary()),
            QuerydslPredicates.userByDepartment(criteria.getDepartmentName()),
            QuerydslPredicates.userByActive(criteria.getActive())
        );
        
        return userRepository.findUsersWithPredicate(predicate);
    }
}
```

## Тестирование Querydsl

```java
@SpringBootTest
@Transactional
class QuerydslTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private QuerydslPredicateService predicateService;
    
    @Test
    void testQuerydslPredicates() {
        // Подготовка данных
        User user1 = createUser("John Doe", 25, "IT", new BigDecimal("50000"));
        User user2 = createUser("Jane Smith", 30, "HR", new BigDecimal("60000"));
        userRepository.saveAll(Arrays.asList(user1, user2));
        
        // Создание критериев поиска
        UserSearchCriteria criteria = UserSearchCriteria.builder()
            .name("John")
            .minAge(20)
            .maxAge(30)
            .departmentName("IT")
            .minSalary(new BigDecimal("40000"))
            .active(true)
            .build();
        
        // Выполнение поиска
        List<User> result = predicateService.findUsers(criteria);
        
        // Проверка результатов
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("John Doe");
    }
    
    @Test
    void testQuerydslWithPagination() {
        // Подготовка данных
        List<User> users = createTestUsers();
        userRepository.saveAll(users);
        
        // Создание критериев и пагинации
        UserSearchCriteria criteria = UserSearchCriteria.builder()
            .active(true)
            .build();
        
        Pageable pageable = PageRequest.of(0, 5, Sort.by("name").ascending());
        
        // Выполнение поиска с пагинацией
        Page<User> result = predicateService.findUsersWithPagination(criteria, pageable);
        
        // Проверка результатов
        assertThat(result.getContent()).hasSize(5);
        assertThat(result.getTotalElements()).isGreaterThan(0);
        assertThat(result.getTotalPages()).isGreaterThan(0);
    }
    
    private User createUser(String name, int age, String departmentName, BigDecimal salary) {
        User user = new User();
        user.setName(name);
        user.setAge(age);
        user.setSalary(salary);
        user.setActive(true);
        
        Department department = new Department();
        department.setName(departmentName);
        user.setDepartment(department);
        
        return user;
    }
    
    private List<User> createTestUsers() {
        List<User> users = new ArrayList<>();
        for (int i = 1; i <= 20; i++) {
            User user = new User();
            user.setName("User " + i);
            user.setAge(20 + i);
            user.setSalary(new BigDecimal(40000 + i * 1000));
            user.setActive(true);
            
            Department department = new Department();
            department.setName("Department " + (i % 3 + 1));
            user.setDepartment(department);
            
            users.add(user);
        }
        return users;
    }
}
```

## Заключение

Querydsl предоставляет мощные возможности для создания типобезопасных запросов в Spring Data JPA. Правильное использование Querydsl позволяет:

- Создавать читаемые и поддерживаемые запросы
- Обеспечивать типобезопасность на уровне компиляции
- Упрощать создание сложных динамических запросов
- Улучшать производительность разработки

Ключевые моменты для успешного использования:

1. **Настройте Querydsl правильно** в конфигурации
2. **Используйте Q-классы** для типобезопасных запросов
3. **Создавайте переиспользуемые предикаты** для сложной логики
4. **Комбинируйте с QuerydslPredicateExecutor** для интеграции с Spring Data JPA
5. **Тестируйте функциональность** Querydsl запросов
6. **Мониторьте производительность** и оптимизируйте при необходимости
