# Page & Slice в Spring Data JPA

## Обзор

Spring Data JPA предоставляет мощные возможности для работы с пагинацией и сортировкой через классы `Page`, `Slice` и `Pageable`. Эти классы позволяют эффективно обрабатывать большие наборы данных, разбивая их на управляемые порции.

## Spring классы Streamable, Slice, Page

### Streamable

`Streamable` - это интерфейс, который позволяет работать с коллекциями как со стримами:

```java
public interface Streamable<T> extends Iterable<T> {
    Streamable<T> filter(Predicate<? super T> predicate);
    <R> Streamable<R> map(Function<? super T, ? extends R> mapper);
    Streamable<T> and(Streamable<? extends T> streamable);
    Streamable<T> or(Streamable<? extends T> streamable);
    Streamable<T> not(Streamable<? extends T> streamable);
}
```

**Пример использования:**

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Streamable<User> findByActiveTrue();
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public Streamable<User> getActiveUsersWithFilter() {
        return userRepository.findByActiveTrue()
            .filter(user -> user.getAge() > 18)
            .map(user -> {
                user.setName(user.getName().toUpperCase());
                return user;
            });
    }
}
```

### Slice

`Slice` представляет собой "срез" данных с информацией о наличии следующей страницы:

```java
public interface Slice<T> extends Streamable<T> {
    int getNumber();           // Номер текущей страницы (0-based)
    int getSize();            // Размер страницы
    int getNumberOfElements(); // Количество элементов на текущей странице
    List<T> getContent();     // Содержимое страницы
    boolean hasContent();      // Есть ли элементы на странице
    Sort getSort();           // Информация о сортировке
    boolean isFirst();        // Это первая страница?
    boolean isLast();         // Это последняя страница?
    boolean hasNext();        // Есть ли следующая страница?
    boolean hasPrevious();    // Есть ли предыдущая страница?
    Pageable nextPageable();  // Pageable для следующей страницы
    Pageable previousPageable(); // Pageable для предыдущей страницы
}
```

### Page

`Page` расширяет `Slice` и добавляет информацию о общем количестве элементов и страниц:

```java
public interface Page<T> extends Slice<T> {
    int getTotalPages();      // Общее количество страниц
    long getTotalElements();  // Общее количество элементов
    boolean hasNext();        // Есть ли следующая страница?
    boolean hasPrevious();    // Есть ли предыдущая страница?
}
```

## Демонстрация работы Slice объекта

### Базовое использование Slice

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Slice<User> findByDepartmentName(String departmentName, Pageable pageable);
    Slice<User> findByAgeBetween(int minAge, int maxAge, Pageable pageable);
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public Slice<User> getUsersByDepartment(String departmentName, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        return userRepository.findByDepartmentName(departmentName, pageable);
    }
    
    public void demonstrateSliceUsage() {
        Slice<User> slice = getUsersByDepartment("IT", 0, 10);
        
        System.out.println("Номер страницы: " + slice.getNumber());
        System.out.println("Размер страницы: " + slice.getSize());
        System.out.println("Количество элементов: " + slice.getNumberOfElements());
        System.out.println("Есть следующая страница: " + slice.hasNext());
        System.out.println("Это первая страница: " + slice.isFirst());
        System.out.println("Это последняя страница: " + slice.isLast());
        
        // Перебор элементов
        slice.forEach(user -> System.out.println(user.getName()));
        
        // Получение следующей страницы
        if (slice.hasNext()) {
            Pageable nextPageable = slice.nextPageable();
            Slice<User> nextSlice = userRepository.findByDepartmentName("IT", nextPageable);
        }
    }
}
```

### Сложные запросы с Slice

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.department.name = :deptName " +
           "AND u.age >= :minAge AND u.salary >= :minSalary")
    Slice<User> findUsersByDepartmentAndCriteria(
        @Param("deptName") String deptName,
        @Param("minAge") int minAge,
        @Param("minSalary") BigDecimal minSalary,
        Pageable pageable
    );
    
    @Query(value = "SELECT u.* FROM users u " +
                   "JOIN departments d ON u.department_id = d.id " +
                   "WHERE d.name = :deptName AND u.age >= :minAge",
           nativeQuery = true)
    Slice<User> findUsersByDepartmentNative(
        @Param("deptName") String deptName,
        @Param("minAge") int minAge,
        Pageable pageable
    );
}

@Service
public class AdvancedUserService {
    @Autowired
    private UserRepository userRepository;
    
