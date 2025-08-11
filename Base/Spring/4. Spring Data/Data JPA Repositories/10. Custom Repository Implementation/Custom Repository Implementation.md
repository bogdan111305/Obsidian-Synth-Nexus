# Custom Repository Implementation в Spring Data JPA

## Обзор

Custom Repository Implementation позволяет создавать собственные реализации методов репозитория, которые не могут быть выражены через стандартные механизмы Spring Data JPA. Это особенно полезно для сложных запросов, специфичной бизнес-логики и интеграции с внешними системами.

## Запрос фильтрации через Custom Implementation

### Базовая структура Custom Implementation

```java
// Интерфейс для custom методов
public interface UserRepositoryCustom {
    List<User> findUsersByComplexCriteria(UserSearchCriteria criteria);
    List<User> findUsersWithCustomFiltering(Map<String, Object> filters);
    Page<User> findUsersWithAdvancedPagination(UserSearchCriteria criteria, Pageable pageable);
    List<User> findUsersByCustomQuery(String customQuery, Object... parameters);
}

// Реализация custom методов
public class UserRepositoryImpl implements UserRepositoryCustom {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public List<User> findUsersByComplexCriteria(UserSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        // Добавление условий на основе критериев
        if (criteria.getName() != null && !criteria.getName().trim().isEmpty()) {
            predicates.add(cb.like(cb.lower(root.get("name")), 
                "%" + criteria.getName().toLowerCase() + "%"));
        }
        
        if (criteria.getMinAge() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("age"), criteria.getMinAge()));
        }
        
        if (criteria.getMaxAge() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("age"), criteria.getMaxAge()));
        }
        
        if (criteria.getDepartmentName() != null && !criteria.getDepartmentName().trim().isEmpty()) {
            Join<User, Department> departmentJoin = root.join("department");
            predicates.add(cb.like(cb.lower(departmentJoin.get("name")), 
                "%" + criteria.getDepartmentName().toLowerCase() + "%"));
        }
        
        if (criteria.getMinSalary() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("salary"), criteria.getMinSalary()));
        }
        
        if (criteria.getMaxSalary() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("salary"), criteria.getMaxSalary()));
        }
        
        if (criteria.getActive() != null) {
            predicates.add(cb.equal(root.get("active"), criteria.getActive()));
        }
        
        // Применение условий
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        // Сортировка
        if (criteria.getSortBy() != null) {
            if ("asc".equalsIgnoreCase(criteria.getSortDirection())) {
                query.orderBy(cb.asc(root.get(criteria.getSortBy())));
            } else {
                query.orderBy(cb.desc(root.get(criteria.getSortBy())));
            }
        }
        
        return entityManager.createQuery(query).getResultList();
    }
    
    @Override
    public List<User> findUsersWithCustomFiltering(Map<String, Object> filters) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        // Динамическое добавление фильтров
        filters.forEach((field, value) -> {
            if (value != null) {
                if (value instanceof String && !((String) value).trim().isEmpty()) {
                    predicates.add(cb.like(cb.lower(root.get(field)), 
                        "%" + ((String) value).toLowerCase() + "%"));
                } else if (value instanceof Number) {
                    predicates.add(cb.equal(root.get(field), value));
                } else if (value instanceof Boolean) {
                    predicates.add(cb.equal(root.get(field), value));
                }
            }
        });
        
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        return entityManager.createQuery(query).getResultList();
    }
    
    @Override
    public Page<User> findUsersWithAdvancedPagination(UserSearchCriteria criteria, Pageable pageable) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        // Построение условий
        List<Predicate> predicates = buildPredicates(criteria, cb, root);
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        // Применение сортировки
        if (pageable.getSort().isSorted()) {
            List<Order> orders = new ArrayList<>();
            pageable.getSort().forEach(sort -> {
                if (sort.getDirection().isAscending()) {
                    orders.add(cb.asc(root.get(sort.getProperty())));
                } else {
                    orders.add(cb.desc(root.get(sort.getProperty())));
                }
            });
            query.orderBy(orders);
        }
        
        // Выполнение запроса с пагинацией
        TypedQuery<User> typedQuery = entityManager.createQuery(query);
        typedQuery.setFirstResult((int) pageable.getOffset());
        typedQuery.setMaxResults(pageable.getPageSize());
        
        List<User> content = typedQuery.getResultList();
        
        // Подсчет общего количества
        CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
        Root<User> countRoot = countQuery.from(User.class);
        countQuery.select(cb.count(countRoot));
        
        List<Predicate> countPredicates = buildPredicates(criteria, cb, countRoot);
        if (!countPredicates.isEmpty()) {
            countQuery.where(countPredicates.toArray(new Predicate[0]));
        }
        
        Long total = entityManager.createQuery(countQuery).getSingleResult();
        
        return new PageImpl<>(content, pageable, total);
    }
    
    @Override
    public List<User> findUsersByCustomQuery(String customQuery, Object... parameters) {
        Query query = entityManager.createQuery(customQuery);
        
        // Установка параметров
        for (int i = 0; i < parameters.length; i++) {
            query.setParameter(i + 1, parameters[i]);
        }
        
        return query.getResultList();
    }
    
    private List<Predicate> buildPredicates(UserSearchCriteria criteria, CriteriaBuilder cb, Root<User> root) {
        List<Predicate> predicates = new ArrayList<>();
        
        if (criteria.getName() != null && !criteria.getName().trim().isEmpty()) {
            predicates.add(cb.like(cb.lower(root.get("name")), 
                "%" + criteria.getName().toLowerCase() + "%"));
        }
        
        if (criteria.getMinAge() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("age"), criteria.getMinAge()));
        }
        
        if (criteria.getMaxAge() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("age"), criteria.getMaxAge()));
        }
        
        if (criteria.getDepartmentName() != null && !criteria.getDepartmentName().trim().isEmpty()) {
            Join<User, Department> departmentJoin = root.join("department");
            predicates.add(cb.like(cb.lower(departmentJoin.get("name")), 
                "%" + criteria.getDepartmentName().toLowerCase() + "%"));
        }
        
        if (criteria.getMinSalary() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("salary"), criteria.getMinSalary()));
        }
        
        if (criteria.getMaxSalary() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("salary"), criteria.getMaxSalary()));
        }
        
        if (criteria.getActive() != null) {
            predicates.add(cb.equal(root.get("active"), criteria.getActive()));
        }
        
        return predicates;
    }
}

// Основной интерфейс репозитория
@Repository
public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom {
    // Стандартные методы Spring Data JPA
    List<User> findByActiveTrue();
    Optional<User> findByEmail(String email);
    Page<User> findByDepartmentName(String departmentName, Pageable pageable);
}
```

