# DAO & Repository. CurrentSessionContext

## Варианты решения LazyInitializationException в Repository

### Проблема LazyInitializationException

`LazyInitializationException` возникает, когда пытаемся обратиться к лениво загружаемым ассоциациям вне сессии Hibernate:

```java
// Проблемный код
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User getUserWithOrders(Long userId) {
        User user = userRepository.findById(userId); // сессия закрывается
        // LazyInitializationException при обращении к orders
        return user.getOrders(); // ❌ Ошибка!
    }
}
```

### Решения проблемы

#### 1. Использование @Transactional

```java
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User getUserWithOrders(Long userId) {
        User user = userRepository.findById(userId);
        // Сессия остается открытой благодаря @Transactional
        return user.getOrders(); // ✅ Работает
    }
}
```

#### 2. Использование Hibernate.initialize()

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User getUserWithOrders(Long userId) {
        User user = userRepository.findById(userId);
        Hibernate.initialize(user.getOrders()); // Принудительная загрузка
        return user;
    }
}
```

#### 3. Использование JOIN FETCH

```java
@Repository
public class UserRepositoryImpl extends AbstractRepository<User, Long> {
    
    public User findByIdWithOrders(Long id) {
        Session session = getCurrentSession();
        return session.createQuery(
            "from User u left join fetch u.orders where u.id = :id", User.class)
            .setParameter("id", id)
            .uniqueResult();
    }
}
```

#### 4. Использование CurrentSessionContext

```java
@Repository
public class UserRepositoryImpl extends AbstractRepository<User, Long> {
    
    public User findByIdWithOrders(Long id) {
        Session session = getCurrentSession();
        User user = session.get(User.class, id);
        
        // Принудительная загрузка ассоциаций
        if (user != null) {
            Hibernate.initialize(user.getOrders());
        }
        
        return user;
    }
}
```

## Класс CurrentSessionContext

### Назначение CurrentSessionContext

`CurrentSessionContext` - это интерфейс Hibernate, который определяет стратегию получения текущей сессии. Основные реализации:

- `ThreadLocalSessionContext` - сессия привязана к потоку
- `ManagedSessionContext` - сессия управляется внешним контекстом
- `JtaSessionContext` - сессия интегрирована с JTA

### Создание кастомного CurrentSessionContext

```java
public class CustomSessionContext implements CurrentSessionContext {
    
    private final SessionFactory sessionFactory;
    private final ThreadLocal<Session> sessionHolder = new ThreadLocal<>();
    
    public CustomSessionContext(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }
    
    @Override
    public Session currentSession() throws HibernateException {
        Session session = sessionHolder.get();
        
        if (session == null || !session.isOpen()) {
            session = sessionFactory.openSession();
            sessionHolder.set(session);
        }
        
        return session;
    }
    
    public void closeSession() {
        Session session = sessionHolder.get();
        if (session != null && session.isOpen()) {
            session.close();
            sessionHolder.remove();
        }
    }
}
```

### Использование ThreadLocalSessionContext

```java
@Configuration
public class HibernateConfig {
    
    @Bean
    public SessionFactory sessionFactory() {
        Configuration configuration = new Configuration();
        configuration.configure();
        
        // Настройка CurrentSessionContext
        configuration.setProperty("hibernate.current_session_context_class", 
                               "thread");
        
        return configuration.buildSessionFactory();
    }
}
```

### Создание SessionManager

```java
@Component
public class SessionManager {
    
    private final SessionFactory sessionFactory;
    private final ThreadLocal<Session> sessionHolder = new ThreadLocal<>();
    
    public SessionManager(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }
    
    public Session getCurrentSession() {
        Session session = sessionHolder.get();
        
        if (session == null || !session.isOpen()) {
            session = sessionFactory.openSession();
            sessionHolder.set(session);
        }
        
        return session;
    }
    
    public void closeSession() {
        Session session = sessionHolder.get();
        if (session != null && session.isOpen()) {
            session.close();
            sessionHolder.remove();
        }
    }
    
    public void beginTransaction() {
        Session session = getCurrentSession();
        session.beginTransaction();
    }
    