    public void processUsersInBatches(String departmentName) {
        int page = 0;
        int size = 100;
        boolean hasNext = true;
        
        while (hasNext) {
            Pageable pageable = PageRequest.of(page, size, 
                Sort.by("salary").descending().and(Sort.by("name").ascending()));
            
            Slice<User> slice = userRepository.findUsersByDepartmentAndCriteria(
                departmentName, 25, new BigDecimal("50000"), pageable);
            
            // Обработка текущей страницы
            processUsers(slice.getContent());
            
            hasNext = slice.hasNext();
            page++;
        }
    }
    
    private void processUsers(List<User> users) {
        users.forEach(user -> {
            // Логика обработки пользователя
            System.out.println("Обработка: " + user.getName());
        });
    }
}
```

## Почему Slice объекта недостаточно

### Ограничения Slice

1. **Отсутствие информации о общем количестве:**
   ```java
   Slice<User> slice = userRepository.findByActiveTrue(PageRequest.of(0, 10));
   
   // НЕ можем получить:
   // slice.getTotalElements() - не существует
   // slice.getTotalPages() - не существует
   
   // Можем только проверить наличие следующей страницы:
   boolean hasNext = slice.hasNext();
   ```

2. **Сложность создания UI пагинации:**
   ```java
   // Без общего количества элементов сложно создать навигацию
   public class PaginationInfo {
       private int currentPage;
       private int pageSize;
       private boolean hasNext;
       private boolean hasPrevious;
       // НЕТ: private long totalElements;
       // НЕТ: private int totalPages;
   }
   ```

3. **Ограниченная аналитика:**
   ```java
   // Не можем показать пользователю прогресс
   // "Показано 10 из ??? элементов"
   
   // Не можем создать полную навигацию
   // "Страница 1 из ???"
   ```

### Когда использовать Slice

```java
@Service
public class SliceUsageExamples {
    @Autowired
    private UserRepository userRepository;
    
    // Хорошо для Slice - обработка больших данных
    public void processLargeDataset() {
        int page = 0;
        Slice<User> slice;
        
        do {
            Pageable pageable = PageRequest.of(page, 1000);
            slice = userRepository.findByActiveTrue(pageable);
            
            // Обработка данных
            processBatch(slice.getContent());
            
            page++;
        } while (slice.hasNext());
    }
    
    // Хорошо для Slice - бесконечная прокрутка
    public Slice<User> getUsersForInfiniteScroll(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return userRepository.findByActiveTrue(pageable);
    }
    
    // Хорошо для Slice - экспорт данных
    public void exportUsersToFile(String filename) {
        try (PrintWriter writer = new PrintWriter(new FileWriter(filename))) {
            int page = 0;
            Slice<User> slice;
            
            do {
                Pageable pageable = PageRequest.of(page, 500);
                slice = userRepository.findAll(pageable);
                
                slice.getContent().forEach(user -> 
                    writer.println(user.getName() + "," + user.getEmail()));
                
                page++;
            } while (slice.hasNext());
        }
    }
}
```

## Демонстрация работы Page объекта

### Базовое использование Page

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByDepartmentName(String departmentName, Pageable pageable);
    Page<User> findByAgeBetween(int minAge, int maxAge, Pageable pageable);
    Page<User> findByActiveTrue(Pageable pageable);
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public Page<User> getUsersByDepartment(String departmentName, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        return userRepository.findByDepartmentName(departmentName, pageable);
    }
    
    public void demonstratePageUsage() {
        Page<User> page = getUsersByDepartment("IT", 0, 10);
        
        System.out.println("Номер страницы: " + page.getNumber());
        System.out.println("Размер страницы: " + page.getSize());
        System.out.println("Количество элементов на странице: " + page.getNumberOfElements());
        System.out.println("Общее количество элементов: " + page.getTotalElements());
        System.out.println("Общее количество страниц: " + page.getTotalPages());
        System.out.println("Есть следующая страница: " + page.hasNext());
        System.out.println("Есть предыдущая страница: " + page.hasPrevious());
        System.out.println("Это первая страница: " + page.isFirst());
        System.out.println("Это последняя страница: " + page.isLast());
        
        // Перебор элементов
        page.forEach(user -> System.out.println(user.getName()));
    }
}
```

### Создание UI пагинации с Page