### Модели данных

```java
// Критерии поиска
@Data
@Builder
public class UserSearchCriteria {
    private String name;
    private Integer minAge;
    private Integer maxAge;
    private String departmentName;
    private BigDecimal minSalary;
    private BigDecimal maxSalary;
    private Boolean active;
    private String sortBy;
    private String sortDirection;
}

// Сущности
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    private Integer age;
    private BigDecimal salary;
    private Boolean active;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
    
    // Геттеры и сеттеры
}

@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String location;
    
    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    private List<User> users;
    
    // Геттеры и сеттеры
}
```

## Criteria API для запроса фильтрации

### Расширенная реализация с Criteria API

```java
public class AdvancedUserRepositoryImpl implements UserRepositoryCustom {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public List<User> findUsersByComplexCriteria(UserSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        // Создание предикатов
        List<Predicate> predicates = createPredicates(criteria, cb, root);
        
        // Применение предикатов
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        // Применение сортировки
        applySorting(criteria, cb, root, query);
        
        // Применение группировки
        if (criteria.isGroupByDepartment()) {
            Join<User, Department> departmentJoin = root.join("department");
            query.groupBy(departmentJoin.get("id"));
        }
        
        return entityManager.createQuery(query).getResultList();
    }
    
    private List<Predicate> createPredicates(UserSearchCriteria criteria, CriteriaBuilder cb, Root<User> root) {
        List<Predicate> predicates = new ArrayList<>();
        
        // Фильтр по имени (поиск с учетом регистра)
        if (criteria.getName() != null && !criteria.getName().trim().isEmpty()) {
            if (criteria.isExactMatch()) {
                predicates.add(cb.equal(root.get("name"), criteria.getName()));
            } else {
                predicates.add(cb.like(cb.lower(root.get("name")), 
                    "%" + criteria.getName().toLowerCase() + "%"));
            }
        }
        
        // Фильтр по возрасту
        if (criteria.getMinAge() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("age"), criteria.getMinAge()));
        }
        
        if (criteria.getMaxAge() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("age"), criteria.getMaxAge()));
        }
        
        // Фильтр по отделу
        if (criteria.getDepartmentName() != null && !criteria.getDepartmentName().trim().isEmpty()) {
            Join<User, Department> departmentJoin = root.join("department");
            predicates.add(cb.like(cb.lower(departmentJoin.get("name")), 
                "%" + criteria.getDepartmentName().toLowerCase() + "%"));
        }
        
        // Фильтр по зарплате
        if (criteria.getMinSalary() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("salary"), criteria.getMinSalary()));
        }
        
        if (criteria.getMaxSalary() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("salary"), criteria.getMaxSalary()));
        }
        
        // Фильтр по активности
        if (criteria.getActive() != null) {
            predicates.add(cb.equal(root.get("active"), criteria.getActive()));
        }
        
        // Фильтр по дате регистрации
        if (criteria.getRegistrationDateFrom() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("registrationDate"), 
                criteria.getRegistrationDateFrom()));
        }
        
        if (criteria.getRegistrationDateTo() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("registrationDate"), 
                criteria.getRegistrationDateTo()));
        }
        
        // Фильтр по количеству заказов
        if (criteria.getMinOrderCount() != null) {
            Subquery<Long> orderCountSubquery = cb.createSubquery(Long.class);
            Root<Order> orderRoot = orderCountSubquery.from(Order.class);
            orderCountSubquery.select(cb.count(orderRoot))
                .where(cb.equal(orderRoot.get("user"), root));
            
            predicates.add(cb.greaterThanOrEqualTo(orderCountSubquery, criteria.getMinOrderCount()));
        }
        
        return predicates;
    }
    
    private void applySorting(UserSearchCriteria criteria, CriteriaBuilder cb, Root<User> root, CriteriaQuery<User> query) {
        if (criteria.getSortBy() != null) {
            String sortDirection = criteria.getSortDirection() != null ? 
                criteria.getSortDirection().toLowerCase() : "asc";
            
            if ("asc".equals(sortDirection)) {
                query.orderBy(cb.asc(root.get(criteria.getSortBy())));
            } else {
                query.orderBy(cb.desc(root.get(criteria.getSortBy())));
            }
        }
    }
    
    // Метод для поиска пользователей с агрегацией
    public List<UserStatistics> findUserStatistics(UserSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<UserStatistics> query = cb.createQuery(UserStatistics.class);
        Root<User> root = query.from(User.class);
        
        // Создание предикатов
        List<Predicate> predicates = createPredicates(criteria, cb, root);
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        // Группировка по отделу
        Join<User, Department> departmentJoin = root.join("department");
        query.groupBy(departmentJoin.get("id"), departmentJoin.get("name"));
        
        // Выборка агрегированных данных
        query.select(cb.construct(UserStatistics.class,
            departmentJoin.get("name"),
            cb.count(root),
            cb.avg(root.get("salary")),
            cb.max(root.get("salary")),
            cb.min(root.get("salary"))
        ));
        
        return entityManager.createQuery(query).getResultList();
    }
}

// Класс для статистики
public class UserStatistics {
    private String departmentName;
    private Long userCount;
    private Double averageSalary;
    private BigDecimal maxSalary;
    private BigDecimal minSalary;
    
    public UserStatistics(String departmentName, Long userCount, Double averageSalary, 
                        BigDecimal maxSalary, BigDecimal minSalary) {
        this.departmentName = departmentName;
        this.userCount = userCount;
        this.averageSalary = averageSalary;
        this.maxSalary = maxSalary;
        this.minSalary = minSalary;
    }
    
    // Геттеры
    public String getDepartmentName() { return departmentName; }
    public Long getUserCount() { return userCount; }
    public Double getAverageSalary() { return averageSalary; }
    public BigDecimal getMaxSalary() { return maxSalary; }
    public BigDecimal getMinSalary() { return minSalary; }
}
```

