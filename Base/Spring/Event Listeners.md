## 1. Введение в события в Spring

**События** в Spring — это объекты, представляющие изменения состояния или действия в приложении, которые могут быть опубликованы и обработаны другими компонентами. Механизм событий позволяет разделить логику отправителя и получателя, обеспечивая слабую связанность.

### 1.1. Основные компоненты

- **Событие (Event)**: Объект, содержащий информацию о произошедшем действии (например, `ApplicationEvent`).
- **Публикатор (Publisher)**: Компонент, отправляющий событие через `ApplicationEventPublisher`.
- **Слушатель (Listener)**: Компонент, обрабатывающий событие, реализующий `ApplicationListener` или использующий `@EventListener`.
- **ApplicationContext**: Контейнер, управляющий публикацией и маршрутизацией событий.

### 1.2. Преимущества

- **Слабая связанность**: Компоненты не зависят друг от друга напрямую.
- **Асинхронность**: События могут обрабатываться в отдельных потоках.
- **Модульность**: Логика обработки событий отделена от бизнес-логики.
- **Расширяемость**: Легко добавить новых слушателей без изменения кода.
## 2. Базовый механизм событий

Spring использует `ApplicationEvent` и `ApplicationEventPublisher` для реализации событийной модели. `ApplicationContext` выступает в роли публикатора событий по умолчанию.

### 2.1. ApplicationEvent

- **Описание**: Базовый класс для всех событий в Spring.
- **Иерархия**: Все пользовательские события наследуются от `ApplicationEvent`.
- **Поля**:
    - `source`: Объект, инициировавший событие.
    - `timestamp`: Время создания события.

**Пример базового события**:

```java
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;

    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }

    public Order getOrder() {
        return order;
    }
}
```

### 2.2. ApplicationEventPublisher

- **Описание**: Интерфейс для публикации событий.
- **Реализация**: `ApplicationContext` реализует `ApplicationEventPublisher`.
- **Методы**:
    - `publishEvent(Object event)`: Публикует событие.
    - `publishEvent(ApplicationEvent event)`: Публикует событие типа `ApplicationEvent`.

**Пример публикации**:

```java
@Service
public class OrderService {
    private final ApplicationEventPublisher publisher;

    @Autowired
    public OrderService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void createOrder(Order order) {
        // Логика создания заказа
        publisher.publishEvent(new OrderCreatedEvent(this, order));
    }
}
```

---

## 3. Создание слушателей событий

Spring поддерживает два основных способа создания слушателей: через реализацию интерфейса `ApplicationListener` и аннотацию `@EventListener`.

### 3.1. ApplicationListener

- **Описание**: Интерфейс для обработки событий определённого типа.
- **Метод**: `onApplicationEvent(E event)`.
- **Ограничение**: Один слушатель обрабатывает события одного типа.

**Пример**:

```java
@Component
public class OrderCreatedListener implements ApplicationListener<OrderCreatedEvent> {
    @Override
    public void onApplicationEvent(OrderCreatedEvent event) {
        System.out.println("Order created: " + event.getOrder().getId());
    }
}
```

### 3.2. @EventListener

- **Описание**: Аннотация для методов, обрабатывающих события.
- **Преимущества**:
    - Не требует реализации интерфейса.
    - Поддерживает несколько типов событий в одном методе.
    - Позволяет использовать SpEL для условий.
- **Свойства**:
    - `value`: Типы событий (массив классов).
    - `condition`: SpEL-условие для обработки.

**Пример**:

```java
@Service
public class NotificationService {
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        System.out.println("Sending notification for order: " + event.getOrder().getId());
    }
}
```

#### 3.2.1. Механизм обработки @EventListener: EventListenerMethodProcessor

- **Описание**: `EventListenerMethodProcessor` — это бин, реализующий интерфейсы `BeanFactoryPostProcessor` и `ApplicationContextAware`, ответственный за обработку методов, помеченных `@EventListener` или `@TransactionalEventListener`. Он сканирует бины на наличие таких методов и регистрирует их как слушатели событий.
    
