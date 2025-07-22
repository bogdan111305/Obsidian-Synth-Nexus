## 1. Inversion of Control (IoC)

**Inversion of Control (IoC)** — принцип проектирования, при котором управление созданием и жизненным циклом объектов (зависимостей) передаётся внешнему контейнеру, а не реализуется внутри классов.

### 1.1. Основные аспекты IoC

- **Передача контроля**: Классы не создают зависимости, а получают их от контейнера.
- **Гибкость**: Упрощает замену реализаций зависимостей без изменения кода.
- **Тестируемость**: Облегчает подмену зависимостей в тестах (например, с помощью мок-объектов).
- **Модульность**: Разделяет логику создания объектов и их использования.

### 1.2. Пример без IoC

```java
public class OrderService {
    private DatabaseRepository repository = new DatabaseRepositoryImpl(); // Жёсткая зависимость

    public Order findOrder(int id) {
        return repository.findById(id);
    }
}
```

**Проблемы**:

- Жёсткая связь между `OrderService` и `DatabaseRepositoryImpl`.
- Сложно заменить реализацию `DatabaseRepository` (например, на тестовую).
- Усложняется тестирование и поддержка.

### 1.3. Пример с IoC

```java
public class OrderService {
    private final DatabaseRepository repository;

    public OrderService(DatabaseRepository repository) {
        this.repository = repository; // Внедрение зависимости
    }

    public Order findOrder(int id) {
        return repository.findById(id);
    }
}
```

**Преимущества**:

- Зависимость передаётся через конструктор.
- Легко заменить `DatabaseRepository` на другую реализацию.

---

## 2. Dependency Injection (DI)

**Dependency Injection (DI)** — реализация IoC, при которой зависимости объекта внедряются извне, а не создаются внутри него. Spring использует DI как основной механизм управления зависимостями.

### 2.1. Типы DI

1. **Конструкторное внедрение**:
    
    - Зависимости передаются через конструктор.
    - Рекомендуется для обязательных зависимостей.
    
    ```java
    @Component
    public class OrderService {
        private final DatabaseRepository repository;
    
        @Autowired
        public OrderService(DatabaseRepository repository) {
            this.repository = repository;
        }
    }
    ```
    
2. **Сеттерное внедрение**:
    
    - Зависимости устанавливаются через сеттеры.
    - Подходит для необязательных зависимостей.
    
    ```java
    @Component
    public class OrderService {
        private DatabaseRepository repository;
    
        @Autowired
        public void setRepository(DatabaseRepository repository) {
            this.repository = repository;
        }
    }
    ```
    
3. **Внедрение через поля**:
    
    - Зависимости внедряются напрямую в поля через `@Autowired`.
    - Менее предпочтительно из-за сложностей с тестированием.
    
    ```java
    @Component
    public class OrderService {
        @Autowired
        private DatabaseRepository repository;
    }
    ```
    

### 2.2. Преимущества DI

- Снижение связанности между классами.
- Упрощение тестирования через подмену зависимостей.
- Гибкость замены компонентов (например, смена базы данных).

---

## 3. IoC Container

**IoC Container** — ядро Spring, управляющее созданием, конфигурацией и жизненным циклом бинов. Он реализует IoC через DI, обеспечивая регистрацию, создание и управление зависимостями.

### 3.1. Типы контейнеров

1. **BeanFactory**:
    - Базовый контейнер, предоставляющий функции IoC.
    - Ленивая инициализация (бины создаются по запросу).
    - Используется редко в современных приложениях.
2. **ApplicationContext**:
    - Расширенная версия `BeanFactory`.
    - Поддерживает автоматическую инициализацию, события, интернационализацию, `Environment`.
    - Основной выбор для большинства приложений.

### 3.2. Основные интерфейсы

- `org.springframework.beans.factory.BeanFactory`: Базовый доступ к бинам (`getBean`).
- `org.springframework.context.ApplicationContext`: Расширенные функции (события, `Environment`).
- `org.springframework.context.ConfigurableApplicationContext`: Методы конфигурации (`refresh`, `close`).

### 3.3. Реализации `ApplicationContext`

- `ClassPathXmlApplicationContext`: Загрузка конфигурации из XML в classpath.
- `FileSystemXmlApplicationContext`: Загрузка XML из файловой системы.
- `AnnotationConfigApplicationContext`: Конфигурация через аннотации и Java.
- `GenericApplicationContext`: Универсальный контекст для программной настройки.
- `XmlWebApplicationContext`: Для веб-приложений (Spring MVC).

---

## 4. Жизненный цикл ApplicationContext

**Описание**: Жизненный цикл `ApplicationContext` охватывает создание, настройку, использование и уничтожение контейнера, включая управление бинами.

### 4.1. Этапы жизненного цикла

1. **Создание контекста**:
    - Инициализация реализации (`AnnotationConfigApplicationContext`, `ClassPathXmlApplicationContext`).
    - Создание `DefaultListableBeanFactory` для хранения `BeanDefinition` и бинов.
2. **Загрузка BeanDefinition**:
    - Чтение конфигурации (XML, Groovy, properties, аннотации) через `BeanDefinitionReader`.
