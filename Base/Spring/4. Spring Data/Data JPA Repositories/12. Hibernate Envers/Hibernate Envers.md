# Hibernate Envers в Spring Data JPA

## Обзор

Hibernate Envers предоставляет автоматическое версионирование сущностей, позволяя отслеживать все изменения в базе данных. Это особенно полезно для аудита, соответствия требованиям и восстановления данных.

## Подключение Hibernate Envers

### Зависимости

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-envers</artifactId>
</dependency>
```

### Конфигурация

```java
@Configuration
@EnableJpaAuditing
public class EnversConfig {
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource());
        factory.setPackagesToScan("com.example.entity");
        factory.setJpaVendorAdapter(jpaVendorAdapter());
        factory.setJpaProperties(enversProperties());
        return factory;
    }
    
    private Properties enversProperties() {
        Properties properties = new Properties();
        properties.setProperty("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
        properties.setProperty("hibernate.show_sql", "true");
        properties.setProperty("hibernate.format_sql", "true");
        
        // Envers свойства
        properties.setProperty("org.hibernate.envers.audit_table_suffix", "_AUDIT");
        properties.setProperty("org.hibernate.envers.revision_field_name", "REV");
        properties.setProperty("org.hibernate.envers.revision_type_field_name", "REV_TYPE");
        properties.setProperty("org.hibernate.envers.store_data_at_delete", "true");
        properties.setProperty("org.hibernate.envers.global_with_modified_flag", "true");
        
        return properties;
    }
}
```

## Создание сущности Revision

```java
@Entity
@Table(name = "revisions")
@RevisionEntity(CustomRevisionListener.class)
public class Revision {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @RevisionNumber
    private Long id;
    
    @RevisionTimestamp
    private LocalDateTime timestamp;
    
    @Column(name = "user_id")
    private String userId;
    
    @Column(name = "ip_address")
    private String ipAddress;
    
    @Column(name = "user_agent")
    private String userAgent;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public LocalDateTime getTimestamp() { return timestamp; }
    public void setTimestamp(LocalDateTime timestamp) { this.timestamp = timestamp; }
    
    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }
    
    public String getIpAddress() { return ipAddress; }
    public void setIpAddress(String ipAddress) { this.ipAddress = ipAddress; }
    
    public String getUserAgent() { return userAgent; }
    public void setUserAgent(String userAgent) { this.userAgent = userAgent; }
}

// Кастомный RevisionListener
public class CustomRevisionListener implements RevisionListener {
    
    @Override
    public void newRevision(Object revisionEntity) {
        Revision revision = (Revision) revisionEntity;
        
        // Получение информации о текущем пользователе
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication != null && authentication.isAuthenticated()) {
            revision.setUserId(authentication.getName());
        } else {
            revision.setUserId("system");
        }
        
        // Получение IP адреса
        try {
            HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder
                .currentRequestAttributes()).getRequest();
            revision.setIpAddress(request.getRemoteAddr());
            revision.setUserAgent(request.getHeader("User-Agent"));
        } catch (Exception e) {
            revision.setIpAddress("unknown");
            revision.setUserAgent("unknown");
        }
    }
}
```

## Аннотация @Audited

```java
@Entity
@Table(name = "users")
@Audited
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "email", unique = true, nullable = false)
    private String email;
    
    @Column(name = "active")
    private Boolean active = true;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    @Audited(targetAuditMode = RelationTargetAuditMode.NOT_AUDITED)
    private Department department;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public Boolean getActive() { return active; }
    public void setActive(Boolean active) { this.active = active; }
    
    public Department getDepartment() { return department; }
    public void setDepartment(Department department) { this.department = department; }
}

// Сущность с выборочным аудитом
@Entity
@Table(name = "departments")
@Audited(withModifiedFlag = true)
public class Department {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name", nullable = false)
    private String name;
    
    @Column(name = "location")
    @NotAudited
    private String location;
    
    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    @NotAudited
    private List<User> users;
    
    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getLocation() { return location; }
    public void setLocation(String location) { this.location = location; }
    
    public List<User> getUsers() { return users; }
    public void setUsers(List<User> users) { this.users = users; }
}
```

## Аннотация @EnableEnversRepositories

```java
@Configuration
@EnableJpaRepositories(basePackages = "com.example.repository")
@EnableEnversRepositories(basePackages = "com.example.repository.envers")
public class EnversRepositoryConfig {
    
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource());
        factory.setPackagesToScan("com.example.entity");
        factory.setJpaVendorAdapter(jpaVendorAdapter());
        factory.setJpaProperties(enversProperties());
        return factory;
    }
    
    private Properties enversProperties() {
        Properties properties = new Properties();
        properties.setProperty("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
        properties.setProperty("hibernate.show_sql", "true");
        properties.setProperty("hibernate.format_sql", "true");
        properties.setProperty("org.hibernate.envers.audit_table_suffix", "_AUDIT");
        properties.setProperty("org.hibernate.envers.revision_field_name", "REV");
        properties.setProperty("org.hibernate.envers.revision_type_field_name", "REV_TYPE");
        return properties;
    }
}
```

## Класс RevisionRepository

```java
@Repository
public interface RevisionRepository extends JpaRepository<Revision, Long> {
    
    // Поиск ревизий по пользователю
    List<Revision> findByUserIdOrderByTimestampDesc(String userId);
    
