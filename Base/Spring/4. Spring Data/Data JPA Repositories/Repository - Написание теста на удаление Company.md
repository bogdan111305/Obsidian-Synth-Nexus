# Написание теста на удаление Company

## Обзор

Тестирование операций удаления в Spring Data JPA является критически важным для обеспечения целостности данных и корректной работы приложения. В этой статье рассмотрим различные подходы к тестированию удаления сущности Company.

## Подготовка тестовой среды

### Зависимости для тестирования
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Модель данных
```java
@Entity
@Table(name = "companies")
public class Company {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String name;
    
    @Column(nullable = false)
    private String address;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @OneToMany(mappedBy = "company", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Employee> employees = new ArrayList<>();
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
    
    // Конструкторы, геттеры, сеттеры
}
```

### Репозиторий
```java
@Repository
public interface CompanyRepository extends JpaRepository<Company, Long> {
    
    Optional<Company> findByName(String name);
    
    List<Company> findByAddressContaining(String address);
    
    @Query("SELECT c FROM Company c WHERE c.employees.size > :minEmployees")
    List<Company> findByEmployeeCountGreaterThan(@Param("minEmployees") int minEmployees);
    
    @Modifying
    @Query("DELETE FROM Company c WHERE c.name = :name")
    int deleteByName(@Param("name") String name);
    
    @Modifying
    @Query("DELETE FROM Company c WHERE c.employees.size = 0")
    int deleteCompaniesWithoutEmployees();
}
```

## Unit тестирование

### Базовый тест удаления по ID
```java
@ExtendWith(MockitoExtension.class)
class CompanyRepositoryTest {
    
    @Mock
    private CompanyRepository companyRepository;
    
    @Test
    void testDeleteById() {
        // given
        Long companyId = 1L;
        Company company = new Company();
        company.setId(companyId);
        company.setName("Test Company");
        
        // when
        companyRepository.deleteById(companyId);
        
        // then
        verify(companyRepository, times(1)).deleteById(companyId);
    }
    
    @Test
    void testDeleteById_WhenCompanyNotExists() {
        // given
        Long nonExistentId = 999L;
        
        // when & then
        assertDoesNotThrow(() -> companyRepository.deleteById(nonExistentId));
        verify(companyRepository, times(1)).deleteById(nonExistentId);
    }
}
```

### Тест удаления сущности
```java
@Test
void testDeleteEntity() {
    // given
    Company company = new Company();
    company.setId(1L);
    company.setName("Test Company");
    company.setAddress("Test Address");
    
    // when
    companyRepository.delete(company);
    
    // then
    verify(companyRepository, times(1)).delete(company);
}

@Test
void testDeleteEntity_WithEmployees() {
    // given
    Company company = new Company();
    company.setId(1L);
    company.setName("Test Company");
    
    Employee employee = new Employee();
    employee.setId(1L);
    employee.setName("John Doe");
    employee.setCompany(company);
    
    company.getEmployees().add(employee);
    
    // when
    companyRepository.delete(company);
    
    // then
    verify(companyRepository, times(1)).delete(company);
}
```

### Тест кастомного метода удаления
```java
@Test
void testDeleteByName() {
    // given
    String companyName = "Test Company";
    when(companyRepository.deleteByName(companyName)).thenReturn(1);
    
    // when
    int deletedCount = companyRepository.deleteByName(companyName);
    
    // then
    assertEquals(1, deletedCount);
    verify(companyRepository, times(1)).deleteByName(companyName);
}

@Test
void testDeleteByName_WhenCompanyNotExists() {
    // given
    String nonExistentName = "Non Existent Company";
    when(companyRepository.deleteByName(nonExistentName)).thenReturn(0);
    
    // when
    int deletedCount = companyRepository.deleteByName(nonExistentName);
    
    // then
    assertEquals(0, deletedCount);
    verify(companyRepository, times(1)).deleteByName(nonExistentName);
}
```

## Интеграционное тестирование