3. **Настройка Environment**:
    - Загрузка свойств (`application.properties`) и активация профилей.
4. **Компонент-сканирование**:
    - Поиск бинов с аннотациями (`@Component`, `@Service`) через `@ComponentScan`.
5. **Подготовка BeanFactory**:
    - Выполнение `BeanFactoryPostProcessor` для модификации `BeanDefinition`.
6. **Создание и конфигурация бинов**:
    - Инстанцирование, внедрение зависимостей, применение `BeanPostProcessor`.
7. **Инициализация бинов**:
    - Вызов методов `@PostConstruct`, `InitializingBean`, `init-method`.
8. **Готовность контекста**:
    - Контекст готов, бины доступны через `getBean` или DI.
9. **Уничтожение**:
    - Вызов методов `@PreDestroy`, `DisposableBean`, `destroy-method`.
    - Закрытие контекста через `close()` или `registerShutdownHook()`.

### 4.2. Пример жизненного цикла

```java
ConfigurableApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
context.refresh(); // Запуск жизненного цикла
OrderService service = context.getBean(OrderService.class);
context.close(); // Завершение
```
## 5. Создание ApplicationContext

### 5.1. Типы контекстов

|**Тип контекста**|**Класс**|**Описание**|**Пример использования**|
|---|---|---|---|
|`GenericApplicationContext`|`org.springframework.context.support.GenericApplicationContext`|Универсальный контекст для программной конфигурации (XML, Groovy, Java).|Программная настройка, Groovy DSL.|
|`ClassPathXmlApplicationContext`|`org.springframework.context.support.ClassPathXmlApplicationContext`|Загружает XML из classpath.|Традиционные приложения с XML.|
|`FileSystemXmlApplicationContext`|`org.springframework.context.support.FileSystemXmlApplicationContext`|Загружает XML из файловой системы.|Конфигурации вне JAR (CI/CD).|
|`AnnotationConfigApplicationContext`|`org.springframework.context.annotation.AnnotationConfigApplicationContext`|Для Java/аннотаций (`@Configuration`, `@Bean`).|Spring Boot, современные приложения.|
|`XmlWebApplicationContext`|`org.springframework.web.context.support.XmlWebApplicationContext`|Для веб-приложений (Spring MVC).|REST API, веб-интерфейсы.|

### 5.2. Примеры создания

- **XML (ClassPathXmlApplicationContext)**:

```java
ConfigurableApplicationContext context = new ClassPathXmlApplicationContext("classpath:beans.xml");
context.refresh();
```

- **XML (FileSystemXmlApplicationContext)**:

```java
ConfigurableApplicationContext context = new FileSystemXmlApplicationContext("file:/config/beans.xml");
context.refresh();
```

- **Java Config (AnnotationConfigApplicationContext)**:

```java
@Configuration
public class AppConfig {
    @Bean
    public DatabaseRepository repository() {
        return new DatabaseRepositoryImpl();
    }

    @Bean
    @Scope("prototype")
    public OrderService orderService(DatabaseRepository repository) {
        return new OrderServiceImpl(repository);
    }
}
```

```java
ConfigurableApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
```

- **Groovy (GenericApplicationContext)**:

```groovy
beans {
    repository(com.example.DatabaseRepositoryImpl)
    orderService(com.example.OrderServiceImpl) {
        scope = 'prototype'
        repository = ref('repository')
    }
}
```

```java
GenericApplicationContext context = new GenericApplicationContext();
GroovyBeanDefinitionReader reader = new GroovyBeanDefinitionReader(context);
reader.loadBeanDefinitions("classpath:beans.groovy");
context.refresh();
```

### 5.3. Особенности

- **Инициализация BeanFactory**: Контекст создаёт `DefaultListableBeanFactory` для управления `BeanDefinition` и бинами.
- **Иерархия контекстов**: Поддерживает родительские и дочерние контексты.

```java
GenericApplicationContext parent = new GenericApplicationContext();
AnnotationConfigApplicationContext child = new AnnotationConfigApplicationContext();
child.setParent(parent);
child.register(AppConfig.class);
child.refresh();
```

- **Ошибки**: Некорректная конфигурация вызывает `BeanDefinitionParsingException`.
- **Spring Boot**: Автоматический вызов `refresh()` через `SpringApplication.run()`.

---

## 6. Чтение BeanDefinition

**Описание**: `BeanDefinition` — объект, содержащий метаданные бина (класс, свойства, зависимости, scope, методы инициализации/уничтожения). Spring загружает `BeanDefinition` через читателей, соответствующих формату конфигурации.

### 6.1. Читатели BeanDefinition

|**Читатель**|**Класс**|**Формат**|**Описание**|
|---|---|---|---|
|`XmlBeanDefinitionReader`|`org.springframework.beans.factory.xml.XmlBeanDefinitionReader`|XML|Парсит `<bean>` и `<alias>` в XML.|
|`GroovyBeanDefinitionReader`|`org.springframework.beans.factory.groovy.GroovyBeanDefinitionReader`|Groovy|Парсит Groovy DSL (`beans { ... }`).|
|`PropertiesBeanDefinitionReader`|`org.springframework.beans.factory.support.PropertiesBeanDefinitionReader`|Properties|Парсит `.properties` файлы.|
|`AnnotatedBeanDefinitionReader`|`org.springframework.context.annotation.AnnotatedBeanDefinitionReader`|Аннотации|Обрабатывает `@Configuration`, `@Bean`.|