- **Как работает EventListenerMethodProcessor**:
    
    1. **Инициализация**: `EventListenerMethodProcessor` регистрируется как бин в контексте (автоматически через Spring или вручную в `@Configuration`).
    2. **Фаза BeanFactoryPostProcessor**:
        - На стадии `postProcessBeanFactory` процессор подготавливает инфраструктуру, но не сканирует методы, так как бины ещё не созданы.
        - Он регистрирует зависимости, такие как `EventListenerFactory` (по умолчанию `DefaultEventListenerFactory`).
    3. **Фаза после инициализации бинов**:
        - После создания всех бинов (`ApplicationContext` вызывает `afterSingletonsInstantiated` через `SmartInitializingSingleton`), процессор сканирует их на наличие методов с `@EventListener`.
        - Для каждого метода создаётся адаптер `ApplicationListener`, который оборачивает метод и делегирует вызовы.
    4. **Регистрация слушателей**:
        - Адаптеры `ApplicationListener` добавляются в `ApplicationEventMulticaster` (по умолчанию `SimpleApplicationEventMulticaster`), а не в `BeanFactory` как отдельные бины.
        - `ApplicationEventMulticaster` отвечает за рассылку событий всем зарегистрированным слушателям.
    5. **Обработка событий**:
        - При вызове `publishEvent` `ApplicationEventMulticaster` вызывает соответствующие адаптеры, которые выполняют методы с `@EventListener`.
- **Ключевые классы**:
    
    - `EventListenerMethodProcessor`: Сканирует методы и регистрирует слушатели.
    - `DefaultEventListenerFactory`: Создаёт адаптеры `ApplicationListener` для методов.
    - `SimpleApplicationEventMulticaster`: Рассылает события синхронно или асинхронно (с настройкой `TaskExecutor`).
    - `SpelExpressionParser`: Обрабатывает SpEL-условия в `condition`.
- **Кастомная настройка**:
    

```java
@Configuration
public class EventConfig {
    @Bean
    public EventListenerMethodProcessor eventListenerMethodProcessor() {
        EventListenerMethodProcessor processor = new EventListenerMethodProcessor();
        processor.setEventListenerFactory(new DefaultEventListenerFactory());
        return processor;
    }

    @Bean
    public ApplicationEventMulticaster applicationEventMulticaster() {
        SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
        multicaster.setTaskExecutor(Executors.newFixedThreadPool(5)); // Асинхронная обработка
        multicaster.setErrorHandler(t -> System.err.println("Event error: " + t.getMessage()));
        return multicaster;
    }
}
```

- **Особенности**:
    - **Не создаёт бины в BeanFactory**: Методы с `@EventListener` не регистрируются как отдельные бины типа `ApplicationListener`. Вместо этого создаются временные адаптеры для `ApplicationEventMulticaster`.
    - **Поддержка SpEL**: Условия в `condition` обрабатываются через `SpelExpressionParser`, что позволяет фильтровать события.
    - **Асинхронность**: Если настроен `TaskExecutor` в `ApplicationEventMulticaster`, события обрабатываются асинхронно (при использовании `@Async`).
    - **Ошибки**: Исключения в методах обрабатываются `ErrorHandler` в `ApplicationEventMulticaster` (или `AsyncUncaughtExceptionHandler` для асинхронных).

#### 3.2.2. Свойство condition и фильтрация событий

- **Описание**: Свойство `condition` в `@EventListener` позволяет задавать SpEL-выражение, которое должно быть истинным для вызова метода. Это мощный инструмент для фильтрации событий на основе их свойств, состояния контекста или внешних условий.
    
- **Синтаксис**:
    
    - SpEL-выражение указывается в атрибуте `condition` и должно возвращать `boolean`.
    - Переменная `#event` ссылается на объект события.
    - Доступны свойства события, `Environment`, `systemProperties`, другие бины через `#beanName`.
- **Примеры фильтрации**:
    

1. **Фильтрация по свойству события**:

```java
@Service
public class NotificationService {
    @EventListener(condition = "#event.order.total > 1000")
    public void handleHighValueOrder(OrderCreatedEvent event) {
        System.out.println("High-value order: " + event.getOrder().getId());
    }
}
```

- **Объяснение**: Метод вызывается, только если `total` заказа в событии `OrderCreatedEvent` превышает 1000.

2. **Фильтрация по нескольким условиям**:

```java
@Service
public class NotificationService {
    @EventListener(condition = "#event.order.total > 1000 and #event.order.id != null")
    public void handlePremiumOrder(OrderCreatedEvent event) {
        System.out.println("Premium order: " + event.getOrder().getId());
    }
}
```

- **Объяснение**: Метод вызывается, если `total` заказа больше 1000 и `id` не null.

3. **Доступ к Environment**:

```java
@Service
public class NotificationService {
    @EventListener(condition = "#event.order.total > 1000 and T(java.lang.Boolean).parseBoolean(environment.getProperty('app.notify.enabled'))")
    public void handleOrder(OrderCreatedEvent event) {
        System.out.println("Notification for order: " + event.getOrder().getId());
    }
}
```

