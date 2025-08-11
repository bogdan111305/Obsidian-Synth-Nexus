# Создание Proxy на классы Repository

## Обзор

Spring Data JPA использует механизм проксирования для создания реализаций интерфейсов репозиториев. Это позволяет автоматически генерировать код для методов, которые не реализованы явно, обеспечивая прозрачность и гибкость в работе с данными.

## Механизм проксирования

### Типы прокси

#### JDK Dynamic Proxy
Используется для интерфейсов, создается во время выполнения:
```java
// Основан на java.lang.reflect.Proxy
public class JdkDynamicProxyExample {
    
    public static <T> T createProxy(Class<T> repositoryInterface) {
        return (T) Proxy.newProxyInstance(
            repositoryInterface.getClassLoader(),
            new Class<?>[] { repositoryInterface },
            new RepositoryInvocationHandler()
        );
    }
}
```

#### CGLIB Proxy
Используется для классов, создается во время выполнения:
```java
// Основан на библиотеке CGLIB
public class CglibProxyExample {
    
    public static <T> T createProxy(Class<T> repositoryClass) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(repositoryClass);
        enhancer.setCallback(new RepositoryMethodInterceptor());
        return (T) enhancer.create();
    }
}
```

### Пример создания прокси

```java
// Интерфейс репозитория
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
    List<User> findByActiveTrue();
}

// Spring автоматически создает прокси
@Repository
public class UserRepositoryImpl implements UserRepository {
    // Реализация генерируется автоматически
    // Методы findByEmail, findByUsername, findByActiveTrue
    // реализуются через механизм проксирования
}
```

## Структура прокси

### InvocationHandler для JDK Proxy
```java
public class RepositoryInvocationHandler implements InvocationHandler {
    
    private final RepositoryFactory repositoryFactory;
    private final Method method;
    private final Object[] args;
    
    public RepositoryInvocationHandler(RepositoryFactory repositoryFactory) {
        this.repositoryFactory = repositoryFactory;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // Анализ метода
        MethodMetadata metadata = analyzeMethod(method);
        
        // Определение типа операции
        OperationType operationType = determineOperationType(metadata);
        
        // Выполнение операции
        switch (operationType) {
            case FIND:
                return executeFindOperation(metadata, args);
            case SAVE:
                return executeSaveOperation(metadata, args);
            case DELETE:
                return executeDeleteOperation(metadata, args);
            case COUNT:
                return executeCountOperation(metadata, args);
            case EXISTS:
                return executeExistsOperation(metadata, args);
            default:
                throw new UnsupportedOperationException("Unknown operation type: " + operationType);
        }
    }
    
    private MethodMetadata analyzeMethod(Method method) {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        Class<?> returnType = method.getReturnType();
        
        return new MethodMetadata(methodName, parameterTypes, returnType);
    }
    
    private OperationType determineOperationType(MethodMetadata metadata) {
        String methodName = metadata.getMethodName();
        
        if (methodName.startsWith("find")) {
            return OperationType.FIND;
        } else if (methodName.startsWith("save")) {
            return OperationType.SAVE;
        } else if (methodName.startsWith("delete")) {
            return OperationType.DELETE;
        } else if (methodName.startsWith("count")) {
            return OperationType.COUNT;
        } else if (methodName.startsWith("exists")) {
            return OperationType.EXISTS;
        }
        
        return OperationType.CUSTOM;
    }
}
```

### MethodInterceptor для CGLIB
```java
public class RepositoryMethodInterceptor implements MethodInterceptor {
    
    private final RepositoryFactory repositoryFactory;
    private final Map<Method, MethodMetadata> methodCache = new ConcurrentHashMap<>();
    
    public RepositoryMethodInterceptor(RepositoryFactory repositoryFactory) {
        this.repositoryFactory = repositoryFactory;
    }
    
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        // Проверка кэша
        MethodMetadata metadata = methodCache.computeIfAbsent(method, this::analyzeMethod);
        
        // Выполнение операции
        return executeOperation(metadata, args);
    }
    
    private MethodMetadata analyzeMethod(Method method) {
        // Анализ метода и создание метаданных
        return new MethodMetadata(method);
    }
    
    private Object executeOperation(MethodMetadata metadata, Object[] args) {
        // Логика выполнения операции
        return repositoryFactory.execute(metadata, args);
    }
}
```