**Иерархия**:

- Читатели реализуют `BeanDefinitionReader`.
- `BeanDefinitionRegistry` (в `DefaultListableBeanFactory`) хранит `BeanDefinition`.

### 6.2. Примеры

- **XML**:

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="repository" class="com.example.DatabaseRepositoryImpl"/>
    <bean id="orderService" class="com.example.OrderServiceImpl" scope="prototype">
        <constructor-arg ref="repository"/>
    </bean>
</beans>
```

```java
GenericApplicationContext context = new GenericApplicationContext();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(context);
reader.loadBeanDefinitions("classpath:beans.xml");
context.refresh();
```

- **Groovy**:

```groovy
beans {
    repository(com.example.DatabaseRepositoryImpl)
    orderService(com.example.OrderServiceImpl) {
        scope = 'prototype'
        repository = ref('repository')
    }
}
```

```java
GenericApplicationContext context = new GenericApplicationContext();
GroovyBeanDefinitionReader reader = new GroovyBeanDefinitionReader(context);
reader.loadBeanDefinitions("classpath:beans.groovy");
context.refresh();
```

- **Properties**:

```properties
repository.(class)=com.example.DatabaseRepositoryImpl
orderService.(class)=com.example.OrderServiceImpl
orderService.scope=prototype
orderService.repository(ref)=repository
```

```java
GenericApplicationContext context = new GenericApplicationContext();
PropertiesBeanDefinitionReader reader = new PropertiesBeanDefinitionReader(context);
reader.loadBeanDefinitions("classpath:beans.properties");
context.refresh();
```

- **Аннотации**:

```java
@Configuration
public class AppConfig {
    @Bean
    public DatabaseRepository repository() {
        return new DatabaseRepositoryImpl();
    }

    @Bean
    @Scope("prototype")
    public OrderService orderService(DatabaseRepository repository) {
        return new OrderServiceImpl(repository);
    }
}
```

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
```

### 6.3. Механизмы

- **XML**: Парсер (DOM/SAX) преобразует `<bean>` в `GenericBeanDefinition`.
- **Groovy**: Groovy DSL динамически создаёт `BeanDefinition`.
- **Properties**: Парсит ключи в формате `beanName.(property)`.
- **Аннотации**: `AnnotatedBeanDefinitionReader` обрабатывает `@Bean` методы.
- **Ошибки**: `BeanDefinitionParsingException` при некорректной конфигурации.
- **Алиасы**: Поддерживаются (`<alias>`, `name = bean`, `@Bean(name = "alias")`).

---

## 7. Компонент-сканирование и TypeFilter

**Описание**: Компонент-сканирование автоматически регистрирует бины с аннотациями (`@Component`, `@Service`, `@Repository`, `@Controller`). `TypeFilter` фильтрует классы при сканировании.

### 7.1. Компонент-сканирование

- Аннотация: `@ComponentScan`.
- Параметры:
    - `basePackages`: Пакеты для сканирования.
    - `includeFilters`/`excludeFilters`: Фильтры классов.
- Механизм: `ClassPathBeanDefinitionScanner` сканирует классы, создавая `ScannedGenericBeanDefinition`.

**Пример**:

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {
}
```

```java
@Service
public class OrderServiceImpl implements OrderService {
    private final DatabaseRepository repository;

    public OrderServiceImpl(DatabaseRepository repository) {
        this.repository = repository;
    }
}
```

### 7.2. TypeFilter

|**Фильтр**|**Класс**|**Описание**|
|---|---|---|
|`AnnotationTypeFilter`|`org.springframework.core.type.filter.AnnotationTypeFilter`|Фильтр по аннотациям (`@Service`).|
|`AssignableTypeFilter`|`org.springframework.core.type.filter.AssignableTypeFilter`|Фильтр по типу (например, `OrderService.class`).|
|`RegexPatternTypeFilter`|`org.springframework.core.type.filter.RegexPatternTypeFilter`|Фильтр по регулярным выражениям.|
|`CustomTypeFilter`|Реализация `org.springframework.core.type.filter.TypeFilter`|Пользовательский фильтр.|

**Пример**:

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @Filter(type = FilterType.ASSIGNABLE_TYPE, classes = OrderService.class),
    excludeFilters = @Filter(type = FilterType.ANNOTATION, classes = Repository.class)
)
public class AppConfig {
}
```

**Кастомный TypeFilter**:

```java
public class ServiceTypeFilter implements TypeFilter {
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory factory) {
        return metadataReader.getClassMetadata().getClassName().contains("Service");
    }
}
```

```java
@ComponentScan(basePackages = "com.example", includeFilters = @Filter(type = FilterType.CUSTOM, classes = ServiceTypeFilter.class))
```

### 7.3. Механизмы

- Используется `ClassPathScanningCandidateComponentProvider`.
- Аннотации обрабатываются через `AnnotationMetadata`.
- Поддерживает JAR-файлы и иерархию пакетов.
- Оптимизировано в Spring Boot через `@SpringBootApplication`.

