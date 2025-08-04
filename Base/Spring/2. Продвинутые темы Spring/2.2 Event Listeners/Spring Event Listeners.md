# Spring Event Listeners

Система событий Spring Framework предоставляет мощный механизм для реализации паттерна Observer (Наблюдатель). Это позволяет компонентам приложения взаимодействовать друг с другом без прямых зависимостей.

## Содержание

1. [Основы системы событий](#основы-системы-событий)
2. [Архитектура и компоненты](#архитектура-и-компоненты)
3. [Создание и публикация событий](#создание-и-публикация-событий)
4. [@EventListener аннотация](#eventlistener-аннотация)
5. [EventListenerMethodProcessor](#eventlistenermethodprocessor)
6. [Порядок выполнения](#порядок-выполнения)
7. [Практические примеры](#практические-примеры)
8. [Лучшие практики](#лучшие-практики)

## Основы системы событий

### Что такое система событий Spring?

Система событий Spring - это реализация паттерна Observer, которая позволяет компонентам приложения взаимодействовать без прямых зависимостей. Компоненты могут публиковать события и реагировать на события других компонентов.

### Зачем нужны события?

- **Слабая связанность** - компоненты не знают друг о друге напрямую
- **Расширяемость** - легко добавлять новые обработчики событий
- **Тестируемость** - можно легко мокать события в тестах
- **Асинхронность** - события могут обрабатываться асинхронно

## Архитектура и компоненты

### Основная архитектура

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Publisher     │───▶│  ApplicationEvent │───▶│   EventListener  │
│                 │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

### Внутренняя архитектура Spring

```
┌─────────────────────────────────────────────────────────────────┐
│                    ApplicationContext                          │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │ EventPublisher  │───▶│ EventMulticaster│───▶│ EventListener│ │
│  │                 │    │                 │    │             │ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Ключевые компоненты

1. **ApplicationEvent** - базовый класс для всех событий
2. **ApplicationEventPublisher** - интерфейс для публикации событий
3. **ApplicationListener** - интерфейс для обработки событий
4. **ApplicationEventMulticaster** - распределяет события между слушателями
5. **EventListenerMethodProcessor** - обрабатывает аннотации @EventListener

## Создание и публикация событий

### ApplicationEvent

Базовый класс для всех событий в Spring:

```java
public abstract class ApplicationEvent extends EventObject {
    private final long timestamp;
    
    public ApplicationEvent(Object source) {
        super(source);
        this.timestamp = System.currentTimeMillis();
    }
    
    public final long getTimestamp() {
        return this.timestamp;
    }
}
```

### Создание пользовательских событий

```java
public class UserCreatedEvent extends ApplicationEvent {
    private final User user;
    private final String createdBy;
    
    public UserCreatedEvent(Object source, User user, String createdBy) {
        super(source);
        this.user = user;
        this.createdBy = createdBy;
    }
    
    public User getUser() {
        return user;
    }
    
    public String getCreatedBy() {
        return createdBy;
    }
}
```

### ApplicationEventPublisher

Интерфейс для публикации событий:

```java
public interface ApplicationEventPublisher {
    void publishEvent(ApplicationEvent event);
    void publishEvent(Object event);
}
```

### Публикация событий

```java
@Service
public class UserService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public UserService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }
    
    public User createUser(UserDto userDto) {
        User user = new User(userDto.getName(), userDto.getEmail());
        user = userRepository.save(user);
        
        // Публикация события
        eventPublisher.publishEvent(new UserCreatedEvent(this, user, "system"));
        
        return user;
    }
}
```

### Реализация в ApplicationContext

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {
    
    private ApplicationEventMulticaster applicationEventMulticaster;
    
    @Override
    public void publishEvent(ApplicationEvent event) {
        publishEvent(event, null);
    }
    
    @Override
    public void publishEvent(Object event) {
        publishEvent(event, null);
    }
    
    protected void publishEvent(Object event, @Nullable ResolvableType type) {
        Assert.notNull(event, "Event must not be null");
        
        // Декорирование события
        ApplicationEvent applicationEvent;
        if (event instanceof ApplicationEvent) {
            applicationEvent = (ApplicationEvent) event;
        } else {
            applicationEvent = new PayloadApplicationEvent<>(this, event);
        }
        
        // Публикация через EventMulticaster
        getApplicationEventMulticaster().multicastEvent(applicationEvent, type);
    }
}
```

## @EventListener аннотация

### Основное использование

```java
@Component
public class UserEventListener {
    
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        System.out.println("User created: " + event.getUser().getName());
        // Логика обработки события
    }
}
```

### Условия выполнения

```java
@Component
public class ConditionalEventListener {
    
    @EventListener(condition = "#event.user.email.contains('@company.com')")
    public void handleCompanyUser(UserCreatedEvent event) {
        System.out.println("Company user created: " + event.getUser().getName());
    }
    
    @EventListener(condition = "#root.event.timestamp > 1000")
    public void handleRecentEvent(UserCreatedEvent event) {
        System.out.println("Recent event: " + event.getUser().getName());
    }
}
```

### Множественные события

```java
@Component
public class MultiEventListener {
    
    @EventListener
    public void handleUserEvents(UserCreatedEvent event) {
        System.out.println("User created: " + event.getUser().getName());
    }
    
    @EventListener
    public void handleUserEvents(UserUpdatedEvent event) {
        System.out.println("User updated: " + event.getUser().getName());
    }
    
    @EventListener
    public void handleUserEvents(UserDeletedEvent event) {
        System.out.println("User deleted: " + event.getUser().getName());
    }
}
```

### Асинхронная обработка

```java
@Component
public class AsyncEventListener {
    
    @EventListener
    @Async
    public void handleUserCreatedAsync(UserCreatedEvent event) {
        // Асинхронная обработка
        System.out.println("Async processing: " + event.getUser().getName());
    }
}
```

## EventListenerMethodProcessor

### Что такое EventListenerMethodProcessor?

`EventListenerMethodProcessor` - это `BeanFactoryPostProcessor`, который сканирует все бины и ищет методы, аннотированные `@EventListener`. Он автоматически регистрирует эти методы как обработчики событий.

### Как работает EventListenerMethodProcessor

```java
public class EventListenerMethodProcessor implements BeanFactoryPostProcessor, Ordered {
    
    private final EventListenerFactory eventListenerFactory;
    
    public EventListenerMethodProcessor() {
        this.eventListenerFactory = new DefaultEventListenerFactory();
    }
    
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // Получение EventListenerFactory
        EventListenerFactory factory = this.eventListenerFactory;
        
        // Сканирование всех бинов
        String[] beanNames = beanFactory.getBeanDefinitionNames();
        for (String beanName : beanNames) {
            try {
                // Получение типа бина
                Class<?> beanType = beanFactory.getType(beanName);
                if (beanType == null) {
                    continue;
                }
                
                // Поиск методов с @EventListener
                Set<Method> annotatedMethods = findAnnotatedMethods(beanType, EventListener.class);
                
                for (Method method : annotatedMethods) {
                    // Создание ApplicationListener
                    ApplicationListener<?> applicationListener = factory.createApplicationListener(beanName, beanType, method);
                    
                    // Регистрация в EventMulticaster
                    if (applicationListener instanceof ApplicationListenerMethodAdapter) {
                        ((ApplicationListenerMethodAdapter) applicationListener).init(beanFactory);
                    }
                    
                    // Добавление в контекст
                    beanFactory.addApplicationListener(applicationListener);
                }
            } catch (Exception e) {
                throw new BeanInitializationException("Failed to process @EventListener methods", e);
            }
        }
    }
    
    private Set<Method> findAnnotatedMethods(Class<?> type, Class<? extends Annotation> annotationType) {
        Set<Method> methods = new HashSet<>();
        
        // Поиск в текущем классе
        Method[] declaredMethods = type.getDeclaredMethods();
        for (Method method : declaredMethods) {
            if (method.isAnnotationPresent(annotationType)) {
                methods.add(method);
            }
        }
        
        // Поиск в суперклассах
        Class<?> superClass = type.getSuperclass();
        if (superClass != null && superClass != Object.class) {
            methods.addAll(findAnnotatedMethods(superClass, annotationType));
        }
        
        return methods;
    }
    
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```

### ApplicationListenerMethodAdapter

`ApplicationListenerMethodAdapter` - это адаптер, который превращает методы с аннотацией `@EventListener` в `ApplicationListener`:

```java
public class ApplicationListenerMethodAdapter implements ApplicationListener<ApplicationEvent>, Ordered {
    
    private final String beanName;
    private final Class<?> targetClass;
    private final Method method;
    private final EventExpressionEvaluator evaluator;
    
    public ApplicationListenerMethodAdapter(String beanName, Class<?> targetClass, Method method) {
        this.beanName = beanName;
        this.targetClass = targetClass;
        this.method = method;
        this.evaluator = new EventExpressionEvaluator();
    }
    
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        // Проверка условия
        EventListener eventListener = method.getAnnotation(EventListener.class);
        if (eventListener != null && !eventListener.condition().isEmpty()) {
            if (!evaluator.condition(eventListener.condition(), event, targetClass, method)) {
                return;
            }
        }
        
        // Вызов метода
        try {
            method.invoke(getTargetBean(), event);
        } catch (Exception e) {
            throw new ApplicationEventException("Error invoking event listener method", e);
        }
    }
    
    private Object getTargetBean() {
        // Получение бина из ApplicationContext
        return ApplicationContextHolder.getApplicationContext().getBean(beanName);
    }
}
```

### EventMulticaster

`EventMulticaster` отвечает за распределение событий между слушателями:

```java
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
    
    @Nullable
    private Executor taskExecutor;
    
    @Override
    public void multicastEvent(ApplicationEvent event) {
        multicastEvent(event, resolveDefaultEventType(event));
    }
    
    @Override
    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType type) {
        ResolvableType typeToUse = (type != null ? type : resolveDefaultEventType(event));
        
        // Получение всех слушателей для данного типа события
        for (ApplicationListener<?> listener : getApplicationListeners(event, typeToUse)) {
            Executor executor = getTaskExecutor();
            if (executor != null) {
                executor.execute(() -> invokeListener(listener, event));
            } else {
                invokeListener(listener, event);
            }
        }
    }
    
    private void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
        try {
            listener.onApplicationEvent(event);
        } catch (ClassCastException ex) {
            // Логирование ошибки
        }
    }
}
```

## Порядок выполнения

### @Order аннотация

```java
@Component
public class OrderedEventListeners {
    
    @EventListener
    @Order(1)
    public void firstListener(UserCreatedEvent event) {
        System.out.println("First listener executed");
    }
    
    @EventListener
    @Order(2)
    public void secondListener(UserCreatedEvent event) {
        System.out.println("Second listener executed");
    }
    
    @EventListener
    @Order(3)
    public void thirdListener(UserCreatedEvent event) {
        System.out.println("Third listener executed");
    }
}
```

### Ordered интерфейс

```java
@Component
public class OrderedEventListener implements Ordered {
    
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        System.out.println("Ordered listener executed");
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

## Практические примеры

### 1. Аудит событий

```java
@Component
public class AuditEventListener {
    
    private final AuditService auditService;
    
    public AuditEventListener(AuditService auditService) {
        this.auditService = auditService;
    }
    
    @EventListener
    public void auditUserCreated(UserCreatedEvent event) {
        AuditEntry audit = AuditEntry.builder()
            .action("USER_CREATED")
            .entityId(event.getUser().getId())
            .entityType("User")
            .performedBy(event.getCreatedBy())
            .timestamp(Instant.now())
            .build();
        
        auditService.save(audit);
    }
}
```

### 2. Уведомления

```java
@Component
public class NotificationEventListener {
    
    private final EmailService emailService;
    private final NotificationService notificationService;
    
    public NotificationEventListener(EmailService emailService, 
                                  NotificationService notificationService) {
        this.emailService = emailService;
        this.notificationService = notificationService;
    }
    
    @EventListener
    public void sendWelcomeEmail(UserCreatedEvent event) {
        User user = event.getUser();
        emailService.sendWelcomeEmail(user.getEmail(), user.getName());
    }
    
    @EventListener
    public void sendAdminNotification(UserCreatedEvent event) {
        notificationService.notifyAdmins("New user registered: " + event.getUser().getName());
    }
}
```

### 3. Кэширование

```java
@Component
public class CacheEventListener {
    
    private final CacheManager cacheManager;
    
    public CacheEventListener(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }
    
    @EventListener
    public void evictUserCache(UserUpdatedEvent event) {
        Cache userCache = cacheManager.getCache("users");
        userCache.evict(event.getUser().getId());
    }
    
    @EventListener
    public void evictAllCaches(UserDeletedEvent event) {
        cacheManager.getCacheNames()
            .forEach(name -> cacheManager.getCache(name).clear());
    }
}
```

### 4. Интеграция с внешними системами

```java
@Component
public class ExternalSystemEventListener {
    
    private final ExternalApiClient apiClient;
    
    public ExternalSystemEventListener(ExternalApiClient apiClient) {
        this.apiClient = apiClient;
    }
    
    @EventListener
    @Async
    public void syncWithExternalSystem(UserCreatedEvent event) {
        try {
            apiClient.createUser(event.getUser());
        } catch (Exception e) {
            // Логирование ошибки
            System.err.println("Failed to sync with external system: " + e.getMessage());
        }
    }
}
```

### 5. Мониторинг и метрики

```java
@Component
public class MetricsEventListener {
    
    private final MeterRegistry meterRegistry;
    
    public MetricsEventListener(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @EventListener
    public void recordUserMetrics(UserCreatedEvent event) {
        meterRegistry.counter("user.created").increment();
        meterRegistry.timer("user.creation.time").record(Duration.ofMillis(
            System.currentTimeMillis() - event.getTimestamp()
        ));
    }
}
```

## Лучшие практики

### 1. Проектирование событий

```java
// Хорошо - событие содержит всю необходимую информацию
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;
    private final String customerId;
    private final BigDecimal totalAmount;
    
    public OrderCreatedEvent(Object source, Order order, String customerId, BigDecimal totalAmount) {
        super(source);
        this.order = order;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
    }
    
    // Геттеры
}
```

### 2. Обработка исключений

```java
@Component
public class SafeEventListener {
    
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        try {
            // Логика обработки
        } catch (Exception e) {
            // Логирование ошибки без прерывания обработки других слушателей
            System.err.println("Error processing event: " + e.getMessage());
        }
    }
}
```

### 3. Условная обработка

```java
@Component
public class ConditionalEventListener {
    
    @EventListener(condition = "#event.user.email.endsWith('@company.com')")
    public void handleCompanyUser(UserCreatedEvent event) {
        // Обработка только корпоративных пользователей
    }
    
    @EventListener(condition = "#event.user.email.endsWith('@gmail.com')")
    public void handleGmailUser(UserCreatedEvent event) {
        // Обработка только Gmail пользователей
    }
}
```

### 4. Асинхронная обработка

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("EventAsync-");
        executor.initialize();
        return executor;
    }
}

@Component
public class AsyncEventListener {
    
    @EventListener
    @Async
    public void handleUserCreatedAsync(UserCreatedEvent event) {
        // Длительная обработка
    }
}
```

### 5. Тестирование

```java
@SpringBootTest
class UserEventListenerTest {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @MockBean
    private EmailService emailService;
    
    @Test
    void shouldSendWelcomeEmail() {
        // Given
        User user = new User("John", "john@example.com");
        UserCreatedEvent event = new UserCreatedEvent(this, user, "system");
        
        // When
        eventPublisher.publishEvent(event);
        
        // Then
        verify(emailService).sendWelcomeEmail("john@example.com", "John");
    }
}
```

## Связанные темы

- [BeanPostProcessor](../2.1 BeanPostProcessor/README.md)
- [IoC и Dependency Injection](../../1. Основы Spring/1.1 IoC и Dependency Injection/README.md) 