### Сложные сценарии с Criteria API

```java
public class ComplexUserRepositoryImpl implements UserRepositoryCustom {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    // Поиск пользователей с вложенными условиями
    public List<User> findUsersWithNestedCriteria(ComplexSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        // Основные условия
        if (criteria.getName() != null) {
            predicates.add(cb.like(root.get("name"), "%" + criteria.getName() + "%"));
        }
        
        // Условия для отдела
        if (criteria.getDepartmentCriteria() != null) {
            Join<User, Department> departmentJoin = root.join("department");
            DepartmentCriteria deptCriteria = criteria.getDepartmentCriteria();
            
            if (deptCriteria.getName() != null) {
                predicates.add(cb.like(departmentJoin.get("name"), 
                    "%" + deptCriteria.getName() + "%"));
            }
            
            if (deptCriteria.getLocation() != null) {
                predicates.add(cb.equal(departmentJoin.get("location"), deptCriteria.getLocation()));
            }
        }
        
        // Условия для заказов
        if (criteria.getOrderCriteria() != null) {
            Join<User, Order> orderJoin = root.join("orders");
            OrderCriteria orderCriteria = criteria.getOrderCriteria();
            
            if (orderCriteria.getMinTotal() != null) {
                predicates.add(cb.greaterThanOrEqualTo(orderJoin.get("total"), 
                    orderCriteria.getMinTotal()));
            }
            
            if (orderCriteria.getOrderDateFrom() != null) {
                predicates.add(cb.greaterThanOrEqualTo(orderJoin.get("orderDate"), 
                    orderCriteria.getOrderDateFrom()));
            }
        }
        
        // Применение условий
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        return entityManager.createQuery(query).getResultList();
    }
    
    // Поиск с использованием подзапросов
    public List<User> findUsersWithSubqueries(SubquerySearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        // Подзапрос для поиска пользователей с заказами выше среднего
        if (criteria.isAboveAverageOrders()) {
            Subquery<Double> avgOrderSubquery = cb.createSubquery(Double.class);
            Root<Order> orderRoot = avgOrderSubquery.from(Order.class);
            avgOrderSubquery.select(cb.avg(orderRoot.get("total")));
            
            Subquery<Double> userOrderSubquery = cb.createSubquery(Double.class);
            Root<Order> userOrderRoot = userOrderSubquery.from(Order.class);
            userOrderSubquery.select(cb.avg(userOrderRoot.get("total")))
                .where(cb.equal(userOrderRoot.get("user"), root));
            
            predicates.add(cb.greaterThan(userOrderSubquery, avgOrderSubquery));
        }
        
        // Подзапрос для поиска пользователей с наибольшим количеством заказов
        if (criteria.isTopOrderUsers()) {
            Subquery<Long> maxOrderSubquery = cb.createSubquery(Long.class);
            Root<Order> maxOrderRoot = maxOrderSubquery.from(Order.class);
            maxOrderSubquery.select(cb.count(maxOrderRoot))
                .groupBy(maxOrderRoot.get("user"))
                .orderBy(cb.desc(cb.count(maxOrderRoot)))
                .limit(1);
            
            Subquery<Long> userOrderCountSubquery = cb.createSubquery(Long.class);
            Root<Order> userOrderCountRoot = userOrderCountSubquery.from(Order.class);
            userOrderCountSubquery.select(cb.count(userOrderCountRoot))
                .where(cb.equal(userOrderCountRoot.get("user"), root));
            
            predicates.add(cb.equal(userOrderCountSubquery, maxOrderSubquery));
        }
        
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        return entityManager.createQuery(query).getResultList();
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

@Data
public class SubquerySearchCriteria {
    private boolean aboveAverageOrders;
    private boolean topOrderUsers;
}
```

