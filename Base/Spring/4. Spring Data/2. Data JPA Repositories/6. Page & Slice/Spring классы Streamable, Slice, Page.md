# Spring классы Streamable, Slice, Page

## Обзор

Spring Data предоставляет несколько интерфейсов для работы с коллекциями данных: `Streamable`, `Slice` и `Page`. Каждый из них предназначен для различных сценариев использования и предоставляет специфическую функциональность.

## Streamable

### Обзор
`Streamable` - это интерфейс, который расширяет `Iterable` и предоставляет дополнительные методы для работы с потоками данных.

### Основные возможности
```java
public interface Streamable<T> extends Iterable<T> {
    
    // Фильтрация
    Streamable<T> filter(Predicate<? super T> predicate);
    
    // Преобразование
    <R> Streamable<R> map(Function<? super T, ? extends R> function);
    
    // Объединение
    Streamable<T> and(Streamable<? extends T> other);
    
    // Преобразование в Stream
    Stream<T> stream();
    
    // Преобразование в List
    List<T> toList();
}
```

### Примеры использования
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Streamable<User> findByActive(boolean active);
    Streamable<User> findByRole(String role);
}

@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public Streamable<User> getActiveUsersWithRole(String role) {
        return userRepository.findByActive(true)
                           .and(userRepository.findByRole(role))
                           .filter(user -> user.getEmail().contains("@"));
    }
    
    public List<String> getActiveUserEmails() {
        return userRepository.findByActive(true)
                           .map(User::getEmail)
                           .toList();
    }
}
```

### Кастомные Streamable
```java
public class UserStreamable implements Streamable<User> {
    
    private final List<User> users;
    
    public UserStreamable(List<User> users) {
        this.users = users;
    }
    
    @Override
    public Iterator<User> iterator() {
        return users.iterator();
    }
    
    public Streamable<User> getActiveUsers() {
        return new UserStreamable(
            users.stream()
                 .filter(User::isActive)
                 .collect(Collectors.toList())
        );
    }
}
```

## Slice

### Обзор
`Slice` представляет собой "срез" данных с информацией о наличии следующей страницы, но без информации о общем количестве элементов.

### Структура интерфейса
```java
public interface Slice<T> extends Streamable<T> {
    
    // Номер текущей страницы (начиная с 0)
    int getNumber();
    
    // Размер страницы
    int getSize();
    
    // Количество элементов на текущей странице
    int getNumberOfElements();
    
    // Общее количество элементов (если доступно)
    long getTotalElements();
    
    // Общее количество страниц (если доступно)
    int getTotalPages();
    
    // Есть ли следующая страница
    boolean hasNext();
    
    // Есть ли предыдущая страница
    boolean hasPrevious();
    
    // Следующая страница
    Slice<T> nextPageable();
    
    // Предыдущая страница
    Slice<T> previousPageable();
    
    // Получить Pageable для текущей страницы
    Pageable getPageable();
}
```

### Примеры использования
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Slice<User> findByActive(boolean active, Pageable pageable);
    Slice<User> findByRole(String role, Pageable pageable);
}

@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public Slice<User> getActiveUsers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("email"));
        return userRepository.findByActive(true, pageable);
    }
    
    public void processAllActiveUsers() {
        int page = 0;
        int size = 100;
        
        Slice<User> slice;
        do {
            Pageable pageable = PageRequest.of(page, size);
            slice = userRepository.findByActive(true, pageable);
            
            // Обработка текущей страницы
            slice.forEach(this::processUser);
            
            page++;
        } while (slice.hasNext());
    }
}
```

### Преимущества Slice
```java
// Эффективная обработка больших наборов данных
public void processLargeDataset() {
    Slice<User> slice = userRepository.findByActive(true, 
        PageRequest.of(0, 1000));
    
    while (slice.hasNext()) {
        // Обработка текущей страницы
        slice.forEach(this::processUser);
        
        // Переход к следующей странице
        slice = userRepository.findByActive(true, slice.nextPageable());
    }
}
```

## Page

### Обзор
`Page` расширяет `Slice` и предоставляет дополнительную информацию о общем количестве элементов и страниц.

### Структура интерфейса
```java
public interface Page<T> extends Slice<T> {
    
    // Общее количество элементов
    long getTotalElements();
    
    // Общее количество страниц
    int getTotalPages();
    
    // Является ли это первой страницей
    boolean isFirst();
    
    // Является ли это последней страницей
    boolean isLast();
    
    // Получить Pageable для следующей страницы
    Pageable nextPageable();
    
    // Получить Pageable для предыдущей страницы
    Pageable previousPageable();
}
```

### Примеры использования
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Page<User> findByActive(boolean active, Pageable pageable);
    Page<User> findByRole(String role, Pageable pageable);
}