### Базовый интеграционный тест
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class CompanyRepositoryIntegrationTest {
    
    @Autowired
    private CompanyRepository companyRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Test
    void testDeleteById_Integration() {
        // given
        Company company = new Company();
        company.setName("Test Company");
        company.setAddress("Test Address");
        
        Company savedCompany = companyRepository.save(company);
        Long companyId = savedCompany.getId();
        
        // verify company exists
        assertTrue(companyRepository.findById(companyId).isPresent());
        
        // when
        companyRepository.deleteById(companyId);
        
        // then
        assertFalse(companyRepository.findById(companyId).isPresent());
    }
    
    @Test
    void testDeleteEntity_Integration() {
        // given
        Company company = new Company();
        company.setName("Test Company");
        company.setAddress("Test Address");
        
        Company savedCompany = companyRepository.save(company);
        
        // verify company exists
        assertTrue(companyRepository.findById(savedCompany.getId()).isPresent());
        
        // when
        companyRepository.delete(savedCompany);
        
        // then
        assertFalse(companyRepository.findById(savedCompany.getId()).isPresent());
    }
}
```

### Тест каскадного удаления
```java
@Test
void testDeleteCompany_WithEmployees_Cascade() {
    // given
    Company company = new Company();
    company.setName("Test Company");
    company.setAddress("Test Address");
    
    Employee employee1 = new Employee();
    employee1.setName("John Doe");
    employee1.setCompany(company);
    
    Employee employee2 = new Employee();
    employee2.setName("Jane Smith");
    employee2.setCompany(company);
    
    company.getEmployees().add(employee1);
    company.getEmployees().add(employee2);
    
    Company savedCompany = companyRepository.save(company);
    
    // verify company and employees exist
    assertTrue(companyRepository.findById(savedCompany.getId()).isPresent());
    assertEquals(2, savedCompany.getEmployees().size());
    
    // when
    companyRepository.delete(savedCompany);
    
    // then
    assertFalse(companyRepository.findById(savedCompany.getId()).isPresent());
}
```

### Тест кастомного метода удаления
```java
@Test
void testDeleteByName_Integration() {
    // given
    Company company1 = new Company();
    company1.setName("Test Company 1");
    company1.setAddress("Address 1");
    
    Company company2 = new Company();
    company2.setName("Test Company 2");
    company2.setAddress("Address 2");
    
    companyRepository.save(company1);
    companyRepository.save(company2);
    
    // verify both companies exist
    assertEquals(2, companyRepository.count());
    
    // when
    int deletedCount = companyRepository.deleteByName("Test Company 1");
    
    // then
    assertEquals(1, deletedCount);
    assertEquals(1, companyRepository.count());
    assertFalse(companyRepository.findByName("Test Company 1").isPresent());
    assertTrue(companyRepository.findByName("Test Company 2").isPresent());
}

@Test
void testDeleteCompaniesWithoutEmployees_Integration() {
    // given
    Company companyWithEmployees = new Company();
    companyWithEmployees.setName("Company with Employees");
    companyWithEmployees.setAddress("Address 1");
    
    Employee employee = new Employee();
    employee.setName("John Doe");
    employee.setCompany(companyWithEmployees);
    companyWithEmployees.getEmployees().add(employee);
    
    Company companyWithoutEmployees = new Company();
    companyWithoutEmployees.setName("Company without Employees");
    companyWithoutEmployees.setAddress("Address 2");
    
    companyRepository.save(companyWithEmployees);
    companyRepository.save(companyWithoutEmployees);
    
    // verify both companies exist
    assertEquals(2, companyRepository.count());
    
    // when
    int deletedCount = companyRepository.deleteCompaniesWithoutEmployees();
    
    // then
    assertEquals(1, deletedCount);
    assertEquals(1, companyRepository.count());
    assertTrue(companyRepository.findByName("Company with Employees").isPresent());
    assertFalse(companyRepository.findByName("Company without Employees").isPresent());
}
```

## Тестирование с транзакциями

### Тест транзакционного удаления
```java
@SpringBootTest
@Transactional
class CompanyRepositoryTransactionalTest {
    
    @Autowired
    private CompanyRepository companyRepository;
    
    @Autowired
    private EntityManager entityManager;
    
    @Test
    void testDeleteInTransaction() {
        // given
        Company company = new Company();
        company.setName("Test Company");
        company.setAddress("Test Address");
        
        Company savedCompany = companyRepository.save(company);
        
        // verify company exists
        assertTrue(companyRepository.findById(savedCompany.getId()).isPresent());
        
        // when
        companyRepository.deleteById(savedCompany.getId());
        
        // then
        assertFalse(companyRepository.findById(savedCompany.getId()).isPresent());
    }
    