## Аннотация @EnableJpaRepository

### Базовая конфигурация

```java
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.repository",
    entityManagerFactoryRef = "entityManagerFactory",
    transactionManagerRef = "transactionManager",
    repositoryImplementationPostfix = "Impl",
    repositoryBaseClass = CustomJpaRepositoryImpl.class
)
public class JpaRepositoryConfig {
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource());
        factory.setPackagesToScan("com.example.entity");
        factory.setJpaVendorAdapter(jpaVendorAdapter());
        factory.setJpaProperties(jpaProperties());
        return factory;
    }
    
    @Bean
    public PlatformTransactionManager transactionManager() {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(entityManagerFactory().getObject());
        return transactionManager;
    }
    
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/testdb");
        dataSource.setUsername("root");
        dataSource.setPassword("password");
        return dataSource;
    }
    
    @Bean
    public JpaVendorAdapter jpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setDatabase(Database.MYSQL);
        adapter.setShowSql(true);
        adapter.setGenerateDdl(true);
        return adapter;
    }
    
    @Bean
    public Properties jpaProperties() {
        Properties properties = new Properties();
        properties.setProperty("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
        properties.setProperty("hibernate.show_sql", "true");
        properties.setProperty("hibernate.format_sql", "true");
        return properties;
    }
}
```