    public void commitTransaction() {
        Session session = getCurrentSession();
        if (session.getTransaction().isActive()) {
            session.getTransaction().commit();
        }
    }
    
    public void rollbackTransaction() {
        Session session = getCurrentSession();
        if (session.getTransaction().isActive()) {
            session.getTransaction().rollback();
        }
    }
}
```

## Настройка CurrentSessionContext в Hibernate xml config

### hibernate.cfg.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">

<hibernate-configuration>
    <session-factory>
        <!-- Database connection settings -->
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/testdb</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">password</property>
        
        <!-- Hibernate properties -->
        <property name="hibernate.dialect">org.hibernate.dialect.MySQL8Dialect</property>
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>
        
        <!-- CurrentSessionContext configuration -->
        <property name="hibernate.current_session_context_class">thread</property>
        
        <!-- Entity mappings -->
        <mapping class="com.example.entity.User"/>
        <mapping class="com.example.entity.Order"/>
    </session-factory>
</hibernate-configuration>
```

### application.properties

```properties
# Hibernate CurrentSessionContext
hibernate.current_session_context_class=thread
```

### application.yml

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        current_session_context_class: thread
        dialect: org.hibernate.dialect.MySQL8Dialect
        show_sql: true
        format_sql: true
```

## Заменяем SessionFactory на EntityManager в Repository

### Использование EntityManager

```java
@Repository
@Transactional
public class UserRepositoryImpl implements UserRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public User findById(Long id) {
        return entityManager.find(User.class, id);
    }
    
    @Override
    public User save(User user) {
        if (user.getId() == null) {
            entityManager.persist(user);
            return user;
        } else {
            return entityManager.merge(user);
        }
    }
    
    @Override
    public void delete(Long id) {
        User user = findById(id);
        if (user != null) {
            entityManager.remove(user);
        }
    }
    
    @Override
    public List<User> findAll() {
        return entityManager.createQuery("from User", User.class)
                          .getResultList();
    }
}
```

### Абстрактный Repository с EntityManager

```java
public abstract class AbstractRepository<T, ID> {
    
    @PersistenceContext
    protected EntityManager entityManager;
    
    protected final Class<T> entityClass;
    
    public AbstractRepository(Class<T> entityClass) {
        this.entityClass = entityClass;
    }
    
    public T findById(ID id) {
        return entityManager.find(entityClass, id);
    }
    
    public T save(T entity) {
        if (entity.getId() == null) {
            entityManager.persist(entity);
            return entity;
        } else {
            return entityManager.merge(entity);
        }
    }
    
    public void delete(ID id) {
        T entity = findById(id);
        if (entity != null) {
            entityManager.remove(entity);
        }
    }
    
    public List<T> findAll() {
        return entityManager.createQuery("from " + entityClass.getSimpleName(), entityClass)
                          .getResultList();
    }
    
    public long count() {
        return (Long) entityManager.createQuery("select count(*) from " + entityClass.getSimpleName())
                                 .getSingleResult();
    }
}
```

### Специализированный Repository

```java
@Repository
@Transactional
public class UserRepositoryImpl extends AbstractRepository<User, Long> 
                               implements UserRepository {
    
    public UserRepositoryImpl() {
        super(User.class);
    }
    
    @Override
    public User findByEmail(String email) {
        return entityManager.createQuery(
            "from User where email = :email", User.class)
            .setParameter("email", email)
            .getResultStream()
            .findFirst()
            .orElse(null);
    }
    
    @Override
    public List<User> findByRole(Role role) {
        return entityManager.createQuery(
            "from User where role = :role", User.class)
            .setParameter("role", role)
            .getResultList();
    }
    
    @Override
    public User findByIdWithOrders(Long id) {
        return entityManager.createQuery(
            "from User u left join fetch u.orders where u.id = :id", User.class)
            .setParameter("id", id)
            .getResultStream()
            .findFirst()
            .orElse(null);
    }
}
```

## Создание EntityManager proxy

### Proxy для EntityManager

```java
@Component
public class EntityManagerProxy {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    private final ThreadLocal<EntityManager> threadLocalEntityManager = new ThreadLocal<>();
    