```java
@Service
public class PaginationService {
    @Autowired
    private UserRepository userRepository;
    
    public PaginationInfo createPaginationInfo(String departmentName, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        Page<User> pageResult = userRepository.findByDepartmentName(departmentName, pageable);
        
        return PaginationInfo.builder()
            .currentPage(pageResult.getNumber())
            .pageSize(pageResult.getSize())
            .totalElements(pageResult.getTotalElements())
            .totalPages(pageResult.getTotalPages())
            .hasNext(pageResult.hasNext())
            .hasPrevious(pageResult.hasPrevious())
            .isFirst(pageResult.isFirst())
            .isLast(pageResult.isLast())
            .build();
    }
    
    public List<Integer> getPageNumbers(int currentPage, int totalPages) {
        List<Integer> pageNumbers = new ArrayList<>();
        
        int start = Math.max(0, currentPage - 2);
        int end = Math.min(totalPages - 1, currentPage + 2);
        
        for (int i = start; i <= end; i++) {
            pageNumbers.add(i);
        }
        
        return pageNumbers;
    }
}

@Data
@Builder
public class PaginationInfo {
    private int currentPage;
    private int pageSize;
    private long totalElements;
    private int totalPages;
    private boolean hasNext;
    private boolean hasPrevious;
    private boolean isFirst;
    private boolean isLast;
}
```

### Сложные запросы с Page

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u " +
           "JOIN u.department d " +
           "WHERE d.name = :deptName " +
           "AND u.age >= :minAge " +
           "AND u.salary >= :minSalary " +
           "AND u.active = true")
    Page<User> findActiveUsersByDepartmentAndCriteria(
        @Param("deptName") String deptName,
        @Param("minAge") int minAge,
        @Param("minSalary") BigDecimal minSalary,
        Pageable pageable
    );
    
    @Query(value = "SELECT u.* FROM users u " +
                   "JOIN departments d ON u.department_id = d.id " +
                   "WHERE d.name = :deptName " +
                   "AND u.age >= :minAge " +
                   "AND u.salary >= :minSalary " +
                   "AND u.active = true",
           countQuery = "SELECT COUNT(u.id) FROM users u " +
                       "JOIN departments d ON u.department_id = d.id " +
                       "WHERE d.name = :deptName " +
                       "AND u.age >= :minAge " +
                       "AND u.salary >= :minSalary " +
                       "AND u.active = true",
           nativeQuery = true)
    Page<User> findActiveUsersByDepartmentNative(
        @Param("deptName") String deptName,
        @Param("minAge") int minAge,
        @Param("minSalary") BigDecimal minSalary,
        Pageable pageable
    );
}

@Service
public class AdvancedUserService {
    @Autowired
    private UserRepository userRepository;
    
    public UserSearchResult searchUsers(UserSearchCriteria criteria) {
        Pageable pageable = PageRequest.of(
            criteria.getPage(), 
            criteria.getSize(), 
            createSort(criteria.getSortBy(), criteria.getSortDirection())
        );
        
        Page<User> page = userRepository.findActiveUsersByDepartmentAndCriteria(
            criteria.getDepartmentName(),
            criteria.getMinAge(),
            criteria.getMinSalary(),
            pageable
        );
        
        return UserSearchResult.builder()
            .users(page.getContent())
            .paginationInfo(createPaginationInfo(page))
            .build();
    }
    
    private Sort createSort(String sortBy, String sortDirection) {
        Sort.Direction direction = "desc".equalsIgnoreCase(sortDirection) 
            ? Sort.Direction.DESC 
            : Sort.Direction.ASC;
        
        return Sort.by(direction, sortBy);
    }
    
    private PaginationInfo createPaginationInfo(Page<User> page) {
        return PaginationInfo.builder()
            .currentPage(page.getNumber())
            .pageSize(page.getSize())
            .totalElements(page.getTotalElements())
            .totalPages(page.getTotalPages())
            .hasNext(page.hasNext())
            .hasPrevious(page.hasPrevious())
            .isFirst(page.isFirst())
            .isLast(page.isLast())
            .build();
    }
}

@Data
@Builder
public class UserSearchCriteria {
    private String departmentName;
    private int minAge;
    private BigDecimal minSalary;
    private int page;
    private int size;
    private String sortBy;
    private String sortDirection;
}

@Data
@Builder
public class UserSearchResult {
    private List<User> users;
    private PaginationInfo paginationInfo;
}
```

## Класс Pageable

`Pageable` - это интерфейс, который определяет параметры пагинации и сортировки:

```java
public interface Pageable {
    int getPageNumber();      // Номер страницы (0-based)
    int getPageSize();        // Размер страницы
    long getOffset();         // Смещение от начала
    Sort getSort();           // Информация о сортировке
    Pageable next();          // Следующая страница
    Pageable previousOrFirst(); // Предыдущая страница или первая
    Pageable first();         // Первая страница
    Pageable withPage(int pageNumber); // Страница с указанным номером
    boolean hasPrevious();    // Есть ли предыдущая страница?
}
```

### Создание Pageable

```java
@Service
public class PageableExamples {
    