### Кастомная базовая реализация

```java
// Кастомная базовая реализация репозитория
public class CustomJpaRepositoryImpl<T, ID extends Serializable> 
    extends SimpleJpaRepository<T, ID> implements CustomJpaRepository<T, ID> {
    
    private final EntityManager entityManager;
    
    public CustomJpaRepositoryImpl(JpaEntityInformation<T, ?> entityInformation, 
                                 EntityManager entityManager) {
        super(entityInformation, entityManager);
        this.entityManager = entityManager;
    }
    
    @Override
    public List<T> findByCustomCriteria(CustomCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<T> query = cb.createQuery(getDomainClass());
        Root<T> root = query.from(getDomainClass());
        
        List<Predicate> predicates = buildCustomPredicates(criteria, cb, root);
        
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        return entityManager.createQuery(query).getResultList();
    }
    
    @Override
    public Page<T> findByCustomCriteriaWithPagination(CustomCriteria criteria, Pageable pageable) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<T> query = cb.createQuery(getDomainClass());
        Root<T> root = query.from(getDomainClass());
        
        List<Predicate> predicates = buildCustomPredicates(criteria, cb, root);
        
        if (!predicates.isEmpty()) {
            query.where(predicates.toArray(new Predicate[0]));
        }
        
        // Применение сортировки
        if (pageable.getSort().isSorted()) {
            List<Order> orders = new ArrayList<>();
            pageable.getSort().forEach(sort -> {
                if (sort.getDirection().isAscending()) {
                    orders.add(cb.asc(root.get(sort.getProperty())));
                } else {
                    orders.add(cb.desc(root.get(sort.getProperty())));
                }
            });
            query.orderBy(orders);
        }
        
        // Выполнение запроса с пагинацией
        TypedQuery<T> typedQuery = entityManager.createQuery(query);
        typedQuery.setFirstResult((int) pageable.getOffset());
        typedQuery.setMaxResults(pageable.getPageSize());
        
        List<T> content = typedQuery.getResultList();
        
        // Подсчет общего количества
        CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
        Root<T> countRoot = countQuery.from(getDomainClass());
        countQuery.select(cb.count(countRoot));
        
        List<Predicate> countPredicates = buildCustomPredicates(criteria, cb, countRoot);
        if (!countPredicates.isEmpty()) {
            countQuery.where(countPredicates.toArray(new Predicate[0]));
        }
        
        Long total = entityManager.createQuery(countQuery).getSingleResult();
        
        return new PageImpl<>(content, pageable, total);
    }
    
    private List<Predicate> buildCustomPredicates(CustomCriteria criteria, CriteriaBuilder cb, Root<T> root) {
        List<Predicate> predicates = new ArrayList<>();
        
        // Базовая реализация - может быть переопределена в наследниках
        if (criteria.getFilters() != null) {
            criteria.getFilters().forEach((field, value) -> {
                if (value != null) {
                    if (value instanceof String && !((String) value).trim().isEmpty()) {
                        predicates.add(cb.like(cb.lower(root.get(field)), 
                            "%" + ((String) value).toLowerCase() + "%"));
                    } else {
                        predicates.add(cb.equal(root.get(field), value));
                    }
                }
            });
        }
        
        return predicates;
    }
}

// Интерфейс для кастомных методов
public interface CustomJpaRepository<T, ID extends Serializable> extends JpaRepository<T, ID> {
    List<T> findByCustomCriteria(CustomCriteria criteria);
    Page<T> findByCustomCriteriaWithPagination(CustomCriteria criteria, Pageable pageable);
}

// Критерии для кастомного поиска
@Data
public class CustomCriteria {
    private Map<String, Object> filters;
    private String sortBy;
    private String sortDirection;
}
```

