# Наследники Repository

## Обзор

В Spring Data JPA существует иерархия интерфейсов, наследующих от базового `Repository`. Каждый уровень иерархии добавляет дополнительную функциональность, предоставляя более богатый API для работы с данными. Понимание этой иерархии критически важно для правильного выбора базового интерфейса для репозиториев.

## Иерархия интерфейсов

### Полная иерархия
```
Repository<T, ID> (базовый маркерный интерфейс)
    ↓
CrudRepository<T, ID> (добавляет CRUD операции)
    ↓
PagingAndSortingRepository<T, ID> (добавляет пагинацию и сортировку)
    ↓
JpaRepository<T, ID> (специфичный для JPA с дополнительными методами)
    ↓
CustomRepository<T, ID> (кастомные расширения)
```

## Детальный анализ каждого уровня

### Repository<T, ID> - Базовый маркерный интерфейс

#### Определение
```java
@Indexed
public interface Repository<T, ID> {
    // Пустой интерфейс - маркер
}
```

#### Характеристики
- **Маркерный интерфейс** - не содержит методов
- **Минимальная функциональность** - только кастомные методы
- **Максимальная гибкость** - полный контроль над API
- **Производительность** - минимальные накладные расходы

#### Использование
```java
@Repository
public interface UserRepository extends Repository<User, Long> {
    // Только кастомные методы - базовые CRUD методы недоступны
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
    long countByActiveTrue();
}
```

#### Преимущества
- Минимальные накладные расходы
- Полный контроль над API
- Подходит для специфичных случаев
- Не навязывает стандартные методы

#### Недостатки
- Нет базовых CRUD операций
- Требует реализации всех необходимых методов
- Больше кода для написания

### CrudRepository<T, ID> - Базовые CRUD операции

#### Определение
```java
@NoRepositoryBean
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    
    // Методы сохранения
    <S extends T> S save(S entity);
    <S extends T> Iterable<S> saveAll(Iterable<S> entities);
    
    // Методы поиска
    Optional<T> findById(ID id);
    Iterable<T> findAll();
    Iterable<T> findAllById(Iterable<ID> ids);
    boolean existsById(ID id);
    long count();
    
    // Методы удаления
    void deleteById(ID id);
    void delete(T entity);
    void deleteAllById(Iterable<? extends ID> ids);
    void deleteAll(Iterable<? extends T> entities);
    void deleteAll();
}
```

#### Характеристики
- **Базовые CRUD операции** - save, find, delete, count
- **Типобезопасность** - работает с конкретными типами
- **Простота использования** - стандартные операции
- **Производительность** - оптимизированные реализации

#### Использование
```java
@Repository
public interface UserRepository extends CrudRepository<User, Long> {
    // Доступны все CRUD методы + кастомные
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
    List<User> findByActiveTrue();
}
```

#### Методы сохранения
```java
// Сохранение одной сущности
User savedUser = userRepository.save(user);

// Массовое сохранение
List<User> users = Arrays.asList(user1, user2, user3);
Iterable<User> savedUsers = userRepository.saveAll(users);
```

#### Методы поиска
```java
// Поиск по ID
Optional<User> user = userRepository.findById(1L);

// Получение всех сущностей
Iterable<User> allUsers = userRepository.findAll();

// Поиск по нескольким ID
Iterable<User> users = userRepository.findAllById(Arrays.asList(1L, 2L, 3L));

// Проверка существования
boolean exists = userRepository.existsById(1L);

// Подсчет количества
long count = userRepository.count();
```

#### Методы удаления
```java
// Удаление по ID
userRepository.deleteById(1L);

// Удаление сущности
userRepository.delete(user);

// Массовое удаление по ID
userRepository.deleteAllById(Arrays.asList(1L, 2L, 3L));

// Массовое удаление сущностей
userRepository.deleteAll(Arrays.asList(user1, user2));

// Удаление всех
userRepository.deleteAll();
```

### PagingAndSortingRepository<T, ID> - Пагинация и сортировка

#### Определение
```java
@NoRepositoryBean
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {
    
    // Методы с сортировкой
    Iterable<T> findAll(Sort sort);
    
    // Методы с пагинацией
    Page<T> findAll(Pageable pageable);
}
```

#### Характеристики
- **Наследует от CrudRepository** - все CRUD операции доступны
- **Добавляет сортировку** - поддержка Sort
- **Добавляет пагинацию** - поддержка Pageable и Page
- **Производительность** - оптимизированные запросы

#### Использование
```java
@Repository
public interface UserRepository extends PagingAndSortingRepository<User, Long> {
    // Доступны все CRUD методы + пагинация/сортировка + кастомные
    List<User> findByEmail(String email);
    Page<User> findByActiveTrue(Pageable pageable);
    List<User> findByRoleOrderByCreatedAtDesc(Role role);
}
```