```properties
app.notify.enabled=true
```

- **Объяснение**: Метод вызывается, если `total` заказа больше 1000 и свойство `app.notify.enabled` равно `true`.

4. **Фильтрация по типу события**:

```java
public class OrderUpdatedEvent extends ApplicationEvent {
    private final Order order;

    public OrderUpdatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }

    public Order getOrder() {
        return order;
    }
}

@Service
public class AuditService {
    @EventListener(condition = "#event instanceof T(com.example.OrderCreatedEvent)")
    public void logEvent(ApplicationEvent event) {
        System.out.println("Logging OrderCreatedEvent: " + ((OrderCreatedEvent) event).getOrder().getId());
    }
}
```

- **Объяснение**: Метод вызывается только для событий типа `OrderCreatedEvent`.

5. **Доступ к бинам**:

```java
@Service
public class ConfigService {
    public boolean isNotificationsEnabled() {
        return true;
    }
}

@Service
public class NotificationService {
    @EventListener(condition = "#event.order.total > 500 and @configService.notificationsEnabled")
    public void handleOrder(OrderCreatedEvent event) {
        System.out.println("Notification for order: " + event.getOrder().getId());
    }
}
```

- **Объяснение**: Метод вызывается, если `total` заказа больше 500 и метод `isNotificationsEnabled` бина `configService` возвращает `true`.

6. **Фильтрация POJO-событий**:

```java
public class PaymentProcessed {
    private final String paymentId;
    private final double amount;

    public PaymentProcessed(String paymentId, double amount) {
        this.paymentId = paymentId;
        this.amount = amount;
    }

    public String getPaymentId() { return paymentId; }
    public double getAmount() { return amount; }
}

@Service
public class AccountingService {
    @EventListener(condition = "#event.amount > 100 and #event.paymentId.startsWith('PAY-')")
    public void recordPayment(PaymentProcessed event) {
        System.out.println("Recording payment: " + event.getPaymentId());
    }
}
```

- **Объяснение**: Метод вызывается для `PaymentProcessed`, если `amount` больше 100 и `paymentId` начинается с "PAY-".
    
- **Механизм обработки condition**:
    
    - SpEL-выражение парсится через `SpelExpressionParser`.
    - Контекст вычисления включает:
        - `#event`: Объект события.
        - `#root`: То же, что `#event`.
        - Бины через `@beanName`.
        - `Environment` через `environment`.
        - `systemProperties`, `systemEnvironment`.
    - Если выражение возвращает `false`, метод не вызывается.
    - Исключения в SpEL (например, `NullPointerException`) обрабатываются `ApplicationEventMulticaster`.
- **Рекомендации по condition**:
    
    - Проверяйте null-значения: используйте `?.` для безопасного доступа (например, `#event.order?.total > 1000`).
    - Избегайте сложных выражений для повышения производительности.
    - Логируйте условия для отладки.

---

## 4. Транзакционные слушатели (@TransactionalEventListener)

- **Описание**: Аннотация для обработки событий, связанных с транзакциями.
- **Назначение**: Слушатель вызывается только после успешного завершения транзакции (или при определённых фазах).
- **Свойства**:
    - `value`: Тип события.
    - `phase`: Фаза транзакции (`AFTER_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION`, `BEFORE_COMMIT`).
    - `fallbackExecution`: Если `true`, слушатель вызывается даже без транзакции (по умолчанию `false`).

**Пример**:

```java
@Service
public class OrderService {
    @Transactional
    public void createOrder(Order order) {
        // Логика создания заказа
        publisher.publishEvent(new OrderCreatedEvent(this, order));
    }
}

@Service
public class EmailService {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendEmail(OrderCreatedEvent event) {
        System.out.println("Email sent for order: " + event.getOrder().getId());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void logFailure(OrderCreatedEvent event) {
        System.out.println("Order creation failed: " + event.getOrder().getId());
    }
}
```

**Пример без транзакции**:

```java
@Service
public class NotificationService {
    @TransactionalEventListener(fallbackExecution = true)
    public void notify(OrderCreatedEvent event) {
        System.out.println("Notification sent for order: " + event.getOrder().getId());
    }
}
```

---

## 5. Кастомные события

**Описание**: Пользовательские события создаются путём наследования от `ApplicationEvent` или создания POJO для использования с `@EventListener`.

### 5.1. Наследование от ApplicationEvent

**Пример**:

```java
public class UserRegisteredEvent extends ApplicationEvent {
    private final User user;

    public UserRegisteredEvent(Object source, User user) {
        super(source);
        this.user = user;
    }

    public User getUser() {
        return user;
    }
}
```