---

## 8. Environment и @Profile

**Описание**: `Environment` предоставляет доступ к свойствам системы и приложения. `@Profile` активирует бины для определённых окружений.

### 8.1. Environment

- Интерфейс: `org.springframework.core.env.Environment`.
- Реализации: `StandardEnvironment`, `StandardServletEnvironment` (веб).
- Источники: `application.properties`, системные переменные, переменные окружения.
- Доступ: `Environment.getProperty`, `@Value`.

**Пример (EnvironmentAware)**:

```java
@Service
public class OrderServiceImpl implements OrderService, EnvironmentAware {
    private Environment env;

    @Override
    public void setEnvironment(Environment env) {
        this.env = env;
    }

    public String getConfig() {
        return env.getProperty("app.config", "default");
    }
}
```

**Пример (@Value)**:

```java
@Service
public class NotificationService {
    @Value("${app.channels:email,sms}")
    private List<String> channels;

    @Value("#{${app.configMap}}")
    private Map<String, String> configMap;
}
```

```properties
app.config=production
app.channels=email,sms
app.configMap={timeout: '5000', retry: '3'}
```

### 8.2. @Profile

- Указывает окружение для активации бинов.
- Активация: `-Dspring.profiles.active=dev` или `context.getEnvironment().setActiveProfiles("dev")`.

**Пример**:

```java
@Service
@Profile("dev")
public class DevOrderService implements OrderService {
    public Order findById(int id) {
        return new Order(id, "DevOrder");
    }
}

@Service
@Profile("prod")
public class ProdOrderService implements OrderService {
    public Order findById(int id) {
        return new Order(id, "ProdOrder");
    }
}
```

**XML**:

```xml
<beans profile="dev">
    <bean id="orderService" class="com.example.DevOrderService"/>
</beans>
```

**Groovy**:

```groovy
beans {
    profile('dev') {
        orderService(com.example.DevOrderService)
    }
}
```

### 8.3. Механизмы

- `PropertySources`: Хранит источники свойств.
- `PropertyResolver`: Разрешает `${...}` и `#{...}`.
- SpEL: Поддерживает динамические выражения в `@Value`.

---

## 9. Подготовка BeanFactory

**Описание**: После чтения `BeanDefinition` Spring подготавливает `BeanFactory`, вызывая `BeanFactoryPostProcessor` для модификации метаданных бинов.

### 9.1. BeanFactoryPostProcessor

- Интерфейс: `org.springframework.beans.factory.config.BeanFactoryPostProcessor`.
- Метод: `postProcessBeanFactory(ConfigurableListableBeanFactory)`.
- Вызывается после загрузки `BeanDefinition`, но до создания бинов.

**Реализации**:

- `ConfigurationClassPostProcessor`: Обрабатывает `@Configuration`, `@Bean`, `@Import`.
- `PropertyPlaceholderConfigurer`: Заменяет `${...}` значениями из `Environment`.
- `PropertyOverrideConfigurer`: Переопределяет свойства через `.properties`.
- `CustomEditorConfigurer`: Регистрирует кастомные `PropertyEditor`.

**Пример**:

```java
@Component
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor, Ordered {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        BeanDefinition definition = factory.getBeanDefinition("orderService");
        definition.setAttribute("customAttr", "value");
        definition.getPropertyValues().add("timeout", 1000);
    }

    @Override
    public int getOrder() { return 1; }
}
```

### 9.2. Сортировка

- Через `PriorityOrdered` (высший приоритет, например, `ConfigurationClassPostProcessor`).
- Через `Ordered` (меньшее значение `getOrder()` — раньше выполнение).
- Без сортировки: в порядке регистрации.

**Пример**:

```java
@Component
@Order(2)
public class FirstProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        System.out.println("FirstProcessor");
    }
}

@Component
@Order(1)
public class SecondProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        System.out.println("SecondProcessor");
    }
}
```

### 9.3. Spring Expression Language (SpEL)

- Используется для динамической конфигурации в `BeanFactoryPostProcessor`, `@Value`, XML, Groovy.
- Синтаксис: `#{expression}`.
- Поддерживает: доступ к `Environment`, `systemProperties`, бинам, коллекциям.

**Пример (SpEL)**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Value("#{systemProperties['app.timeout'] ?: 1000}")
    private int timeout;

    @Value("${app.channels:email,sms}")
    private List<String> channels;
}
```

**XML**:

```xml
<bean id="orderService" class="com.example.OrderServiceImpl">
    <property name="timeout" value="#{systemProperties['app.timeout'] ?: 1000}"/>