## Обработка методов

### Анализ сигнатуры метода
```java
public class MethodSignatureAnalyzer {
    
    public static MethodMetadata analyze(Method method) {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        Class<?> returnType = method.getReturnType();
        Annotation[] annotations = method.getAnnotations();
        
        // Анализ имени метода для определения операции
        OperationType operationType = determineOperationType(methodName);
        
        // Анализ параметров
        List<ParameterMetadata> parameters = analyzeParameters(method);
        
        // Анализ возвращаемого типа
        ReturnTypeMetadata returnTypeMetadata = analyzeReturnType(returnType);
        
        return new MethodMetadata(
            methodName,
            parameterTypes,
            returnType,
            operationType,
            parameters,
            returnTypeMetadata,
            annotations
        );
    }
    
    private static OperationType determineOperationType(String methodName) {
        if (methodName.startsWith("find")) {
            return OperationType.FIND;
        } else if (methodName.startsWith("save")) {
            return OperationType.SAVE;
        } else if (methodName.startsWith("delete")) {
            return OperationType.DELETE;
        } else if (methodName.startsWith("count")) {
            return OperationType.COUNT;
        } else if (methodName.startsWith("exists")) {
            return OperationType.EXISTS;
        }
        
        return OperationType.CUSTOM;
    }
    
    private static List<ParameterMetadata> analyzeParameters(Method method) {
        List<ParameterMetadata> parameters = new ArrayList<>();
        Parameter[] methodParameters = method.getParameters();
        
        for (int i = 0; i < methodParameters.length; i++) {
            Parameter parameter = methodParameters[i];
            parameters.add(new ParameterMetadata(
                parameter.getName(),
                parameter.getType(),
                parameter.getAnnotations(),
                i
            ));
        }
        
        return parameters;
    }
    
    private static ReturnTypeMetadata analyzeReturnType(Class<?> returnType) {
        if (returnType.equals(void.class)) {
            return new ReturnTypeMetadata(ReturnType.VOID, null);
        } else if (returnType.equals(Optional.class)) {
            return new ReturnTypeMetadata(ReturnType.OPTIONAL, getGenericType(returnType));
        } else if (returnType.equals(List.class)) {
            return new ReturnTypeMetadata(ReturnType.LIST, getGenericType(returnType));
        } else if (returnType.equals(Page.class)) {
            return new ReturnTypeMetadata(ReturnType.PAGE, getGenericType(returnType));
        } else if (returnType.equals(Slice.class)) {
            return new ReturnTypeMetadata(ReturnType.SLICE, getGenericType(returnType));
        }
        
        return new ReturnTypeMetadata(ReturnType.SINGLE, returnType);
    }
}
```

