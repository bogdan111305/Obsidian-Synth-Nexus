# CompanyRepository IT - Создание интеграционных тестов

## Обзор

Интеграционные тесты для `CompanyRepository` позволяют проверить корректность работы с базой данных в реальных условиях. Эти тесты выполняются в транзакциях и обеспечивают изоляцию тестовых данных.

## Структура тестового класса

### Базовая настройка

```java
@ExtendWith(SpringExtension.class)
@DataJpaTest
@Transactional
@Rollback
class CompanyRepositoryIT {
    
    @Autowired
    private CompanyRepository companyRepository;
    
    @Autowired
    private TestEntityManager testEntityManager;
    
    @Test
    void contextLoads() {
        assertThat(companyRepository).isNotNull();
    }
}
```

### Конфигурация тестовой базы данных

```java
@TestConfiguration
@EnableJpaRepositories(basePackages = "com.example.repository")
@EntityScan(basePackages = "com.example.entity")
public class TestConfig {
    
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("schema.sql")
            .addScript("data.sql")
            .build();
    }
}
```

## Тестирование CRUD операций

### 1. Создание компании

```java
@Test
@Transactional
@Rollback
void testCreateCompany() {
    // Given
    Company company = new Company();
    company.setName("Test Company");
    company.setAddress("Test Address");
    
    // When
    Company savedCompany = companyRepository.save(company);
    
    // Then
    assertThat(savedCompany.getId()).isNotNull();
    assertThat(savedCompany.getName()).isEqualTo("Test Company");
    
    // Проверяем, что компания действительно сохранена в БД
    Company foundCompany = testEntityManager.find(Company.class, savedCompany.getId());
    assertThat(foundCompany).isNotNull();
    assertThat(foundCompany.getName()).isEqualTo("Test Company");
}
```

### 2. Поиск компании

```java
@Test
@Transactional
@Rollback
void testFindCompanyById() {
    // Given
    Company company = new Company();
    company.setName("Test Company");
    Company savedCompany = companyRepository.save(company);
    
    // When
    Optional<Company> foundCompany = companyRepository.findById(savedCompany.getId());
    
    // Then
    assertThat(foundCompany).isPresent();
    assertThat(foundCompany.get().getName()).isEqualTo("Test Company");
}
```

### 3. Обновление компании

```java
@Test
@Transactional
@Rollback
void testUpdateCompany() {
    // Given
    Company company = new Company();
    company.setName("Original Name");
    Company savedCompany = companyRepository.save(company);
    
    // When
    savedCompany.setName("Updated Name");
    Company updatedCompany = companyRepository.save(savedCompany);
    
    // Then
    assertThat(updatedCompany.getName()).isEqualTo("Updated Name");
    
    // Проверяем в БД
    Company foundCompany = testEntityManager.find(Company.class, savedCompany.getId());
    assertThat(foundCompany.getName()).isEqualTo("Updated Name");
}
```

### 4. Удаление компании

```java
@Test
@Transactional
@Rollback
void testDeleteCompany() {
    // Given
    Company company = new Company();
    company.setName("Test Company");
    Company savedCompany = companyRepository.save(company);
    
    // When
    companyRepository.deleteById(savedCompany.getId());
    
    // Then
    Optional<Company> foundCompany = companyRepository.findById(savedCompany.getId());
    assertThat(foundCompany).isEmpty();
}
```

## Тестирование кастомных запросов

### 1. Поиск по имени

```java
@Test
@Transactional
@Rollback
void testFindByName() {
    // Given
    Company company1 = new Company();
    company1.setName("Test Company 1");
    companyRepository.save(company1);
    
    Company company2 = new Company();
    company2.setName("Test Company 2");
    companyRepository.save(company2);
    
    // When
    List<Company> companies = companyRepository.findByNameContaining("Test");
    
    // Then
    assertThat(companies).hasSize(2);
    assertThat(companies).extracting("name")
        .contains("Test Company 1", "Test Company 2");
}
```

### 2. Поиск по адресу

```java
@Test
@Transactional
@Rollback
void testFindByAddress() {
    // Given
    Company company = new Company();
    company.setName("Test Company");
    company.setAddress("Test Address");
    companyRepository.save(company);
    
    // When
    Optional<Company> foundCompany = companyRepository.findByAddress("Test Address");
    
    // Then
    assertThat(foundCompany).isPresent();
    assertThat(foundCompany.get().getName()).isEqualTo("Test Company");
}
```

## Тестирование с использованием @Query

### 1. JPQL запросы

```java
@Test
@Transactional
@Rollback
void testFindCompaniesByCustomQuery() {
    // Given
    Company company1 = new Company();
    company1.setName("Tech Company");
    companyRepository.save(company1);
    
    Company company2 = new Company();
    company2.setName("Finance Company");
    companyRepository.save(company2);
    
    // When
    List<Company> techCompanies = companyRepository.findTechCompanies();
    
    // Then
    assertThat(techCompanies).hasSize(1);
    assertThat(techCompanies.get(0).getName()).isEqualTo("Tech Company");
}
```

### 2. Нативные SQL запросы