</bean>
```

**Groovy**:

```groovy
beans {
    orderService(com.example.OrderServiceImpl) {
        timeout = '#{systemProperties["app.timeout"] ?: 1000}'
    }
}
```

**SpEL в BeanFactoryPostProcessor**:

```java
@Component
public class SpELProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        SpelExpressionParser parser = new SpelExpressionParser();
        Expression expr = parser.parseExpression("systemProperties['app.timeout'] ?: 1000");
        Integer timeout = expr.getValue(Integer.class);
        BeanDefinition definition = factory.getBeanDefinition("orderService");
        definition.getPropertyValues().add("timeout", timeout);
    }
}
```

### 9.4. Механизмы

- **Доступ**: Через `ConfigurableListableBeanFactory.getBeanDefinition`.
- **Модификация**: Изменение свойств, атрибутов, регистрация новых `BeanDefinition`.
- **Ошибки**: `BeanDefinitionStoreException` при некорректной модификации.

---

## 10. Создание и конфигурация бинов

**Описание**: Spring создаёт бины, внедряет зависимости и применяет `BeanPostProcessor`.

### 10.1. Способы создания бинов

- **XML**:

```xml
<bean id="orderService" class="com.example.OrderServiceImpl" scope="prototype">
    <constructor-arg ref="repository"/>
</bean>
```

- **Groovy**:

```groovy
beans {
    orderService(com.example.OrderServiceImpl) {
        scope = 'prototype'
        repository = ref('repository')
    }
}
```

- **Properties**:

```properties
orderService.(class)=com.example.OrderServiceImpl
orderService.scope=prototype
orderService.repository(ref)=repository
```

- **Аннотации**:

```java
@Configuration
public class AppConfig {
    @Bean
    @Scope("prototype")
    public OrderService orderService(DatabaseRepository repository) {
        return new OrderServiceImpl(repository);
    }
}
```

### 10.2. Внедрение зависимостей (DI)

- **Конструктор**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    private final DatabaseRepository repository;

    @Autowired
    public OrderServiceImpl(DatabaseRepository repository) {
        this.repository = repository;
    }
}
```

- **Сеттер**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    private DatabaseRepository repository;

    @Autowired
    public void setRepository(DatabaseRepository repository) {
        this.repository = repository;
    }
}
```

- **Поле**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Autowired
    private DatabaseRepository repository;
}
```

- **Коллекции**:

```java
@Service
public class NotificationService {
    private final List<NotificationProvider> providers;

    @Autowired
    public NotificationService(List<NotificationProvider> providers) {
        this.providers = providers;
    }
}
```

**XML для коллекций**:

```xml
<bean id="notificationService" class="com.example.NotificationService">
    <constructor-arg>
        <list>
            <ref bean="emailProvider"/>
            <ref bean="smsProvider"/>
        </list>
    </constructor-arg>
</bean>
```

**Groovy для коллекций**:

```groovy
beans {
    notificationService(com.example.NotificationService) {
        providers = [ref('emailProvider'), ref('smsProvider')]
    }
}
```

### 10.3. Scopes бинов

|**Scope**|**Описание**|
|---|---|
|`singleton`|Один экземпляр на контекст (по умолчанию).|
|`prototype`|Новый экземпляр для каждого `getBean`.|
|`request`|Один экземпляр на HTTP-запрос (веб).|
|`session`|Один экземпляр на HTTP-сессию (веб).|
|`application`|Один экземпляр на `ServletContext` (веб).|
|`websocket`|Один экземпляр на WebSocket-сессию.|

**Пример**:

```java
@Service
@Scope("prototype")
public class OrderServiceImpl implements OrderService { }
```

**Groovy**:

```groovy
beans {
    orderService(com.example.OrderServiceImpl) { scope = 'prototype' }
}
```

### 10.4. BeanPostProcessor

- Методы:
    - `postProcessBeforeInitialization`: До `@PostConstruct`.
    - `postProcessAfterInitialization`: После инициализации.
- Использование: Настройка бинов, внедрение прокси, логирование.

**Пример**:

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof OrderService) {
            System.out.println("Before init: " + beanName);
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof OrderService) {
            System.out.println("After init: " + beanName);
        }
        return bean;
    }
}
```

### 10.5. Прокси и JSR-спецификации

- **Прокси**:
    - **JDK Dynamic Proxy**: Для интерфейсов.
    - **CGLIB**: Для классов.
    - **Режим**: `proxyTargetClass=true` (CGLIB), `false` (JDK).
    - Используется для AOP (`@Transactional`, `@Async`).
- **Подмена бина на прокси**: В `BeanPostProcessor.postProcessAfterInitialization`.

```java
@Component
public class ProxyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof OrderService) {
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    System.out.println("Proxy invoked for: " + method.getName());
                    return method.invoke(bean, args);
                }
            );
        }
        return bean;
    }
}
```

**XML для прокси**:

```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

- **JSR-250**: `@PostConstruct`, `@PreDestroy`.
- **JSR-330**: `@Inject`, `@Named`.

**Пример (@Transactional)**:

```java
@Service
@Transactional
public class OrderServiceImpl implements OrderService {
    public Order findById(int id) {
        return new Order(id, "Order");
    }
}
```

---

## 11. Инициализация бинов

**Описание**: После внедрения зависимостей Spring инициализирует бины, вызывая методы инициализации.

### 11.1. Методы инициализации

- `@PostConstruct` (JSR-250):

```java
@Service
public class OrderServiceImpl implements OrderService {
    @PostConstruct
    public void init() {
        System.out.println("PostConstruct called");
    }
}
```

- `InitializingBean`:

```java
@Service
public class OrderServiceImpl implements OrderService, InitializingBean {
    @Override
    public void afterPropertiesSet() {
        System.out.println("InitializingBean init");
    }
}
```

