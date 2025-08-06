# DAO & Repository. CRUD

## Использование Hibernate на разных слоях приложения

### Архитектурные слои

В современном Java-приложении с Hibernate выделяют следующие слои:

1. **Presentation Layer** - контроллеры, REST API
2. **Service Layer** - бизнес-логика
3. **Repository/DAO Layer** - работа с данными
4. **Entity Layer** - модели данных

### Роль Repository/DAO слоя

```java
// Пример интерфейса Repository
public interface UserRepository {
    User save(User user);
    User findById(Long id);
    List<User> findAll();
    User update(User user);
    void delete(Long id);
}
```

## Создание CRUD DAO

### Базовый интерфейс DAO

```java
public interface BaseDao<T, ID> {
    T save(T entity);
    T findById(ID id);
    List<T> findAll();
    T update(T entity);
    void delete(ID id);
    void delete(T entity);
}
```

### Абстрактная реализация

```java
public abstract class AbstractDao<T, ID> implements BaseDao<T, ID> {
    
    protected final SessionFactory sessionFactory;
    protected final Class<T> entityClass;
    
    public AbstractDao(SessionFactory sessionFactory, Class<T> entityClass) {
        this.sessionFactory = sessionFactory;
        this.entityClass = entityClass;
    }
    
    protected Session getCurrentSession() {
        return sessionFactory.getCurrentSession();
    }
}
```

## Реализация метода save

### Базовая реализация

```java
@Override
public T save(T entity) {
    Session session = getCurrentSession();
    session.save(entity);
    return entity;
}
```

### С проверкой существования

```java
@Override
public T save(T entity) {
    Session session = getCurrentSession();
    
    if (entity.getId() == null) {
        session.save(entity);
    } else {
        session.update(entity);
    }
    
    return entity;
}
```

### С использованием merge

```java
@Override
public T save(T entity) {
    Session session = getCurrentSession();
    return (T) session.merge(entity);
}
```

## Реализация метода delete

### Удаление по ID

```java
@Override
public void delete(ID id) {
    Session session = getCurrentSession();
    T entity = findById(id);
    if (entity != null) {
        session.delete(entity);
    }
}
```

### Прямое удаление

```java
@Override
public void delete(T entity) {
    Session session = getCurrentSession();
    session.delete(entity);
}
```

### Удаление с проверкой

```java
@Override
public boolean delete(ID id) {
    Session session = getCurrentSession();
    T entity = findById(id);
    if (entity != null) {
        session.delete(entity);
        return true;
    }
    return false;
}
```

## Реализация метода update

### Простое обновление

```java
@Override
public T update(T entity) {
    Session session = getCurrentSession();
    session.update(entity);
    return entity;
}
```

### С проверкой существования

```java
@Override
public T update(T entity) {
    Session session = getCurrentSession();
    
    if (entity.getId() == null) {
        throw new IllegalArgumentException("Entity must have an ID for update");
    }
    
    T existingEntity = findById(entity.getId());
    if (existingEntity == null) {
        throw new EntityNotFoundException("Entity not found for update");
    }
    
    session.update(entity);
    return entity;
}
```

### С использованием merge

```java
@Override
public T update(T entity) {
    Session session = getCurrentSession();
    return (T) session.merge(entity);
}
```

## Реализация метода findById

### Базовая реализация

```java
@Override
public T findById(ID id) {
    Session session = getCurrentSession();
    return session.get(entityClass, id);
}
```

### С обработкой исключений

```java
@Override
public T findById(ID id) {
    if (id == null) {
        throw new IllegalArgumentException("ID cannot be null");
    }
    
    Session session = getCurrentSession();
    T entity = session.get(entityClass, id);
    
    if (entity == null) {
        throw new EntityNotFoundException("Entity with ID " + id + " not found");
    }
    
    return entity;
}
```

### С использованием load

```java
@Override
public T findById(ID id) {
    Session session = getCurrentSession();
    return session.load(entityClass, id);
}
```