### Генерация реализации
```java
public class MethodImplementationGenerator {
    
    private final EntityManager entityManager;
    private final QueryMethodFactory queryMethodFactory;
    
    public MethodImplementationGenerator(EntityManager entityManager) {
        this.entityManager = entityManager;
        this.queryMethodFactory = new QueryMethodFactory(entityManager);
    }
    
    public static Object generateImplementation(Method method, Object[] args) {
        MethodMetadata metadata = MethodSignatureAnalyzer.analyze(method);
        
        switch (metadata.getOperationType()) {
            case FIND:
                return generateFindImplementation(metadata, args);
            case SAVE:
                return generateSaveImplementation(metadata, args);
            case DELETE:
                return generateDeleteImplementation(metadata, args);
            case COUNT:
                return generateCountImplementation(metadata, args);
            case EXISTS:
                return generateExistsImplementation(metadata, args);
            default:
                throw new UnsupportedOperationException("Unknown operation type: " + metadata.getOperationType());
        }
    }
    
    private static Object generateFindImplementation(MethodMetadata metadata, Object[] args) {
        // Генерация SQL запроса на основе метаданных
        String query = generateQuery(metadata);
        
        // Создание TypedQuery
        TypedQuery<?> typedQuery = entityManager.createQuery(query, metadata.getReturnType());
        
        // Установка параметров
        setParameters(typedQuery, metadata.getParameters(), args);
        
        // Выполнение запроса
        return executeQuery(typedQuery, metadata.getReturnTypeMetadata());
    }
    
    private static String generateQuery(MethodMetadata metadata) {
        StringBuilder query = new StringBuilder();
        
        switch (metadata.getReturnTypeMetadata().getType()) {
            case SINGLE:
                query.append("SELECT e FROM ").append(getEntityName(metadata)).append(" e");
                break;
            case LIST:
                query.append("SELECT e FROM ").append(getEntityName(metadata)).append(" e");
                break;
            case OPTIONAL:
                query.append("SELECT e FROM ").append(getEntityName(metadata)).append(" e");
                break;
            case PAGE:
                query.append("SELECT e FROM ").append(getEntityName(metadata)).append(" e");
                break;
        }
        
        // Добавление условий WHERE
        String whereClause = generateWhereClause(metadata);
        if (!whereClause.isEmpty()) {
            query.append(" WHERE ").append(whereClause);
        }
        
        // Добавление ORDER BY
        String orderByClause = generateOrderByClause(metadata);
        if (!orderByClause.isEmpty()) {
            query.append(" ORDER BY ").append(orderByClause);
        }
        
        return query.toString();
    }
}
```

## Конфигурация прокси

### Настройка через @EnableJpaRepositories
```java
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.repository",
    repositoryImplementationPostfix = "Impl",
    repositoryBaseClass = CustomRepositoryImpl.class,
    namedQueriesLocation = "classpath:jpa-named-queries.properties",
    considerNestedRepositories = true
)
public class JpaConfig {
    
    @Bean
    public RepositoryFactorySupport repositoryFactorySupport(EntityManager entityManager) {
        JpaRepositoryFactory factory = new JpaRepositoryFactory(entityManager);
        factory.setRepositoryBaseClass(CustomRepositoryImpl.class);
        factory.setRepositoryImplementationPostfix("Impl");
        return factory;
    }
}
```

### Кастомная реализация
```java
public class CustomRepositoryImpl<T, ID> extends SimpleJpaRepository<T, ID> {
    
    private final EntityManager entityManager;
    private final JpaEntityInformation<T, ?> entityInformation;
    
    public CustomRepositoryImpl(JpaEntityInformation<T, ?> entityInformation, 
                              EntityManager entityManager) {
        super(entityInformation, entityManager);
        this.entityManager = entityManager;
        this.entityInformation = entityInformation;
    }
    
    @Override
    public <S extends T> S save(S entity) {
        // Кастомная логика сохранения
        if (entity instanceof Auditable) {
            ((Auditable) entity).setLastModified(LocalDateTime.now());
        }
        
        if (entityInformation.isNew(entity)) {
            if (entity instanceof Auditable) {
                ((Auditable) entity).setCreatedAt(LocalDateTime.now());
            }
        }
        
        return super.save(entity);
    }
    
    @Override
    public void delete(T entity) {
        // Кастомная логика удаления
        if (entity instanceof SoftDeletable) {
            ((SoftDeletable) entity).setDeleted(true);
            ((SoftDeletable) entity).setDeletedAt(LocalDateTime.now());
            super.save(entity);
        } else {
            super.delete(entity);
        }
    }
    
    @Override
    public void deleteById(ID id) {
        // Кастомная логика удаления по ID
        Optional<T> entity = findById(id);
        if (entity.isPresent()) {
            delete(entity.get());
        }
    }
    
    @Override
    public <S extends T> List<S> saveAll(Iterable<S> entities) {
        // Кастомная логика массового сохранения
        List<S> result = new ArrayList<>();
        for (S entity : entities) {
            result.add(save(entity));
        }
        return result;
    }
}
```

## Отладка прокси

### Включение логирования
```properties
# Spring Data JPA логирование
logging.level.org.springframework.data.jpa=DEBUG
logging.level.org.springframework.data.jpa.repository.support=DEBUG

# AOP логирование
logging.level.org.springframework.aop=DEBUG

# Proxy логирование
logging.level.org.springframework.data.jpa.repository.support.RepositoryFactorySupport=DEBUG
```