- `init-method`:

```xml
<bean id="orderService" class="com.example.OrderServiceImpl" init-method="init"/>
```

```groovy
beans {
    orderService(com.example.OrderServiceImpl) { initMethod = 'init' }
}
```

### 11.2. Порядок вызова

1. Внедрение зависимостей.
2. `BeanPostProcessor.postProcessBeforeInitialization`.
3. `@PostConstruct` или `InitializingBean.afterPropertiesSet`.
4. `init-method`.
5. `BeanPostProcessor.postProcessAfterInitialization`.

---

## 12. Готовность контекста и использование бинов

**Описание**: После инициализации контекст готов, бины доступны через `getBean` или DI.

**Пример**:

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService service = context.getBean(OrderService.class);
```

### 12.1. Aware-интерфейсы

|**Интерфейс**|**Доступ**|
|---|---|
|`BeanFactoryAware`|`BeanFactory`|
|`ApplicationContextAware`|`ApplicationContext`|
|`EnvironmentAware`|`Environment`|
|`BeanNameAware`|Имя бина|

**Пример**:

```java
@Service
public class OrderServiceImpl implements OrderService, ApplicationContextAware, BeanNameAware {
    private ApplicationContext context;
    private String beanName;

    @Override
    public void setApplicationContext(ApplicationContext context) {
        this.context = context;
    }

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
    }

    public Object getBean(String name) {
        return context.getBean(name);
    }
}
```

---

## 13. Уничтожение бинов и контекста

**Описание**: При закрытии контекста Spring уничтожает бины, вызывая методы уничтожения.

### 13.1. Методы уничтожения

- `@PreDestroy` (JSR-250):

```java
@Service
public class OrderServiceImpl implements OrderService {
    @PreDestroy
    public void cleanup() {
        System.out.println("PreDestroy called");
    }
}
```

- `DisposableBean`:

```java
@Service
public class OrderServiceImpl implements OrderService, DisposableBean {
    @Override
    public void destroy() {
        System.out.println("DisposableBean destroy");
    }
}
```

- `destroy-method`:

```xml
<bean id="orderService" class="com.example.OrderServiceImpl" destroy-method="cleanup"/>
```

```groovy
beans {
    orderService(com.example.OrderServiceImpl) { destroyMethod = 'cleanup' }
}
```

### 13.2. Порядок вызова

1. `BeanPostProcessor.postProcessBeforeDestruction` (если реализовано).
2. `@PreDestroy` или `DisposableBean.destroy`.
3. `destroy-method`.

### 13.3. Закрытие контекста

- **Методы**:
    - `context.close()`: Явное закрытие.
    - `context.registerShutdownHook()`: Автоматическое закрытие при завершении JVM.
- **Для singleton**: Уничтожение автоматическое.
- **Для prototype**: Требуется ручное уничтожение или кастомный `BeanPostProcessor`.

**Пример**:

```java
ConfigurableApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
context.registerShutdownHook();
```

**Ручное уничтожение prototype**:

```java
DisposableBeanAdapter.destroySingleton("orderService", context.getBean("orderService"));
```

---

## 14. Аннотации Spring: использование и настройка

**Описание**: Spring предоставляет множество аннотаций для конфигурации бинов, DI, компонент-сканирования, профилей и других функций. Аннотации упрощают настройку, заменяя XML-конфигурацию, и поддерживают настройку через свойства.

### 14.1. Основные аннотации и их свойства

|**Аннотация**|**Назначение**|**Свойства**|**Пример**|
|---|---|---|---|
|`@Component`|Обозначает класс как бин для компонент-сканирования.|`value` (имя бина, по умолчанию — имя класса с маленькой буквы).|`@Component("customBean")`|
|`@Service`|Специализация `@Component` для сервисного слоя.|`value` (имя бина).|`@Service("orderService")`|
|`@Repository`|Специализация `@Component` для доступа к данным (DAO).|`value` (имя бина).|`@Repository("dataRepo")`|
|`@Controller`|Специализация `@Component` для контроллеров (Spring MVC).|`value` (имя бина).|`@Controller("webController")`|
|`@RestController`|Комбинирует `@Controller` и `@ResponseBody` для REST API.|`value` (имя бина).|`@RestController("apiController")`|
|`@Configuration`|Обозначает класс как источник `@Bean`-определений.|`proxyBeanMethods` (true/false, включает прокси для `@Bean`, по умолчанию true).|`@Configuration(proxyBeanMethods = false)`|
|`@Bean`|Определяет метод, создающий бин.|`name` (имя бина), `initMethod`, `destroyMethod`.|`@Bean(name = "customBean", initMethod = "init")`|
|`@Autowired`|Внедряет зависимости (конструктор, сеттер, поле).|`required` (true/false, обязательность зависимости, по умолчанию true).|`@Autowired(required = false)`|
|`@Resource`|Внедряет зависимости по имени (JSR-250).|`name` (имя бина), `type` (тип бина).|`@Resource(name = "dataRepo")`|
|`@Inject`|Внедряет зависимости (JSR-330).|Нет свойств.|`@Inject`|
|`@Named`|Аналог `@Component` для JSR-330.|`value` (имя бина).|`@Named("customBean")`|
|`@Value`|Внедряет значения из `Environment` или SpEL.|Нет свойств (значение указывается в аннотации).|`@Value("${app.timeout:1000}")`|
|`@Scope`|Устанавливает область видимости бина.|`value` (scope: `singleton`, `prototype`, etc.), `proxyMode` (NO, INTERFACES, TARGET_CLASS).|`@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)`|
|`@Profile`|Указывает профиль для активации бина.|`value` (имя профиля).|`@Profile("dev")`|
|`@ComponentScan`|Включает компонент-сканирование.|`basePackages`, `includeFilters`, `excludeFilters`, `lazyInit`.|`@ComponentScan(basePackages = "com.example")`|
|`@Import`|Импортирует конфигурации (`@Configuration`).|`value` (классы для импорта).|`@Import(AppConfig.class)`|
|`@PostConstruct`|Метод инициализации (JSR-250).|Нет свойств.|`@PostConstruct`|
|`@PreDestroy`|Метод уничтожения (JSR-250).|Нет свойств.|`@PreDestroy`|
|`@Transactional`|Управляет транзакциями.|`value` (имя менеджера транзакций), `propagation`, `isolation`, `timeout`, `readOnly`, `rollbackOn`.|`@Transactional(readOnly = true)`|
|`@Async`|Асинхронное выполнение метода.|`value` (имя исполнителя).|`@Async("taskExecutor")`|
|`@Conditional`|Условная регистрация бина.|`value` (класс условия).|`@Conditional(OnProdCondition.class)`|

### 14.2. Подробное описание ключевых аннотаций

#### @Component, @Service, @Repository, @Controller, @RestController

- **Назначение**: Помечают классы как бины для компонент-сканирования.
- **Свойства**:
    - `value`: Имя бина (по умолчанию — имя класса с маленькой буквы).
- **Особенности**:
    - `@Service` для бизнес-логики, `@Repository` для DAO (добавляет обработку исключений), `@Controller`/`@RestController` для веб-контроллеров.
- **Пример**:

```java
@Service("orderService")
public class OrderServiceImpl implements OrderService {
    private final DatabaseRepository repository;