    public EntityManager getEntityManager() {
        EntityManager em = threadLocalEntityManager.get();
        if (em == null || !em.isOpen()) {
            em = entityManager.getEntityManagerFactory().createEntityManager();
            threadLocalEntityManager.set(em);
        }
        return em;
    }
    
    public void closeEntityManager() {
        EntityManager em = threadLocalEntityManager.get();
        if (em != null && em.isOpen()) {
            em.close();
            threadLocalEntityManager.remove();
        }
    }
    
    public void beginTransaction() {
        EntityManager em = getEntityManager();
        if (!em.getTransaction().isActive()) {
            em.getTransaction().begin();
        }
    }
    
    public void commitTransaction() {
        EntityManager em = getEntityManager();
        if (em.getTransaction().isActive()) {
            em.getTransaction().commit();
        }
    }
    
    public void rollbackTransaction() {
        EntityManager em = getEntityManager();
        if (em.getTransaction().isActive()) {
            em.getTransaction().rollback();
        }
    }
}
```

### Использование EntityManagerProxy в Repository

```java
@Repository
public class UserRepositoryImpl implements UserRepository {
    
    private final EntityManagerProxy entityManagerProxy;
    
    public UserRepositoryImpl(EntityManagerProxy entityManagerProxy) {
        this.entityManagerProxy = entityManagerProxy;
    }
    
    @Override
    public User findById(Long id) {
        EntityManager em = entityManagerProxy.getEntityManager();
        return em.find(User.class, id);
    }
    
    @Override
    public User save(User user) {
        EntityManager em = entityManagerProxy.getEntityManager();
        
        if (user.getId() == null) {
            em.persist(user);
            return user;
        } else {
            return em.merge(user);
        }
    }
    
    @Override
    public void delete(Long id) {
        EntityManager em = entityManagerProxy.getEntityManager();
        User user = findById(id);
        if (user != null) {
            em.remove(user);
        }
    }
    
    @Override
    public List<User> findAll() {
        EntityManager em = entityManagerProxy.getEntityManager();
        return em.createQuery("from User", User.class).getResultList();
    }
}
```

### Конфигурация с EntityManagerProxy

```java
@Configuration
@EnableTransactionManagement
public class RepositoryConfig {
    
    @Bean
    public EntityManagerProxy entityManagerProxy() {
        return new EntityManagerProxy();
    }
    
    @Bean
    public UserRepository userRepository(EntityManagerProxy entityManagerProxy) {
        return new UserRepositoryImpl(entityManagerProxy);
    }
}
```

## Лучшие практики

### 1. Использование @Transactional

```java
@Service
@Transactional
public class UserService {
    
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User getUserWithOrders(Long userId) {
        return userRepository.findByIdWithOrders(userId);
    }
}
```

### 2. Обработка исключений

```java
@Repository
public class UserRepositoryImpl implements UserRepository {
    
    @Override
    public User findById(Long id) {
        try {
            return entityManager.find(User.class, id);
        } catch (Exception e) {
            throw new DataAccessException("Error finding user by ID: " + id, e);
        }
    }
}
```

### 3. Использование JOIN FETCH для предотвращения N+1

```java
@Override
public List<User> findAllWithOrders() {
    return entityManager.createQuery(
        "from User u left join fetch u.orders", User.class)
        .getResultList();
}
```

### 4. Логирование операций

```java
@Slf4j
@Repository
public class UserRepositoryImpl implements UserRepository {
    
    @Override
    public User findById(Long id) {
        log.debug("Finding user by ID: {}", id);
        User user = entityManager.find(User.class, id);
        log.debug("Found user: {}", user);
        return user;
    }
}
```

## Заключение

Правильная настройка `CurrentSessionContext` и использование `EntityManager` позволяет:

- **Избежать** `LazyInitializationException`
- **Управлять** жизненным циклом сессий
- **Оптимизировать** производительность
- **Обеспечить** правильную работу транзакций
- **Упростить** тестирование

Выбор между `SessionFactory` и `EntityManager` зависит от требований проекта и предпочтений команды разработки. 