### Расширенная конфигурация

```java
@Configuration
@EnableJpaRepositories(
    basePackages = {
        "com.example.repository.user",
        "com.example.repository.department",
        "com.example.repository.order"
    },
    entityManagerFactoryRef = "entityManagerFactory",
    transactionManagerRef = "transactionManager",
    repositoryImplementationPostfix = "Impl",
    repositoryBaseClass = CustomJpaRepositoryImpl.class,
    includeFilters = {
        @ComponentScan.Filter(type = FilterType.REGEX, pattern = ".*Repository")
    },
    excludeFilters = {
        @ComponentScan.Filter(type = FilterType.REGEX, pattern = ".*TestRepository")
    }
)
@EnableTransactionManagement
public class AdvancedJpaRepositoryConfig {
    
    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource());
        factory.setPackagesToScan("com.example.entity");
        factory.setJpaVendorAdapter(jpaVendorAdapter());
        factory.setJpaProperties(jpaProperties());
        factory.setLoadTimeWeaver(loadTimeWeaver());
        return factory;
    }
    
    @Bean
    public LoadTimeWeaver loadTimeWeaver() {
        return new InstrumentationLoadTimeWeaver();
    }
    
    @Bean
    public PlatformTransactionManager transactionManager() {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(entityManagerFactory().getObject());
        transactionManager.setDefaultTimeout(30);
        transactionManager.setRollbackOnCommitFailure(true);
        return transactionManager;
    }
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/testdb");
        config.setUsername("root");
        config.setPassword("password");
        config.setDriverClassName("com.mysql.cj.jdbc.Driver");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        return new HikariDataSource(config);
    }
    
    @Bean
    public JpaVendorAdapter jpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setDatabase(Database.MYSQL);
        adapter.setShowSql(true);
        adapter.setGenerateDdl(true);
        adapter.setDatabasePlatform("org.hibernate.dialect.MySQL8Dialect");
        return adapter;
    }
    
    @Bean
    public Properties jpaProperties() {
        Properties properties = new Properties();
        properties.setProperty("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
        properties.setProperty("hibernate.show_sql", "true");
        properties.setProperty("hibernate.format_sql", "true");
        properties.setProperty("hibernate.use_sql_comments", "true");
        properties.setProperty("hibernate.jdbc.batch_size", "20");
        properties.setProperty("hibernate.order_inserts", "true");
        properties.setProperty("hibernate.order_updates", "true");
        properties.setProperty("hibernate.jdbc.batch_versioned_data", "true");
        properties.setProperty("hibernate.cache.use_second_level_cache", "true");
        properties.setProperty("hibernate.cache.region.factory_class", 
            "org.hibernate.cache.ehcache.EhCacheRegionFactory");
        return properties;
    }
}
```

## Лучшие практики

### 1. Структура Custom Implementation