## Реализация метода findAll

### Простая реализация

```java
@Override
public List<T> findAll() {
    Session session = getCurrentSession();
    return session.createQuery("from " + entityClass.getSimpleName(), entityClass)
                 .getResultList();
}
```

### С пагинацией

```java
public List<T> findAll(int page, int size) {
    Session session = getCurrentSession();
    return session.createQuery("from " + entityClass.getSimpleName(), entityClass)
                 .setFirstResult(page * size)
                 .setMaxResults(size)
                 .getResultList();
}
```

### С сортировкой

```java
public List<T> findAll(String sortBy, String direction) {
    Session session = getCurrentSession();
    String hql = "from " + entityClass.getSimpleName() + 
                 " order by " + sortBy + " " + direction;
    return session.createQuery(hql, entityClass).getResultList();
}
```

## Создание BaseDao

### Универсальный BaseDao

```java
public abstract class BaseDao<T, ID> {
    
    protected final SessionFactory sessionFactory;
    protected final Class<T> entityClass;
    
    public BaseDao(SessionFactory sessionFactory, Class<T> entityClass) {
        this.sessionFactory = sessionFactory;
        this.entityClass = entityClass;
    }
    
    protected Session getCurrentSession() {
        return sessionFactory.getCurrentSession();
    }
    
    // CRUD методы
    public T save(T entity) {
        Session session = getCurrentSession();
        return (T) session.merge(entity);
    }
    
    public T findById(ID id) {
        Session session = getCurrentSession();
        return session.get(entityClass, id);
    }
    
    public List<T> findAll() {
        Session session = getCurrentSession();
        return session.createQuery("from " + entityClass.getSimpleName(), entityClass)
                     .getResultList();
    }
    
    public void delete(ID id) {
        Session session = getCurrentSession();
        T entity = findById(id);
        if (entity != null) {
            session.delete(entity);
        }
    }
    
    public void delete(T entity) {
        Session session = getCurrentSession();
        session.delete(entity);
    }
}
```

### Специализированный BaseDao

```java
public abstract class BaseDao<T, ID> {
    
    protected final SessionFactory sessionFactory;
    protected final Class<T> entityClass;
    
    public BaseDao(SessionFactory sessionFactory, Class<T> entityClass) {
        this.sessionFactory = sessionFactory;
        this.entityClass = entityClass;
    }
    
    // Дополнительные методы
    public long count() {
        Session session = getCurrentSession();
        return (Long) session.createQuery("select count(*) from " + entityClass.getSimpleName())
                           .uniqueResult();
    }
    
    public boolean exists(ID id) {
        return findById(id) != null;
    }
    
    public List<T> findByIds(List<ID> ids) {
        if (ids == null || ids.isEmpty()) {
            return new ArrayList<>();
        }
        
        Session session = getCurrentSession();
        return session.createQuery("from " + entityClass.getSimpleName() + 
                                 " where id in (:ids)", entityClass)
                     .setParameterList("ids", ids)
                     .getResultList();
    }
}
```

## Слой Repository

### Интерфейс Repository

```java
public interface Repository<T, ID> {
    T save(T entity);
    T findById(ID id);
    List<T> findAll();
    void delete(ID id);
    void delete(T entity);
    boolean exists(ID id);
    long count();
}
```

### Абстрактная реализация Repository