### 5.2. POJO-события

- **Описание**: События могут быть любыми объектами, если используются с `@EventListener`.
- **Ограничение**: Не передают `source` и `timestamp`.

**Пример**:

```java
public class PaymentProcessed {
    private final String paymentId;
    private final double amount;

    public PaymentProcessed(String paymentId, double amount) {
        this.paymentId = paymentId;
        this.amount = amount;
    }

    public String getPaymentId() { return paymentId; }
    public double getAmount() { return amount; }
}
```

```java
@Service
public class PaymentService {
    private final ApplicationEventPublisher publisher;

    @Autowired
    public PaymentService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void processPayment(String paymentId, double amount) {
        // Логика обработки платежа
        publisher.publishEvent(new PaymentProcessed(paymentId, amount));
    }
}

@Service
public class AccountingService {
    @EventListener
    public void recordPayment(PaymentProcessed event) {
        System.out.println("Payment recorded: " + event.getPaymentId() + ", amount: " + event.getAmount());
    }
}
```

---

## 6. Асинхронная обработка событий

**Описание**: События могут обрабатываться асинхронно для повышения производительности и отзывчивости приложения.

### 6.1. Включение асинхронности

- **Аннотация**: `@EnableAsync` на конфигурационном классе.
- **Конфигурация пула потоков**:

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("EventListener-");
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> System.err.println("Async error: " + ex.getMessage());
    }
}
```

### 6.2. Асинхронные слушатели

- **Аннотация**: `@Async` на методе `@EventListener`.

**Пример**:

```java
@Service
public class NotificationService {
    @Async
    @EventListener
    public void sendAsyncNotification(OrderCreatedEvent event) {
        System.out.println("Async notification for order: " + event.getOrder().getId() + 
                           ", Thread: " + Thread.currentThread().getName());
    }
}
```

**Примечание**:

- Требуется `@EnableAsync`.
- Асинхронные слушатели не поддерживают `@TransactionalEventListener` без кастомной настройки.

---

## 7. Порядок выполнения слушателей

**Описание**: Порядок выполнения слушателей событий можно контролировать через аннотацию `@Order` или интерфейс `Ordered`.

### 7.1. Использование @Order

**Пример**:

```java
@Service
public class NotificationService {
    @EventListener
    @Order(1)
    public void sendEmail(OrderCreatedEvent event) {
        System.out.println("Email sent for order: " + event.getOrder().getId());
    }

    @EventListener
    @Order(2)
    public void sendSms(OrderCreatedEvent event) {
        System.out.println("SMS sent for order: " + event.getOrder().getId());
    }
}
```

### 7.2. Использование Ordered

**Пример**:

```java
@Component
public class AuditListener implements ApplicationListener<OrderCreatedEvent>, Ordered {
    @Override
    public void onApplicationEvent(OrderCreatedEvent event) {
        System.out.println("Audit logged for order: " + event.getOrder().getId());
    }

    @Override
    public int getOrder() {
        return 0; // Высший приоритет
    }
}
```

**Примечание**:

- Меньшее значение `Order` означает более высокий приоритет.
- По умолчанию слушатели выполняются в порядке регистрации.

---

## 8. Интеграция с ApplicationContext

**Описание**: `ApplicationContext` автоматически публикует встроенные события и управляет пользовательскими событиями.

### 8.1. Встроенные события

| **Событие**             | **Описание**                           |
| ----------------------- | -------------------------------------- |
| `ContextRefreshedEvent` | Контекст инициализирован или обновлён. |
| `ContextStartedEvent`   | Контекст запущен.                      |
| `ContextStoppedEvent`   | Контекст остановлен.                   |
| `ContextClosedEvent`    | Контекст закрыт.                       |
| `RequestHandledEvent`   | HTTP-запрос обработан (веб).           |

**Пример обработки встроенного события**:

```java
@Service
public class ContextListener {
    @EventListener
    public void handleContextRefresh(ContextRefreshedEvent event) {
        System.out.println("Context refreshed at: " + event.getTimestamp());
    }
}
```

### 8.2. Публикация через ApplicationContext

```java
@Configuration
public class AppConfig {
    @Bean
    public ApplicationListener<ContextRefreshedEvent> contextListener() {
        return event -> System.out.println("Context refreshed: " + event.getTimestamp());
    }
}
```

---

## 9. Примеры конфигурации и использования

### 9.1. Полный пример с кастомным событием и condition

```java
// Модель
public class Order {
    private final int id;
    private final double total;
    private final String status;