    // Базовое создание
    public Pageable createBasicPageable(int page, int size) {
        return PageRequest.of(page, size);
    }
    
    // С сортировкой
    public Pageable createPageableWithSort(int page, int size, String sortBy) {
        return PageRequest.of(page, size, Sort.by(sortBy).ascending());
    }
    
    // С множественной сортировкой
    public Pageable createPageableWithMultipleSort(int page, int size) {
        Sort sort = Sort.by("name").ascending()
            .and(Sort.by("age").descending())
            .and(Sort.by("salary").ascending());
        return PageRequest.of(page, size, sort);
    }
    
    // С направлением сортировки
    public Pageable createPageableWithDirection(int page, int size, String sortBy, String direction) {
        Sort.Direction sortDirection = "desc".equalsIgnoreCase(direction) 
            ? Sort.Direction.DESC 
            : Sort.Direction.ASC;
        return PageRequest.of(page, size, Sort.by(sortDirection, sortBy));
    }
}
```

## Класс Sort

`Sort` определяет порядок сортировки:

```java
public class Sort implements Streamable<Sort.Order> {
    public static final Sort UNSORTED = Sort.unsorted();
    
    // Создание сортировки
    public static Sort by(String... properties);
    public static Sort by(Direction direction, String... properties);
    public static Sort by(List<Order> orders);
    
    // Методы
    public Sort and(Sort sort);
    public Sort and(Direction direction, String... properties);
    public Sort and(String... properties);
    public Sort and(Order... orders);
    
    public boolean isSorted();
    public boolean isUnsorted();
    public Iterator<Order> iterator();
    public Stream<Order> stream();
}

public static class Order {
    public static Order asc(String property);
    public static Order desc(String property);
    public static Order by(String property);
    public static Order by(Direction direction, String property);
    
    public String getProperty();
    public Direction getDirection();
    public boolean isAscending();
    public boolean isDescending();
    public Order with(Direction direction);
    public Order with(Direction direction, NullHandling nullHandling);
    public Order with(NullHandling nullHandling);
    public Order ignoreCase();
    public Order nullsFirst();
    public Order nullsLast();
    public Order nullsNative();
}

public enum Direction {
    ASC, DESC
}

public enum NullHandling {
    NATIVE, NULLS_FIRST, NULLS_LAST
}
```

### Примеры использования Sort

```java
@Service
public class SortExamples {
    
    // Простая сортировка
    public Sort createSimpleSort() {
        return Sort.by("name").ascending();
    }
    
    // Множественная сортировка
    public Sort createMultipleSort() {
        return Sort.by("department.name").ascending()
            .and(Sort.by("salary").descending())
            .and(Sort.by("age").ascending());
    }
    
    // Сортировка с обработкой null
    public Sort createSortWithNullHandling() {
        return Sort.by("name").ascending().nullsLast()
            .and(Sort.by("age").descending().nullsFirst());
    }
    
    // Сортировка без учета регистра
    public Sort createCaseInsensitiveSort() {
        return Sort.by("name").ascending().ignoreCase();
    }
    
    // Создание Pageable с сортировкой
    public Pageable createPageableWithComplexSort(int page, int size) {
        Sort sort = Sort.by("department.name").ascending().nullsLast()
            .and(Sort.by("salary").descending())
            .and(Sort.by("name").ascending().ignoreCase());
        
        return PageRequest.of(page, size, sort);
    }
}
```

## Производительность и оптимизация

### Оптимизация запросов с пагинацией

```java
@Repository
public interface OptimizedUserRepository extends JpaRepository<User, Long> {
    
    // Использование @QueryHints для оптимизации
    @QueryHints(value = {
        @QueryHint(name = HINT_FETCH_SIZE, value = "50"),
        @QueryHint(name = HINT_CACHEABLE, value = "true")
    })
    @Query("SELECT u FROM User u WHERE u.department.name = :deptName")
    Page<User> findUsersByDepartmentOptimized(
        @Param("deptName") String deptName, 
        Pageable pageable
    );
    
    // Использование @EntityGraph для избежания N+1
    @EntityGraph(attributePaths = {"department", "roles"})
    @Query("SELECT u FROM User u WHERE u.active = true")
    Page<User> findActiveUsersWithGraph(Pageable pageable);
    
    // Нативный запрос с оптимизацией
    @Query(value = "SELECT u.* FROM users u " +
                   "JOIN departments d ON u.department_id = d.id " +
                   "WHERE d.name = :deptName " +
                   "ORDER BY u.salary DESC, u.name ASC " +
                   "LIMIT :size OFFSET :offset",
           countQuery = "SELECT COUNT(*) FROM users u " +
                       "JOIN departments d ON u.department_id = d.id " +
                       "WHERE d.name = :deptName",
           nativeQuery = true)
    Page<User> findUsersByDepartmentNativeOptimized(
        @Param("deptName") String deptName,
        @Param("size") int size,
        @Param("offset") long offset
    );
}