```java
@Test
@Transactional
@Rollback
void testFindCompaniesByNativeQuery() {
    // Given
    Company company = new Company();
    company.setName("Test Company");
    company.setAddress("Test Address");
    companyRepository.save(company);
    
    // When
    List<Company> companies = companyRepository.findCompaniesByAddress("Test Address");
    
    // Then
    assertThat(companies).hasSize(1);
    assertThat(companies.get(0).getName()).isEqualTo("Test Company");
}
```

## Тестирование транзакций

### 1. Проверка отката транзакции

```java
@Test
@Transactional
@Rollback
void testTransactionRollback() {
    // Given
    Company company = new Company();
    company.setName("Test Company");
    Company savedCompany = companyRepository.save(company);
    
    // When - симулируем исключение
    try {
        companyRepository.saveAndThrowException(savedCompany);
    } catch (RuntimeException e) {
        // Ожидаемое исключение
    }
    
    // Then - проверяем, что транзакция откатилась
    Optional<Company> foundCompany = companyRepository.findById(savedCompany.getId());
    assertThat(foundCompany).isEmpty();
}
```

### 2. Проверка коммита транзакции

```java
@Test
@Transactional
@Commit
void testTransactionCommit() {
    // Given
    Company company = new Company();
    company.setName("Test Company");
    
    // When
    Company savedCompany = companyRepository.save(company);
    
    // Then - проверяем, что транзакция зафиксирована
    Company foundCompany = testEntityManager.find(Company.class, savedCompany.getId());
    assertThat(foundCompany).isNotNull();
    assertThat(foundCompany.getName()).isEqualTo("Test Company");
}
```

## Тестирование производительности

### 1. Тест с большим количеством данных

```java
@Test
@Transactional
@Rollback
void testBulkOperations() {
    // Given
    List<Company> companies = new ArrayList<>();
    for (int i = 0; i < 1000; i++) {
        Company company = new Company();
        company.setName("Company " + i);
        company.setAddress("Address " + i);
        companies.add(company);
    }
    
    // When
    long startTime = System.currentTimeMillis();
    List<Company> savedCompanies = companyRepository.saveAll(companies);
    long endTime = System.currentTimeMillis();
    
    // Then
    assertThat(savedCompanies).hasSize(1000);
    assertThat(endTime - startTime).isLessThan(5000); // Не более 5 секунд
}
```

### 2. Тест с пагинацией

```java
@Test
@Transactional
@Rollback
void testPagination() {
    // Given
    for (int i = 0; i < 50; i++) {
        Company company = new Company();
        company.setName("Company " + i);
        companyRepository.save(company);
    }
    
    // When
    Page<Company> page = companyRepository.findAll(PageRequest.of(0, 10));
    
    // Then
    assertThat(page.getContent()).hasSize(10);
    assertThat(page.getTotalElements()).isEqualTo(50);
    assertThat(page.getTotalPages()).isEqualTo(5);
}
```

## Настройка тестовых данных

### 1. Использование @BeforeEach

```java
@BeforeEach
void setUp() {
    // Очищаем базу данных перед каждым тестом
    companyRepository.deleteAll();
    
    // Создаем тестовые данные
    Company company1 = new Company();
    company1.setName("Test Company 1");
    companyRepository.save(company1);
    
    Company company2 = new Company();
    company2.setName("Test Company 2");
    companyRepository.save(company2);
}
```

### 2. Использование SQL скриптов

```sql
-- schema.sql
CREATE TABLE IF NOT EXISTS company (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address VARCHAR(255)
);

-- data.sql
INSERT INTO company (name, address) VALUES 
('Test Company 1', 'Address 1'),
('Test Company 2', 'Address 2');
```

## Обработка исключений

### 1. Тест с проверкой исключений

```java
@Test
@Transactional
@Rollback
void testDuplicateCompanyName() {
    // Given
    Company company1 = new Company();
    company1.setName("Test Company");
    companyRepository.save(company1);
    
    Company company2 = new Company();
    company2.setName("Test Company"); // Дублирующее имя
    
    // When & Then
    assertThatThrownBy(() -> companyRepository.save(company2))
        .isInstanceOf(DataIntegrityViolationException.class);
}
```

### 2. Тест с кастомными исключениями

```java
@Test
@Transactional
@Rollback
void testCompanyNotFoundException() {
    // Given
    Long nonExistentId = 999L;
    
    // When & Then
    assertThatThrownBy(() -> companyRepository.findById(nonExistentId)
        .orElseThrow(() -> new CompanyNotFoundException("Company not found")))
        .isInstanceOf(CompanyNotFoundException.class)
        .hasMessage("Company not found");
}
```

## Лучшие практики

### 1. Изоляция тестов
```java
@Test
@Transactional
@Rollback
void testIsolated() {
    // Каждый тест должен быть независимым
    // Не полагайтесь на порядок выполнения тестов
}
```

### 2. Использование @DirtiesContext
```java
@Test
@DirtiesContext
void testThatModifiesContext() {
    // Используйте только когда тест модифицирует контекст Spring
}
```

### 3. Оптимизация производительности
```java
@Test
@Transactional
@Rollback
void testOptimized() {
    // Используйте batch операции для больших объемов данных
    // Избегайте N+1 проблем в тестах
}
``` 