### Проверка типа прокси
```java
@Component
public class RepositoryProxyChecker {
    
    @Autowired
    private ApplicationContext applicationContext;
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkProxyTypes() {
        System.out.println("=== Repository Proxy Check ===");
        
        String[] beanNames = applicationContext.getBeanDefinitionNames();
        
        for (String beanName : beanNames) {
            Object bean = applicationContext.getBean(beanName);
            if (bean.getClass().getName().contains("Repository")) {
                System.out.println("Bean: " + beanName);
                System.out.println("  Type: " + bean.getClass().getName());
                System.out.println("  Is Proxy: " + AopUtils.isAopProxy(bean));
                System.out.println("  Proxy Type: " + AopUtils.getTargetClass(bean).getName());
                
                // Проверка методов
                Method[] methods = bean.getClass().getMethods();
                for (Method method : methods) {
                    if (method.getName().startsWith("find") || 
                        method.getName().startsWith("save") || 
                        method.getName().startsWith("delete")) {
                        System.out.println("    Method: " + method.getName());
                    }
                }
                System.out.println();
            }
        }
        
        System.out.println("=== Proxy Check Complete ===");
    }
}
```

### Анализ прокси методов
```java
@Component
public class RepositoryMethodAnalyzer {
    
    @Autowired
    private ApplicationContext applicationContext;
    
    public void analyzeRepositoryMethods(String repositoryBeanName) {
        Object repository = applicationContext.getBean(repositoryBeanName);
        Class<?> repositoryClass = repository.getClass();
        
        System.out.println("=== Repository Method Analysis ===");
        System.out.println("Repository: " + repositoryBeanName);
        System.out.println("Class: " + repositoryClass.getName());
        
        Method[] methods = repositoryClass.getMethods();
        for (Method method : methods) {
            if (method.getDeclaringClass() != Object.class) {
                System.out.println("  Method: " + method.getName());
                System.out.println("    Return Type: " + method.getReturnType().getSimpleName());
                System.out.println("    Parameters: " + Arrays.toString(method.getParameterTypes()));
                
                // Анализ аннотаций
                Annotation[] annotations = method.getAnnotations();
                if (annotations.length > 0) {
                    System.out.println("    Annotations: " + Arrays.toString(annotations));
                }
            }
        }
        
        System.out.println("=== Analysis Complete ===");
    }
}
```

## Производительность

### Кэширование прокси
```java
public class RepositoryProxyCache {
    
    private static final Map<Class<?>, Object> proxyCache = new ConcurrentHashMap<>();
    private static final Map<Class<?>, MethodMetadata[]> methodCache = new ConcurrentHashMap<>();
    
    public static <T> T getOrCreateProxy(Class<T> repositoryInterface) {
        return (T) proxyCache.computeIfAbsent(repositoryInterface, 
            RepositoryProxyFactory::createProxy);
    }
    
    public static MethodMetadata[] getOrCreateMethodMetadata(Class<?> repositoryInterface) {
        return methodCache.computeIfAbsent(repositoryInterface, 
            RepositoryProxyFactory::analyzeMethods);
    }
    
    public static void clearCache() {
        proxyCache.clear();
        methodCache.clear();
    }
}
```

### Оптимизация создания
```java
@Configuration
public class OptimizedJpaConfig {
    
    @Bean
    public RepositoryFactorySupport repositoryFactorySupport(EntityManager entityManager) {
        JpaRepositoryFactory factory = new JpaRepositoryFactory(entityManager);
        factory.setRepositoryBaseClass(CustomRepositoryImpl.class);
        
        // Включение кэширования
        factory.setRepositoryImplementationPostfix("Impl");
        factory.setRepositoryBaseClass(CustomRepositoryImpl.class);
        
        // Оптимизация создания прокси
        factory.setRepositoryFactoryBeanClass(OptimizedJpaRepositoryFactoryBean.class);
        
        return factory;
    }
}

public class OptimizedJpaRepositoryFactoryBean<R extends JpaRepository<T, I>, T, I extends Serializable>
        extends JpaRepositoryFactoryBean<R, T, I> {
    
    public OptimizedJpaRepositoryFactoryBean(Class<? extends R> repositoryInterface) {
        super(repositoryInterface);
    }
    
    @Override
    protected RepositoryFactorySupport createRepositoryFactory(EntityManager entityManager) {
        return new OptimizedJpaRepositoryFactory(entityManager);
    }
}
```