    @Test
    void testDeleteWithRollback() {
        // given
        Company company = new Company();
        company.setName("Test Company");
        company.setAddress("Test Address");
        
        Company savedCompany = companyRepository.save(company);
        Long companyId = savedCompany.getId();
        
        // verify company exists
        assertTrue(companyRepository.findById(companyId).isPresent());
        
        try {
            // when - simulate error after delete
            companyRepository.deleteById(companyId);
            throw new RuntimeException("Simulated error");
        } catch (RuntimeException e) {
            // then - transaction should be rolled back
            assertTrue(companyRepository.findById(companyId).isPresent());
        }
    }
}
```

## Тестирование производительности

### Тест массового удаления
```java
@Test
void testBulkDelete_Performance() {
    // given
    List<Company> companies = new ArrayList<>();
    for (int i = 0; i < 1000; i++) {
        Company company = new Company();
        company.setName("Company " + i);
        company.setAddress("Address " + i);
        companies.add(company);
    }
    
    companyRepository.saveAll(companies);
    assertEquals(1000, companyRepository.count());
    
    // when
    long startTime = System.currentTimeMillis();
    companyRepository.deleteAll();
    long endTime = System.currentTimeMillis();
    
    // then
    assertEquals(0, companyRepository.count());
    long duration = endTime - startTime;
    assertTrue(duration < 5000, "Bulk delete took too long: " + duration + "ms");
}
```

### Тест удаления с условиями
```java
@Test
void testDeleteWithConditions_Performance() {
    // given
    List<Company> companies = new ArrayList<>();
    for (int i = 0; i < 100; i++) {
        Company company = new Company();
        company.setName("Company " + i);
        company.setAddress(i % 2 == 0 ? "Even Address" : "Odd Address");
        companies.add(company);
    }
    
    companyRepository.saveAll(companies);
    assertEquals(100, companyRepository.count());
    
    // when
    long startTime = System.currentTimeMillis();
    List<Company> companiesToDelete = companyRepository.findByAddressContaining("Even");
    companyRepository.deleteAll(companiesToDelete);
    long endTime = System.currentTimeMillis();
    
    // then
    assertEquals(50, companyRepository.count());
    long duration = endTime - startTime;
    assertTrue(duration < 1000, "Conditional delete took too long: " + duration + "ms");
}
```

## Тестирование исключений

### Тест исключений при удалении
```java
@Test
void testDeleteById_WhenReferencedByForeignKey() {
    // given
    Company company = new Company();
    company.setName("Test Company");
    company.setAddress("Test Address");
    
    Employee employee = new Employee();
    employee.setName("John Doe");
    employee.setCompany(company);
    company.getEmployees().add(employee);
    
    Company savedCompany = companyRepository.save(company);
    
    // when & then
    assertThrows(DataIntegrityViolationException.class, () -> {
        companyRepository.deleteById(savedCompany.getId());
    });
}

@Test
void testDeleteByName_WhenNoRowsAffected() {
    // given
    String nonExistentName = "Non Existent Company";
    
    // when
    int deletedCount = companyRepository.deleteByName(nonExistentName);
    
    // then
    assertEquals(0, deletedCount);
}
```

## Тестирование с мок-данными

### Тест с @MockBean
```java
@SpringBootTest
class CompanyServiceTest {
    
    @MockBean
    private CompanyRepository companyRepository;
    
    @Autowired
    private CompanyService companyService;
    
    @Test
    void testDeleteCompany_Service() {
        // given
        Long companyId = 1L;
        Company company = new Company();
        company.setId(companyId);
        company.setName("Test Company");
        
        when(companyRepository.findById(companyId)).thenReturn(Optional.of(company));
        doNothing().when(companyRepository).deleteById(companyId);
        
        // when
        companyService.deleteCompany(companyId);
        
        // then
        verify(companyRepository, times(1)).findById(companyId);
        verify(companyRepository, times(1)).deleteById(companyId);
    }
    
    @Test
    void testDeleteCompany_WhenCompanyNotExists() {
        // given
        Long companyId = 1L;
        when(companyRepository.findById(companyId)).thenReturn(Optional.empty());
        
        // when & then
        assertThrows(CompanyNotFoundException.class, () -> {
            companyService.deleteCompany(companyId);
        });
        
        verify(companyRepository, times(1)).findById(companyId);
        verify(companyRepository, never()).deleteById(any());
    }
}
```

## Лучшие практики тестирования удаления

### 1. Подготовка данных
```java
@BeforeEach
void setUp() {
    companyRepository.deleteAll();
    entityManager.flush();
    entityManager.clear();
}
```

### 2. Проверка состояния после удаления
```java
@Test
void testDeleteAndVerify() {
    // given
    Company company = createTestCompany();
    Company savedCompany = companyRepository.save(company);
    
    // when
    companyRepository.deleteById(savedCompany.getId());
    
    // then
    assertFalse(companyRepository.findById(savedCompany.getId()).isPresent());
    assertEquals(0, companyRepository.count());
}
```

### 3. Тестирование каскадных операций
```java
@Test
void testCascadeDelete() {
    // given
    Company company = createCompanyWithEmployees();
    Company savedCompany = companyRepository.save(company);
    
    // verify cascade setup
    assertTrue(savedCompany.getEmployees().size() > 0);
    
    // when
    companyRepository.deleteById(savedCompany.getId());
    
    // then
    assertFalse(companyRepository.findById(savedCompany.getId()).isPresent());
    // Verify employees are also deleted (if cascade is configured)
}
```

### 4. Тестирование производительности
```java
@Test
void testDeletePerformance() {
    // given
    List<Company> companies = createLargeDataset();
    companyRepository.saveAll(companies);
    
    // when
    long startTime = System.currentTimeMillis();
    companyRepository.deleteAll();
    long endTime = System.currentTimeMillis();
    
    // then
    long duration = endTime - startTime;
    assertTrue(duration < 5000, "Delete operation took too long: " + duration + "ms");
}
```

## Заключение

Правильное тестирование операций удаления в Spring Data JPA включает в себя:
- Unit тесты с моками для изоляции логики
- Интеграционные тесты для проверки реального взаимодействия с БД
- Тесты транзакций для проверки атомарности операций
- Тесты производительности для больших объемов данных
- Тесты исключений для обработки ошибок

Комплексный подход к тестированию удаления обеспечивает надежность и корректность работы приложения.