#### Сортировка
```java
// Сортировка по одному полю
Sort sort = Sort.by("email").ascending();
Iterable<User> users = userRepository.findAll(sort);

// Сортировка по нескольким полям
Sort sort = Sort.by("lastName").ascending()
    .and(Sort.by("firstName").ascending());
Iterable<User> users = userRepository.findAll(sort);

// Сортировка с направлением
Sort sort = Sort.by(Sort.Direction.DESC, "createdAt");
Iterable<User> users = userRepository.findAll(sort);
```

#### Пагинация
```java
// Простая пагинация
Pageable pageable = PageRequest.of(0, 10); // первая страница, 10 элементов
Page<User> page = userRepository.findAll(pageable);

// Пагинация с сортировкой
Pageable pageable = PageRequest.of(0, 10, Sort.by("email").ascending());
Page<User> page = userRepository.findAll(pageable);

// Навигация по страницам
Page<User> firstPage = userRepository.findAll(PageRequest.of(0, 10));
Page<User> secondPage = userRepository.findAll(PageRequest.of(1, 10));
```

### JpaRepository<T, ID> - Специфичный для JPA

#### Определение
```java
@NoRepositoryBean
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>, JpaSpecificationExecutor<T> {
    
    // Дополнительные методы поиска
    List<T> findAll();
    List<T> findAll(Sort sort);
    List<T> findAllById(Iterable<ID> ids);
    
    // Методы сохранения с возвратом результата
    <S extends T> List<S> saveAllAndFlush(Iterable<S> entities);
    <S extends T> S saveAndFlush(S entity);
    
    // Методы удаления
    void deleteAllInBatch(Iterable<T> entities);
    void deleteAllByIdInBatch(Iterable<ID> ids);
    
    // Методы flush
    void flush();
    
    // Методы getOne (deprecated) и getById
    T getById(ID id);
    T getReferenceById(ID id);
}
```

#### Характеристики
- **Наследует от PagingAndSortingRepository** - все предыдущие возможности
- **Специфичный для JPA** - дополнительные JPA методы
- **Batch операции** - оптимизированные массовые операции
- **Flush методы** - контроль над flush операциями

#### Использование
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // Доступны все методы JpaRepository + кастомные
    List<User> findByEmail(String email);
    Page<User> findByActiveTrue(Pageable pageable);
    List<User> findByRoleOrderByCreatedAtDesc(Role role);
}
```

#### Дополнительные методы поиска
```java
// Получение всех с возвратом List
List<User> allUsers = userRepository.findAll();

// Получение всех с сортировкой
List<User> sortedUsers = userRepository.findAll(Sort.by("email").ascending());

// Получение по нескольким ID
List<User> users = userRepository.findAllById(Arrays.asList(1L, 2L, 3L));
```

#### Методы сохранения с flush
```java
// Сохранение с немедленным flush
User savedUser = userRepository.saveAndFlush(user);

// Массовое сохранение с flush
List<User> users = Arrays.asList(user1, user2, user3);
List<User> savedUsers = userRepository.saveAllAndFlush(users);
```

#### Batch операции удаления
```java
// Batch удаление сущностей
userRepository.deleteAllInBatch(Arrays.asList(user1, user2));

// Batch удаление по ID
userRepository.deleteAllByIdInBatch(Arrays.asList(1L, 2L, 3L));
```

#### Методы flush
```java
// Принудительный flush
userRepository.flush();

// Сохранение с flush
User user = new User();
user.setEmail("test@example.com");
User savedUser = userRepository.saveAndFlush(user);
```

## Кастомные расширения

### Создание кастомного базового интерфейса
```java
@NoRepositoryBean
public interface CustomRepository<T, ID> extends JpaRepository<T, ID> {
    
    // Дополнительные методы
    List<T> findByCreatedDateBetween(LocalDateTime start, LocalDateTime end);
    
    // Методы с аудитом
    <S extends T> S saveWithAudit(S entity);
    
    // Методы с логированием
    <S extends T> S saveWithLogging(S entity);
    
    // Методы с валидацией
    <S extends T> S saveWithValidation(S entity);
}
```

### Реализация кастомного интерфейса
```java
public class CustomRepositoryImpl<T, ID> extends SimpleJpaRepository<T, ID> implements CustomRepository<T, ID> {
    
    private final EntityManager entityManager;
    private final JpaEntityInformation<T, ?> entityInformation;
    
    public CustomRepositoryImpl(JpaEntityInformation<T, ?> entityInformation, EntityManager entityManager) {
        super(entityInformation, entityManager);
        this.entityManager = entityManager;
        this.entityInformation = entityInformation;
    }
    
    @Override
    public List<T> findByCreatedDateBetween(LocalDateTime start, LocalDateTime end) {
        String jpql = "SELECT e FROM " + getEntityName() + " e WHERE e.createdDate BETWEEN :start AND :end";
        TypedQuery<T> query = entityManager.createQuery(jpql, getDomainClass());
        query.setParameter("start", start);
        query.setParameter("end", end);
        return query.getResultList();
    }
    