    @Autowired
    public OrderServiceImpl(DatabaseRepository repository) {
        this.repository = repository;
    }
}
```

#### @Configuration

- **Назначение**: Определяет класс как источник `@Bean`-определений.
- **Свойства**:
    - `proxyBeanMethods`: Если `true`, `@Bean`-методы проксируются для сохранения зависимостей; если `false`, оптимизирует производительность.
- **Пример**:

```java
@Configuration(proxyBeanMethods = false)
public class AppConfig {
    @Bean
    public DatabaseRepository repository() {
        return new DatabaseRepositoryImpl();
    }
}
```

#### @Bean

- **Назначение**: Определяет метод, возвращающий бин.
- **Свойства**:
    - `name`: Имя бина (массив строк для нескольких имён).
    - `initMethod`: Метод инициализации.
    - `destroyMethod`: Метод уничтожения.
- **Пример**:

```java
@Configuration
public class AppConfig {
    @Bean(name = {"orderService", "serviceAlias"}, initMethod = "init", destroyMethod = "cleanup")
    @Scope("prototype")
    public OrderService orderService(DatabaseRepository repository) {
        return new OrderServiceImpl(repository);
    }
}
```

#### @Autowired

- **Назначение**: Внедряет зависимости (конструктор, сеттер, поле).
- **Свойства**:
    - `required`: Если `false`, зависимость необязательна (по умолчанию `true`).
- **Пример**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Autowired(required = false)
    private Logger logger;
}
```

#### @Resource (JSR-250)

- **Назначение**: Внедряет зависимости по имени бина (в отличие от `@Autowired`, которая ориентируется на тип). Часть JSR-250.
- **Свойства**:
    - `name`: Имя бина (обязательное).
    - `type`: Тип бина (опционально).
- **Особенности**:
    - Используется для точного указания бина, когда несколько бинов соответствуют типу.
    - Работает с полями и сеттерами, не поддерживает конструкторы.
    - Менее гибка, чем `@Autowired`, но полезна для интеграции с JSR-250.
- **Пример**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Resource(name = "dataRepo")
    private DatabaseRepository repository;

    @Resource(name = "dataRepo", type = DatabaseRepositoryImpl.class)
    public void setRepository(DatabaseRepository repository) {
        this.repository = repository;
    }
}
```

**XML-эквивалент**:

```xml
<bean id="orderService" class="com.example.OrderServiceImpl">
    <property name="repository" ref="dataRepo"/>
</bean>
```

#### @Inject (JSR-330)

- **Назначение**: Аналог `@Autowired` для JSR-330, внедряет зависимости по типу.
- **Особенности**: Требует зависимости `javax.inject`.
- **Пример**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Inject
    private DatabaseRepository repository;
}
```

#### @Named (JSR-330)

- **Назначение**: Аналог `@Component` для JSR-330.
- **Свойства**:
    - `value`: Имя бина.
- **Пример**:

```java
@Named("orderService")
public class OrderServiceImpl implements OrderService {
    @Inject
    private DatabaseRepository repository;
}
```