```java
@Service
public class CustomRepositoryBestPractices {
    
    @Autowired
    private UserRepository userRepository;
    
    // Хорошо: Использование custom методов для сложной логики
    public List<User> findUsersByComplexCriteria(UserSearchCriteria criteria) {
        return userRepository.findUsersByComplexCriteria(criteria);
    }
    
    // Хорошо: Комбинирование стандартных и custom методов
    public Page<User> findActiveUsersWithCustomFiltering(UserSearchCriteria criteria, Pageable pageable) {
        if (criteria.isSimple()) {
            // Использование стандартных методов для простых случаев
            return userRepository.findByActiveTrue(pageable);
        } else {
            // Использование custom методов для сложных случаев
            return userRepository.findUsersWithAdvancedPagination(criteria, pageable);
        }
    }
    
    // Плохо: Создание сложной логики в сервисе
    public List<User> findUsersBad(UserSearchCriteria criteria) {
        // Сложная логика должна быть в custom implementation
        List<User> allUsers = userRepository.findAll();
        return allUsers.stream()
            .filter(user -> criteria.getName() == null || 
                user.getName().contains(criteria.getName()))
            .filter(user -> criteria.getMinAge() == null || 
                user.getAge() >= criteria.getMinAge())
            .collect(Collectors.toList());
    }
}
```

### 2. Оптимизация производительности

```java
@Service
public class CustomRepositoryPerformanceService {
    
    @Autowired
    private UserRepository userRepository;
    
    // Мониторинг производительности custom методов
    public <T> T monitorCustomQuery(Supplier<T> querySupplier, String queryName) {
        long startTime = System.currentTimeMillis();
        
        try {
            T result = querySupplier.get();
            
            long endTime = System.currentTimeMillis();
            long duration = endTime - startTime;
            
            logQueryPerformance(queryName, duration);
            
            return result;
        } catch (Exception e) {
            logQueryError(queryName, e);
            throw e;
        }
    }
    
    private void logQueryPerformance(String queryName, long duration) {
        System.out.printf("Custom query '%s' выполнен за %dms%n", queryName, duration);
    }
    
    private void logQueryError(String queryName, Exception e) {
        System.err.printf("Ошибка в custom query '%s': %s%n", queryName, e.getMessage());
    }
    
    // Кэширование результатов custom запросов
    @Cacheable(value = "customUserQueries", key = "#criteria.hashCode() + '_' + #pageable.hashCode()")
    public Page<User> getCachedCustomUsers(UserSearchCriteria criteria, Pageable pageable) {
        return userRepository.findUsersWithAdvancedPagination(criteria, pageable);
    }
}
```

### 3. Тестирование Custom Implementation

```java
@SpringBootTest
@Transactional
class CustomRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testFindUsersByComplexCriteria() {
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
        
        // Выполнение custom запроса
        List<User> result = userRepository.findUsersByComplexCriteria(criteria);
        
        // Проверка результатов
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("John Doe");
    }
    
    @Test
    void testFindUsersWithAdvancedPagination() {
        // Подготовка данных
        List<User> users = createTestUsers();
        userRepository.saveAll(users);
        
        // Создание критериев и пагинации
        UserSearchCriteria criteria = UserSearchCriteria.builder()
            .active(true)
            .sortBy("name")
            .sortDirection("asc")
            .build();
        
        Pageable pageable = PageRequest.of(0, 5);
        
        // Выполнение custom запроса с пагинацией
        Page<User> result = userRepository.findUsersWithAdvancedPagination(criteria, pageable);
        
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

Custom Repository Implementation предоставляет мощные возможности для создания сложной бизнес-логики в Spring Data JPA. Правильное использование custom implementation позволяет:

- Создавать сложные запросы, которые невозможно выразить стандартными методами
- Оптимизировать производительность для специфичных сценариев
- Интегрировать внешние системы и API
- Создавать переиспользуемую бизнес-логику

Ключевые моменты для успешного использования:

1. **Используйте custom implementation** только для сложной логики
2. **Применяйте Criteria API** для динамических запросов
3. **Настройте @EnableJpaRepository** правильно
4. **Мониторьте производительность** custom методов
5. **Тестируйте custom implementation** тщательно
6. **Кэшируйте результаты** для часто используемых запросов