    @Override
    public <S extends T> S saveWithAudit(S entity) {
        if (entityInformation.isNew(entity)) {
            if (entity instanceof Auditable) {
                ((Auditable) entity).setCreatedAt(LocalDateTime.now());
            }
        }
        if (entity instanceof Auditable) {
            ((Auditable) entity).setLastModified(LocalDateTime.now());
        }
        return super.save(entity);
    }
    
    @Override
    public <S extends T> S saveWithLogging(S entity) {
        System.out.println("Saving entity: " + entity);
        S savedEntity = super.save(entity);
        System.out.println("Saved entity: " + savedEntity);
        return savedEntity;
    }
    
    @Override
    public <S extends T> S saveWithValidation(S entity) {
        // Кастомная валидация
        validateEntity(entity);
        return super.save(entity);
    }
    
    private void validateEntity(T entity) {
        // Логика валидации
        if (entity instanceof Validatable) {
            ((Validatable) entity).validate();
        }
    }
}
```

### Использование кастомного интерфейса
```java
@Repository
public interface UserRepository extends CustomRepository<User, Long> {
    // Доступны все методы CustomRepository + кастомные
    List<User> findByEmail(String email);
    Page<User> findByActiveTrue(Pageable pageable);
    
    // Использование кастомных методов
    List<User> findUsersCreatedBetween(LocalDateTime start, LocalDateTime end);
}
```

## Выбор правильного базового интерфейса

### Критерии выбора

#### 1. Функциональные требования
```java
// Минимальная функциональность - только кастомные методы
public interface UserRepository extends Repository<User, Long> {
    List<User> findByEmail(String email);
}

// Базовые CRUD операции
public interface UserRepository extends CrudRepository<User, Long> {
    List<User> findByEmail(String email);
}

// CRUD + пагинация/сортировка
public interface UserRepository extends PagingAndSortingRepository<User, Long> {
    Page<User> findByActiveTrue(Pageable pageable);
}

// Полная JPA функциональность
public interface UserRepository extends JpaRepository<User, Long> {
    // Все возможности JPA
}
```

#### 2. Производительность
```java
// Минимальные накладные расходы
public interface UserRepository extends Repository<User, Long> {
    // Только необходимые методы
}

// Оптимизированные batch операции
public interface UserRepository extends JpaRepository<User, Long> {
    // Batch операции доступны
}
```

#### 3. Специфичные требования
```java
// Кастомная логика
public interface UserRepository extends CustomRepository<User, Long> {
    // Кастомные методы + стандартные
}
```

### Рекомендации по выбору

#### Для простых случаев
```java
// Используйте CrudRepository для базовых CRUD операций
public interface UserRepository extends CrudRepository<User, Long> {
    // Достаточно для большинства случаев
}
```

#### Для сложных случаев
```java
// Используйте JpaRepository для полной функциональности
public interface UserRepository extends JpaRepository<User, Long> {
    // Все возможности JPA
}
```

#### Для специфичных случаев
```java
// Используйте кастомные интерфейсы для специфичной логики
public interface UserRepository extends CustomRepository<User, Long> {
    // Кастомная логика + стандартные возможности
}
```

## Лучшие практики

### 1. Выбор правильного уровня
```java
// Правильно - выбирайте минимально необходимый уровень
public interface UserRepository extends CrudRepository<User, Long> {
    // Только необходимые методы
}

// Неправильно - не используйте JpaRepository без необходимости
public interface UserRepository extends JpaRepository<User, Long> {
    // Избыточная функциональность
}
```

### 2. Кастомные расширения
```java
// Создавайте кастомные интерфейсы для повторяющейся логики
@NoRepositoryBean
public interface AuditableRepository<T, ID> extends JpaRepository<T, ID> {
    <S extends T> S saveWithAudit(S entity);
}

// Используйте кастомные интерфейсы
public interface UserRepository extends AuditableRepository<User, Long> {
    // Кастомная логика + стандартные возможности
}
```

### 3. Производительность
```java
// Используйте batch операции для массовых операций
@Transactional
public void saveUsersInBatch(List<User> users) {
    userRepository.saveAll(users); // Batch операция
}

// Используйте flush для контроля над транзакциями
public void saveUserWithFlush(User user) {
    userRepository.saveAndFlush(user); // Немедленный flush
}
```

### 4. Типизация
```java
// Всегда указывайте типы
public interface UserRepository extends JpaRepository<User, Long> {
    // Правильная типизация
}

// Избегайте сырых типов
public interface UserRepository extends JpaRepository {
    // Неправильно - нет типизации
}
```

## Заключение

Понимание иерархии наследников `Repository` в Spring Data JPA критически важно для:

- **Правильного выбора базового интерфейса** - выбор минимально необходимого уровня
- **Оптимизации производительности** - использование подходящих методов
- **Создания кастомных расширений** - добавление специфичной логики
- **Поддержки кода** - понимание доступных возможностей

Ключевые принципы:
- Выбирайте минимально необходимый уровень иерархии
- Используйте кастомные интерфейсы для повторяющейся логики
- Оптимизируйте производительность через batch операции
- Следуйте принципам типизации и лучших практик