#### @Value

- **Назначение**: Внедряет значения из `Environment` или SpEL.
- **Пример**:

```java
@Service
public class NotificationService {
    @Value("${app.timeout:1000}")
    private int timeout;

    @Value("#{${app.configMap}}")
    private Map<String, String> configMap;
}
```

#### @Scope

- **Назначение**: Устанавливает область видимости бина.
- **Свойства**:
    - `value`: `singleton`, `prototype`, `request`, etc.
    - `proxyMode`: `NO` (без прокси), `INTERFACES` (JDK), `TARGET_CLASS` (CGLIB).
- **Пример**:

```java
@Service
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class OrderServiceImpl implements OrderService { }
```

#### @Profile

- **Назначение**: Активирует бины для определённых профилей.
- **Свойства**:
    - `value`: Имя профиля (массив строк).
- **Пример**:

```java
@Service
@Profile({"dev", "test"})
public class DevOrderService implements OrderService {
    public Order findById(int id) {
        return new Order(id, "DevOrder");
    }
}
```

#### @ComponentScan

- **Назначение**: Включает компонент-сканирование.
- **Свойства**:
    - `basePackages`: Пакеты для сканирования.
    - `includeFilters`/`excludeFilters`: Фильтры (`@Filter`).
    - `lazyInit`: Ленивая инициализация всех бинов.
- **Пример**:

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @Filter(type = FilterType.ANNOTATION, classes = Service.class),
    lazyInit = true
)
public class AppConfig {
}
```

#### @Import

- **Назначение**: Импортирует другие `@Configuration`-классы.
- **Свойства**:
    - `value`: Классы для импорта.
- **Пример**:

```java
@Configuration
@Import({DatabaseConfig.class, SecurityConfig.class})
public class AppConfig {
}
```

#### @PostConstruct и @PreDestroy (JSR-250)

- **Назначение**: Методы инициализации и уничтожения.
- **Пример**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    @PostConstruct
    public void init() {
        System.out.println("Initialized");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Destroyed");
    }
}
```

#### @Transactional

- **Назначение**: Управляет транзакциями.
- **Свойства**:
    - `value`: Имя менеджера транзакций.
    - `propagation`: Поведение транзакции (`REQUIRED`, `NESTED`, etc.).
    - `isolation`: Уровень изоляции (`READ_COMMITTED`, etc.).
    - `timeout`: Тайм-аут в секундах.
    - `readOnly`: Только чтение.
    - `rollbackOn`: Классы исключений для отката.
- **Пример**:

```java
@Service
@Transactional(readOnly = true, propagation = Propagation.REQUIRED)
public class OrderServiceImpl implements OrderService {
    public Order findById(int id) {
        return new Order(id, "Order");
    }
}
```

#### @Async

- **Назначение**: Асинхронное выполнение метода.
- **Свойства**:
    - `value`: Имя `Executor`.
- **Пример**:

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Async("taskExecutor")
    public CompletableFuture<Order> findByIdAsync(int id) {
        return CompletableFuture.completedFuture(new Order(id, "Order"));
    }
}
```

#### @Conditional

- **Назначение**: Условная регистрация бина.
- **Свойства**:
    - `value`: Класс условия (`Condition`).
- **Пример**:

```java
public class OnProdCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return context.getEnvironment().acceptsProfiles(Profiles.of("prod"));
    }
}

@Service
@Conditional(OnProdCondition.class)
public class ProdOrderService implements OrderService {
    public Order findById(int id) {
        return new Order(id, "ProdOrder");
    }
}
```

### 14.3. Механизмы аннотаций

- **Обработка**: Через `ConfigurationClassPostProcessor` для `@Configuration`, `@Bean`, `@ComponentScan`.
- **DI**: Через `AutowiredAnnotationBeanPostProcessor` для `@Autowired`, `@Inject`, `@Resource`, `@Value`.
- **Прокси**: Через `AnnotationAwareAspectJAutoProxyCreator` для `@Transactional`, `@Async`.
- **JSR-250/330**: Поддержка через `CommonAnnotationBeanPostProcessor` (`@Resource`, `@PostConstruct`, `@PreDestroy`) и `javax.inject` (`@Inject`, `@Named`).

### 14.4. Пример комплексной конфигурации

```java
@Configuration
@ComponentScan(basePackages = "com.example")
@Import(DatabaseConfig.class)
public class AppConfig {
    @Bean
    @Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
    public OrderService orderService(DatabaseRepository repository) {
        return new OrderServiceImpl(repository);
    }
}

@Service
@Profile("prod")
public class OrderServiceImpl implements OrderService {
    @Resource(name = "dataRepo")
    private DatabaseRepository repository;

    @Value("${app.timeout:1000}")
    private int timeout;

    @PostConstruct
    public void init() {
        System.out.println("Initialized with timeout: " + timeout);
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Destroyed");
    }

    @Transactional(readOnly = true)
    public Order findById(int id) {
        return new Order(id, "Order");
    }
}
```

---

**Источники**:

- Spring Framework Documentation (version 6.x).
- Groovy Language Documentation.
- JSR-250: Common Annotations for the Java Platform.
- JSR-330: Dependency Injection for Java.