    // Поиск ревизий в диапазоне дат
    List<Revision> findByTimestampBetweenOrderByTimestampDesc(
        LocalDateTime startDate, LocalDateTime endDate);
    
    // Поиск ревизий по IP адресу
    List<Revision> findByIpAddressOrderByTimestampDesc(String ipAddress);
    
    // Поиск последних ревизий
    @Query("SELECT r FROM Revision r ORDER BY r.timestamp DESC")
    Page<Revision> findLatestRevisions(Pageable pageable);
    
    // Поиск ревизий с фильтрацией
    @Query("SELECT r FROM Revision r WHERE " +
           "(:userId IS NULL OR r.userId = :userId) AND " +
           "(:ipAddress IS NULL OR r.ipAddress = :ipAddress) AND " +
           "(:startDate IS NULL OR r.timestamp >= :startDate) AND " +
           "(:endDate IS NULL OR r.timestamp <= :endDate) " +
           "ORDER BY r.timestamp DESC")
    Page<Revision> findRevisionsWithFilters(
        @Param("userId") String userId,
        @Param("ipAddress") String ipAddress,
        @Param("startDate") LocalDateTime startDate,
        @Param("endDate") LocalDateTime endDate,
        Pageable pageable
    );
}

// Сервис для работы с аудитом
@Service
public class AuditService {
    
    @Autowired
    private RevisionRepository revisionRepository;
    
    @Autowired
    private AuditReader auditReader;
    
    // Получение истории изменений сущности
    public List<User> getUserHistory(Long userId) {
        return auditReader.createQuery()
            .forRevisionsOfEntity(User.class, false, true)
            .add(AuditEntity.id().eq(userId))
            .getResultList();
    }
    
    // Получение версии сущности на определенную дату
    public User getUserAtRevision(Long userId, Number revision) {
        return auditReader.find(User.class, userId, revision);
    }
    
    // Получение изменений между ревизиями
    public List<Object[]> getChangesBetweenRevisions(Long userId, Number fromRevision, Number toRevision) {
        return auditReader.createQuery()
            .forRevisionsOfEntity(User.class, false, true)
            .add(AuditEntity.id().eq(userId))
            .add(AuditEntity.revisionNumber().between(fromRevision, toRevision))
            .getResultList();
    }
    
    // Получение статистики аудита
    public AuditStatistics getAuditStatistics(LocalDateTime startDate, LocalDateTime endDate) {
        List<Revision> revisions = revisionRepository.findByTimestampBetweenOrderByTimestampDesc(
            startDate, endDate);
        
        Map<String, Long> userActivity = revisions.stream()
            .collect(Collectors.groupingBy(Revision::getUserId, Collectors.counting()));
        
        Map<String, Long> ipActivity = revisions.stream()
            .collect(Collectors.groupingBy(Revision::getIpAddress, Collectors.counting()));
        
        return AuditStatistics.builder()
            .totalRevisions((long) revisions.size())
            .userActivity(userActivity)
            .ipActivity(ipActivity)
            .build();
    }
}

// Класс для статистики аудита
@Data
@Builder
public class AuditStatistics {
    private Long totalRevisions;
    private Map<String, Long> userActivity;
    private Map<String, Long> ipActivity;
}
```

## Тестирование Envers

```java
@SpringBootTest
@Transactional
class EnversTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private AuditService auditService;
    
    @Test
    void testUserAuditing() {
        // Создание пользователя
        User user = new User();
        user.setName("John Doe");
        user.setEmail("john@example.com");
        
        User savedUser = userRepository.save(user);
        
        // Обновление пользователя
        savedUser.setName("John Smith");
        userRepository.save(savedUser);
        
        // Получение истории изменений
        List<User> history = auditService.getUserHistory(savedUser.getId());
        
        // Проверка, что есть две версии
        assertThat(history).hasSize(2);
        assertThat(history.get(0).getName()).isEqualTo("John Smith");
        assertThat(history.get(1).getName()).isEqualTo("John Doe");
    }
    
    @Test
    void testRevisionTracking() {
        // Создание пользователя
        User user = new User();
        user.setName("Test User");
        user.setEmail("test@example.com");
        
        User savedUser = userRepository.save(user);
        
        // Получение ревизий
        List<Revision> revisions = revisionRepository.findByUserIdOrderByTimestampDesc("system");
        
        // Проверка, что ревизия создана
        assertThat(revisions).isNotEmpty();
        assertThat(revisions.get(0).getUserId()).isEqualTo("system");
        assertThat(revisions.get(0).getTimestamp()).isNotNull();
    }
}
```

## Заключение

Hibernate Envers предоставляет мощные возможности для автоматического версионирования сущностей. Правильное использование Envers позволяет:

- Отслеживать все изменения в базе данных
- Восстанавливать данные на определенные моменты времени
- Обеспечивать соответствие требованиям к аудиту
- Анализировать активность пользователей

Ключевые моменты для успешного использования:

1. **Настройте Envers правильно** в конфигурации
2. **Используйте @Audited** для сущностей, требующих версионирования
3. **Создайте кастомную сущность Revision** для дополнительной информации
4. **Используйте AuditReader** для работы с историей изменений
5. **Тестируйте функциональность** версионирования
6. **Мониторьте производительность** аудиторских операций
