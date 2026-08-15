# Создание Proxy на классы Repository

## Обзор

Spring Data JPA использует механизм проксирования для создания реализаций интерфейсов репозиториев. Это позволяет автоматически генерировать код для методов, которые не реализованы явно.

## Механизм проксирования

### Типы прокси

#### JDK Dynamic Proxy
- Используется для интерфейсов
- Создается во время выполнения
- Основан на java.lang.reflect.Proxy

#### CGLIB Proxy
- Используется для классов
- Создается во время выполнения
- Основан на библиотеке CGLIB

### Пример создания прокси

```java
// Интерфейс репозитория
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
}

// Spring автоматически создает прокси
@Repository
public class UserRepositoryImpl implements UserRepository {
    // Реализация генерируется автоматически
}
```

## Структура прокси

### InvocationHandler
```java
public class RepositoryInvocationHandler implements InvocationHandler {
    
    private final RepositoryFactory repositoryFactory;
    private final Method method;
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // Логика обработки вызова метода
        return repositoryFactory.invoke(method, args);
    }
}
```

### Создание прокси
```java
public class RepositoryProxyFactory {
    
    public static <T> T createProxy(Class<T> repositoryInterface) {
        return (T) Proxy.newProxyInstance(
            repositoryInterface.getClassLoader(),
            new Class<?>[] { repositoryInterface },
            new RepositoryInvocationHandler()
        );
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
        
        // Анализ имени метода для определения операции
        if (methodName.startsWith("find")) {
            return new FindMethodMetadata(method);
        } else if (methodName.startsWith("save")) {
            return new SaveMethodMetadata(method);
        }
        
        return new CustomMethodMetadata(method);
    }
}
```

### Генерация реализации
```java
public class MethodImplementationGenerator {
    
    public static Object generateImplementation(Method method, Object[] args) {
        MethodMetadata metadata = MethodSignatureAnalyzer.analyze(method);
        
        switch (metadata.getOperationType()) {
            case FIND:
                return generateFindImplementation(metadata, args);
            case SAVE:
                return generateSaveImplementation(metadata, args);
            case DELETE:
                return generateDeleteImplementation(metadata, args);
            default:
                throw new UnsupportedOperationException();
        }
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
    repositoryBaseClass = CustomRepositoryImpl.class
)
public class JpaConfig {
    // конфигурация
}
```

### Кастомная реализация
```java
public class CustomRepositoryImpl<T, ID> extends SimpleJpaRepository<T, ID> {
    
    public CustomRepositoryImpl(JpaEntityInformation<T, ?> entityInformation, 
                              EntityManager entityManager) {
        super(entityInformation, entityManager);
    }
    
    @Override
    public <S extends T> S save(S entity) {
        // кастомная логика сохранения
        return super.save(entity);
    }
}
```

## Отладка прокси

### Включение логирования
```properties
logging.level.org.springframework.data.jpa.repository.support=DEBUG
logging.level.org.springframework.aop=DEBUG
```

### Проверка типа прокси
```java
@Component
public class RepositoryProxyChecker {
    
    @Autowired
    private ApplicationContext applicationContext;
    
    public void checkProxyTypes() {
        String[] beanNames = applicationContext.getBeanDefinitionNames();
        
        for (String beanName : beanNames) {
            Object bean = applicationContext.getBean(beanName);
            if (bean.getClass().getName().contains("Repository")) {
                System.out.println("Bean: " + beanName + 
                                 ", Type: " + bean.getClass().getName() +
                                 ", Proxy: " + AopUtils.isAopProxy(bean));
            }
        }
    }
}
```

## Производительность

### Кэширование прокси
```java
public class RepositoryProxyCache {
    
    private static final Map<Class<?>, Object> proxyCache = new ConcurrentHashMap<>();
    
    public static <T> T getOrCreateProxy(Class<T> repositoryInterface) {
        return (T) proxyCache.computeIfAbsent(repositoryInterface, 
            RepositoryProxyFactory::createProxy);
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
        return factory;
    }
}
```

## Лучшие практики

1. **Используйте интерфейсы** для репозиториев
2. **Избегайте сложной логики** в методах репозитория
3. **Используйте кастомные реализации** для сложных случаев
4. **Мониторьте производительность** прокси
5. **Тестируйте прокси** в unit-тестах
6. **Используйте логирование** для отладки 