## Тестирование прокси

### Unit тестирование
```java
@ExtendWith(MockitoExtension.class)
class RepositoryProxyTest {
    
    @Mock
    private EntityManager entityManager;
    
    @Mock
    private TypedQuery<User> typedQuery;
    
    @Test
    void testProxyCreation() {
        // given
        when(entityManager.createQuery(anyString(), eq(User.class))).thenReturn(typedQuery);
        when(typedQuery.setParameter(anyString(), any())).thenReturn(typedQuery);
        when(typedQuery.getResultList()).thenReturn(Arrays.asList(new User()));
        
        // when
        UserRepository userRepository = createProxy(UserRepository.class);
        List<User> users = userRepository.findByEmail("test@example.com");
        
        // then
        assertNotNull(users);
        assertEquals(1, users.size());
        verify(entityManager).createQuery(contains("SELECT"), eq(User.class));
    }
    
    private <T> T createProxy(Class<T> repositoryInterface) {
        // Создание прокси для тестирования
        return RepositoryProxyFactory.createProxy(repositoryInterface, entityManager);
    }
}
```

### Интеграционное тестирование
```java
@DataJpaTest
class RepositoryProxyIntegrationTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testProxyMethods() {
        // given
        User user = new User();
        user.setEmail("test@example.com");
        user.setUsername("testuser");
        
        // when
        User savedUser = userRepository.save(user);
        Optional<User> foundUser = userRepository.findByUsername("testuser");
        List<User> usersByEmail = userRepository.findByEmail("test@example.com");
        
        // then
        assertTrue(foundUser.isPresent());
        assertEquals("testuser", foundUser.get().getUsername());
        assertEquals(1, usersByEmail.size());
        
        // Проверка типа прокси
        assertTrue(AopUtils.isAopProxy(userRepository));
    }
}
```

## Лучшие практики

### 1. Использование интерфейсов
```java
// Правильно - используйте интерфейсы для репозиториев
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
}

// Неправильно - не используйте классы
public class UserRepository extends JpaRepository<User, Long> {
    // методы
}
```

### 2. Избегание сложной логики
```java
// Правильно - простая логика в методах репозитория
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
}

// Неправильно - сложная бизнес-логика в репозитории
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findActiveUsersWithComplexBusinessLogic();
}
```

### 3. Использование кастомных реализаций
```java
// Для сложных случаев используйте кастомные реализации
public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom {
    List<User> findByEmail(String email);
}

public interface UserRepositoryCustom {
    List<User> findUsersWithComplexQuery();
}

public class UserRepositoryImpl implements UserRepositoryCustom {
    @Override
    public List<User> findUsersWithComplexQuery() {
        // сложная логика
    }
}
```

### 4. Мониторинг производительности
```java
@Component
public class RepositoryPerformanceMonitor {
    
    @EventListener(ApplicationReadyEvent.class)
    public void monitorPerformance() {
        // Логика мониторинга производительности прокси
    }
}
```

### 5. Тестирование прокси
```java
// Тестируйте прокси в unit-тестах
@ExtendWith(MockitoExtension.class)
class RepositoryProxyTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Test
    void testProxyMethods() {
        // тестирование методов прокси
    }
}
```

### 6. Использование логирования
```properties
# Включите логирование для отладки прокси
logging.level.org.springframework.data.jpa=DEBUG
logging.level.org.springframework.aop=DEBUG
```

## Заключение

Механизм проксирования в Spring Data JPA является мощным инструментом, который обеспечивает:
- Автоматическую генерацию реализации методов
- Прозрачность работы с данными
- Гибкость в настройке и кастомизации
- Высокую производительность через кэширование
- Простоту тестирования и отладки

Правильное понимание и использование проксирования позволяет создавать эффективные и надежные репозитории в Spring приложениях.