```java
public abstract class AbstractRepository<T, ID> implements Repository<T, ID> {
    
    protected final SessionFactory sessionFactory;
    protected final Class<T> entityClass;
    
    public AbstractRepository(SessionFactory sessionFactory, Class<T> entityClass) {
        this.sessionFactory = sessionFactory;
        this.entityClass = entityClass;
    }
    
    protected Session getCurrentSession() {
        return sessionFactory.getCurrentSession();
    }
    
    @Override
    public T save(T entity) {
        Session session = getCurrentSession();
        return (T) session.merge(entity);
    }
    
    @Override
    public T findById(ID id) {
        Session session = getCurrentSession();
        return session.get(entityClass, id);
    }
    
    @Override
    public List<T> findAll() {
        Session session = getCurrentSession();
        return session.createQuery("from " + entityClass.getSimpleName(), entityClass)
                     .getResultList();
    }
    
    @Override
    public void delete(ID id) {
        Session session = getCurrentSession();
        T entity = findById(id);
        if (entity != null) {
            session.delete(entity);
        }
    }
    
    @Override
    public void delete(T entity) {
        Session session = getCurrentSession();
        session.delete(entity);
    }
    
    @Override
    public boolean exists(ID id) {
        return findById(id) != null;
    }
    
    @Override
    public long count() {
        Session session = getCurrentSession();
        return (Long) session.createQuery("select count(*) from " + entityClass.getSimpleName())
                           .uniqueResult();
    }
}
```

### Специализированный Repository

```java
public interface UserRepository extends Repository<User, Long> {
    User findByEmail(String email);
    List<User> findByRole(Role role);
    List<User> findActiveUsers();
}
```

```java
public class UserRepositoryImpl extends AbstractRepository<User, Long> 
                               implements UserRepository {
    
    public UserRepositoryImpl(SessionFactory sessionFactory) {
        super(sessionFactory, User.class);
    }
    
    @Override
    public User findByEmail(String email) {
        Session session = getCurrentSession();
        return session.createQuery("from User where email = :email", User.class)
                     .setParameter("email", email)
                     .uniqueResult();
    }
    
    @Override
    public List<User> findByRole(Role role) {
        Session session = getCurrentSession();
        return session.createQuery("from User where role = :role", User.class)
                     .setParameter("role", role)
                     .getResultList();
    }
    
    @Override
    public List<User> findActiveUsers() {
        Session session = getCurrentSession();
        return session.createQuery("from User where active = true", User.class)
                     .getResultList();
    }
}
```

### Использование в Service слое

```java
@Service
@Transactional
public class UserService {
    
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    public User getUserById(Long id) {
        return userRepository.findById(id);
    }
    
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
    
    public void deleteUser(Long id) {
        userRepository.delete(id);
    }
    
    public User getUserByEmail(String email) {
        return userRepository.findByEmail(email);
    }
}
```

## Лучшие практики

### 1. Использование транзакций

```java
@Repository
@Transactional
public class UserRepositoryImpl extends AbstractRepository<User, Long> 
                               implements UserRepository {
    // методы автоматически выполняются в транзакции
}
```

### 2. Обработка исключений

```java
@Override
public T findById(ID id) {
    try {
        Session session = getCurrentSession();
        return session.get(entityClass, id);
    } catch (Exception e) {
        throw new DataAccessException("Error finding entity by ID: " + id, e);
    }
}
```

### 3. Валидация параметров

```java
@Override
public T save(T entity) {
    if (entity == null) {
        throw new IllegalArgumentException("Entity cannot be null");
    }
    
    Session session = getCurrentSession();
    return (T) session.merge(entity);
}
```

### 4. Логирование

```java
@Slf4j
public abstract class AbstractRepository<T, ID> implements Repository<T, ID> {
    
    @Override
    public T save(T entity) {
        log.debug("Saving entity: {}", entity);
        Session session = getCurrentSession();
        T savedEntity = (T) session.merge(entity);
        log.debug("Entity saved: {}", savedEntity);
        return savedEntity;
    }
}
```

### 5. Кэширование

```java
@Override
@Cacheable("users")
public User findById(Long id) {
    Session session = getCurrentSession();
    return session.get(User.class, id);
}
```

## Заключение

Паттерн DAO/Repository обеспечивает:

- **Разделение ответственности** между слоями приложения
- **Абстракцию** работы с данными
- **Переиспользование** кода через базовые классы
- **Тестируемость** через интерфейсы
- **Гибкость** в реализации различных стратегий доступа к данным

Правильная реализация CRUD операций в DAO/Repository слое является основой для создания масштабируемых и поддерживаемых приложений. 