@Service
public class OptimizedUserService {
    @Autowired
    private OptimizedUserRepository userRepository;
    
    public Page<User> getOptimizedUsers(String departmentName, int page, int size) {
        // Использование кэшируемой сортировки
        Sort cachedSort = Sort.by("salary").descending()
            .and(Sort.by("name").ascending());
        
        Pageable pageable = PageRequest.of(page, size, cachedSort);
        return userRepository.findUsersByDepartmentOptimized(departmentName, pageable);
    }
    
    public void demonstratePerformanceOptimization() {
        long startTime = System.currentTimeMillis();
        
        Page<User> page = getOptimizedUsers("IT", 0, 100);
        
        long endTime = System.currentTimeMillis();
        System.out.println("Время выполнения: " + (endTime - startTime) + "ms");
        System.out.println("Количество элементов: " + page.getTotalElements());
    }
}
```

### Мониторинг производительности

```java
@Service
public class PerformanceMonitoringService {
    
    public <T> Page<T> monitorPageQuery(Supplier<Page<T>> querySupplier, String queryName) {
        long startTime = System.currentTimeMillis();
        
        try {
            Page<T> result = querySupplier.get();
            
            long endTime = System.currentTimeMillis();
            long duration = endTime - startTime;
            
            logQueryPerformance(queryName, duration, result.getTotalElements());
            
            return result;
        } catch (Exception e) {
            logQueryError(queryName, e);
            throw e;
        }
    }
    
    private void logQueryPerformance(String queryName, long duration, long totalElements) {
        System.out.printf("Запрос '%s' выполнен за %dms, найдено %d элементов%n", 
            queryName, duration, totalElements);
    }
    
    private void logQueryError(String queryName, Exception e) {
        System.err.printf("Ошибка в запросе '%s': %s%n", queryName, e.getMessage());
    }
}
```

## Лучшие практики

### 1. Выбор между Page и Slice

```java
@Service
public class BestPracticesService {
    
    // Используйте Page когда нужна информация о общем количестве
    public Page<User> getUsersForUI(String departmentName, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("name").ascending());
        return userRepository.findByDepartmentName(departmentName, pageable);
    }
    
    // Используйте Slice для обработки больших данных
    public void processLargeDataset(String departmentName) {
        int page = 0;
        Slice<User> slice;
        
        do {
            Pageable pageable = PageRequest.of(page, 1000);
            slice = userRepository.findByDepartmentName(departmentName, pageable);
            
            processBatch(slice.getContent());
            page++;
        } while (slice.hasNext());
    }
}
```

### 2. Обработка пустых результатов

```java
@Service
public class EmptyResultsHandler {
    
    public Page<User> handleEmptyResults(String departmentName, int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        Page<User> result = userRepository.findByDepartmentName(departmentName, pageable);
        
        if (result.isEmpty()) {
            // Логика обработки пустого результата
            System.out.println("Пользователи в отделе '" + departmentName + "' не найдены");
        }
        
        return result;
    }
}
```

### 3. Валидация параметров пагинации

```java
@Service
public class PaginationValidator {
    
    public Pageable validateAndCreatePageable(int page, int size, int maxSize) {
        if (page < 0) {
            throw new IllegalArgumentException("Номер страницы не может быть отрицательным");
        }
        
        if (size <= 0) {
            throw new IllegalArgumentException("Размер страницы должен быть положительным");
        }
        
        if (size > maxSize) {
            throw new IllegalArgumentException("Размер страницы не может превышать " + maxSize);
        }
        
        return PageRequest.of(page, size);
    }
}
```

### 4. Кэширование результатов

```java
@Service
public class CachedUserService {
    
    @Cacheable(value = "users", key = "#departmentName + '_' + #page + '_' + #size")
    public Page<User> getCachedUsers(String departmentName, int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findByDepartmentName(departmentName, pageable);
    }
}
```

## Заключение

Классы `Page`, `Slice` и `Pageable` предоставляют мощные возможности для работы с пагинацией в Spring Data JPA. Выбор между `Page` и `Slice` зависит от конкретных требований:

- **Page**: когда нужна информация о общем количестве элементов и страниц
- **Slice**: для обработки больших данных и случаев, когда общее количество не важно

Правильное использование этих классов позволяет создавать эффективные и масштабируемые приложения с поддержкой пагинации.