@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public Page<User> getActiveUsers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, 
            Sort.by("email").ascending());
        return userRepository.findByActive(true, pageable);
    }
    
    public UserPageInfo getPageInfo(int page, int size) {
        Page<User> userPage = getActiveUsers(page, size);
        
        return UserPageInfo.builder()
            .currentPage(page)
            .totalPages(userPage.getTotalPages())
            .totalElements(userPage.getTotalElements())
            .hasNext(userPage.hasNext())
            .hasPrevious(userPage.hasPrevious())
            .build();
    }
}
```

### Создание кастомных Page
```java
public class CustomPage<T> implements Page<T> {
    
    private final List<T> content;
    private final Pageable pageable;
    private final long totalElements;
    
    public CustomPage(List<T> content, Pageable pageable, long totalElements) {
        this.content = content;
        this.pageable = pageable;
        this.totalElements = totalElements;
    }
    
    @Override
    public int getNumber() {
        return pageable.getPageNumber();
    }
    
    @Override
    public int getSize() {
        return pageable.getPageSize();
    }
    
    @Override
    public int getNumberOfElements() {
        return content.size();
    }
    
    @Override
    public long getTotalElements() {
        return totalElements;
    }
    
    @Override
    public int getTotalPages() {
        return (int) Math.ceil((double) totalElements / getSize());
    }
    
    @Override
    public boolean hasNext() {
        return getNumber() < getTotalPages() - 1;
    }
    
    @Override
    public boolean hasPrevious() {
        return getNumber() > 0;
    }
    
    @Override
    public Pageable nextPageable() {
        return hasNext() ? pageable.next() : pageable;
    }
    
    @Override
    public Pageable previousPageable() {
        return hasPrevious() ? pageable.previousOrFirst() : pageable;
    }
    
    @Override
    public Pageable getPageable() {
        return pageable;
    }
    
    @Override
    public Iterator<T> iterator() {
        return content.iterator();
    }
}
```

## Сравнение типов

### Streamable vs Slice vs Page

| Характеристика | Streamable | Slice | Page |
|----------------|------------|-------|------|
| Общее количество элементов | ❌ | ❌ | ✅ |
| Общее количество страниц | ❌ | ❌ | ✅ |
| Следующая страница | ❌ | ✅ | ✅ |
| Предыдущая страница | ❌ | ✅ | ✅ |
| Фильтрация/маппинг | ✅ | ❌ | ❌ |
| Производительность | Высокая | Высокая | Средняя |

### Когда использовать

#### Streamable
```java
// Для операций с потоками данных
Streamable<User> activeUsers = userRepository.findByActive(true);
List<String> emails = activeUsers
    .filter(user -> user.getEmail().contains("@"))
    .map(User::getEmail)
    .toList();
```

#### Slice
```java
// Для больших наборов данных без необходимости знать общее количество
Slice<User> users = userRepository.findByActive(true, 
    PageRequest.of(0, 100));
while (users.hasNext()) {
    users.forEach(this::processUser);
    users = userRepository.findByActive(true, users.nextPageable());
}
```

#### Page
```java
// Для пагинации с полной информацией
Page<User> users = userRepository.findByActive(true, 
    PageRequest.of(0, 20));
System.out.println("Total users: " + users.getTotalElements());
System.out.println("Total pages: " + users.getTotalPages());
```

## Лучшие практики

### Производительность
```java
// Используйте Slice для больших наборов данных
@Query("SELECT u FROM User u WHERE u.active = ?1")
Slice<User> findByActive(boolean active, Pageable pageable);

// Используйте Page для небольших наборов с полной информацией
@Query("SELECT u FROM User u WHERE u.role = ?1")
Page<User> findByRole(String role, Pageable pageable);
```

### Обработка данных
```java
// Эффективная обработка с Slice
public void processAllUsers() {
    Slice<User> slice = userRepository.findAll(PageRequest.of(0, 1000));
    
    while (slice.hasNext()) {
        slice.forEach(this::processUser);
        slice = userRepository.findAll(slice.nextPageable());
    }
}
```

### Кастомные операции
```java
// Создание кастомных Streamable
public Streamable<User> getUsersWithCustomFilter() {
    return userRepository.findAll()
        .filter(user -> user.isActive())
        .filter(user -> user.getEmail().contains("@company.com"));
}
```

## Лучшие практики

1. **Используйте Streamable** для операций с потоками данных
2. **Применяйте Slice** для больших наборов данных без необходимости знать общее количество
3. **Используйте Page** для пагинации с полной информацией
4. **Оптимизируйте запросы** для пагинации
5. **Избегайте N+1 проблем** при работе с связанными сущностями
6. **Тестируйте производительность** для больших наборов данных 