    public Order(int id, double total, String status) {
        this.id = id;
        this.total = total;
        this.status = status;
    }

    public int getId() { return id; }
    public double getTotal() { return total; }
    public String getStatus() { return status; }
}

// Событие
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;

    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }

    public Order getOrder() { return order; }
}

// Сервис
@Service
public class OrderService {
    private final ApplicationEventPublisher publisher;

    @Autowired
    public OrderService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    @Transactional
    public void createOrder(int id, double total, String status) {
        Order order = new Order(id, total, status);
        publisher.publishEvent(new OrderCreatedEvent(this, order));
    }
}

// Конфигурация
@Service
public class ConfigService {
    public boolean isVipNotificationsEnabled() {
        return true;
    }
}

// Слушатели
@Service
public class NotificationService {
    @EventListener(condition = "#event.order.total > 1000 and #event.order.status == 'CONFIRMED'")
    @Order(1)
    public void sendEmail(OrderCreatedEvent event) {
        System.out.println("Email sent for confirmed high-value order: " + event.getOrder().getId());
    }

    @Async
    @EventListener(condition = "#event.order.total > 500 and @configService.vipNotificationsEnabled")
    public void sendVipNotification(OrderCreatedEvent event) {
        System.out.println("VIP notification for order: " + event.getOrder().getId() + 
                           ", Thread: " + Thread.currentThread().getName());
    }
}

@Service
public class EmailService {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void confirmOrder(OrderCreatedEvent event) {
        System.out.println("Confirmation email sent for order: " + event.getOrder().getId());
    }
}

// Конфигурация
@Configuration
@EnableAsync
public class AppConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.initialize();
        return executor;
    }
}

// Тест
public class Main {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        OrderService service = context.getBean(OrderService.class);
        service.createOrder(1, 1500.0, "CONFIRMED"); // Публикует событие
        service.createOrder(2, 600.0, "PENDING"); // Публикует событие
        context.close();
    }
}
```

### 9.2. Пример с POJO-событием и condition

```java
// POJO-событие
public class PaymentProcessed {
    private final String paymentId;
    private final double amount;
    private final String currency;

    public PaymentProcessed(String paymentId, double amount, String currency) {
        this.paymentId = paymentId;
        this.amount = amount;
        this.currency = currency;
    }

    public String getPaymentId() { return paymentId; }
    public double getAmount() { return amount; }
    public String getCurrency() { return currency; }
}

// Сервис
@Service
public class PaymentService {
    private final ApplicationEventPublisher publisher;

    @Autowired
    public PaymentService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void processPayment(String paymentId, double amount, String currency) {
        publisher.publishEvent(new PaymentProcessed(paymentId, amount, currency));
    }
}

// Слушатель
@Service
public class AccountingService {
    @EventListener(condition = "#event.amount > 100 and #event.currency == 'USD'")
    public void recordPayment(PaymentProcessed event) {
        System.out.println("Recording USD payment: " + event.getPaymentId());
    }
}
```

---

## 10. Рекомендации и ограничения

### 10.1. Рекомендации

- **Используйте `@EventListener`**: Более гибкий и современный подход по сравнению с `ApplicationListener`.
- **Применяйте `@TransactionalEventListener` для транзакций**: Гарантирует обработку после коммита.
- **Контролируйте порядок**: Используйте `@Order` для упорядочивания слушателей.
- **Добавляйте асинхронность**: `@EnableAsync` и `@Async` для долгих операций.
- **Создавайте кастомные события**: Наследуйте `ApplicationEvent` или используйте POJO для простоты.
- **Логируйте события**: Добавляйте аудит для отладки.
- **Используйте SpEL в condition**: Фильтруйте события по свойствам, `Environment` или бинам.
- **Проверяйте null в SpEL**: Используйте `?.` для безопасного доступа в `condition`.

### 10.2. Ограничения

- **Синхронность по умолчанию**: Без `@Async` события обрабатываются в том же потоке.
- **Циклические зависимости**: Публикация события в конструкторе может вызвать проблемы.
- **Транзакции**: `@TransactionalEventListener` требует активной транзакции, если `fallbackExecution = false`.
- **Ошибки**: Исключения в слушателях могут прервать обработку, если не настроен `AsyncUncaughtExceptionHandler`.
- **Ограничения condition**: Сложные SpEL-выражения могут снижать производительность и усложнять отладку.

---

**Источники**:

- Spring Framework Documentation (version 6.x).
- JSR-250: Common Annotations for the Java Platform.