# Spring Core - IoC и Dependency Injection

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

## 2. Dependency Injection (DI)

**Dependency Injection (DI)** — реализация IoC, при которой зависимости объекта внедряются извне, а не создаются внутри него. Spring использует DI как основной механизм управления зависимостями.

### 2.1. Типы DI

#### Конструкторное внедрение
Зависимости передаются через конструктор. Рекомендуется для обязательных зависимостей.

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

#### Сеттерное внедрение
Зависимости устанавливаются через сеттеры. Подходит для необязательных зависимостей.

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

#### Внедрение через поля
Зависимости внедряются напрямую в поля через `@Autowired`. Менее предпочтительно из-за сложностей с тестированием.

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

## 3. IoC Container

**IoC Container** — ядро Spring, управляющее созданием, конфигурацией и жизненным циклом бинов. Он реализует IoC через DI, обеспечивая регистрацию, создание и управление зависимостями.

### 3.1. Типы контейнеров

#### BeanFactory
- Базовый контейнер, предоставляющий функции IoC.
- Ленивая инициализация (бины создаются по запросу).
- Используется редко в современных приложениях.

#### ApplicationContext
- Расширенная версия `BeanFactory`.
- Поддерживает автоматическую инициализацию, события, интернационализацию, `Environment`.
- Основной выбор для большинства приложений.

### 3.2. Основные интерфейсы

- `org.springframework.beans.factory.BeanFactory`: Базовый доступ к бинам (`getBean`).
- `org.springframework.context.ApplicationContext`: Расширенные функции (события, `Environment`).
- `org.springframework.context.ConfigurableApplicationContext`: Методы конфигурации (`refresh`, `close`).

### 3.3. Реализации ApplicationContext

| Тип контекста | Класс | Описание | Пример использования |
|---------------|-------|----------|---------------------|
| `GenericApplicationContext` | `org.springframework.context.support.GenericApplicationContext` | Универсальный контекст для программной конфигурации | Программная настройка, Groovy DSL |
| `ClassPathXmlApplicationContext` | `org.springframework.context.support.ClassPathXmlApplicationContext` | Загружает XML из classpath | Традиционные приложения с XML |
| `FileSystemXmlApplicationContext` | `org.springframework.context.support.FileSystemXmlApplicationContext` | Загружает XML из файловой системы | Конфигурации вне JAR (CI/CD) |
| `AnnotationConfigApplicationContext` | `org.springframework.context.annotation.AnnotationConfigApplicationContext` | Для Java/аннотаций (`@Configuration`, `@Bean`) | Spring Boot, современные приложения |
| `XmlWebApplicationContext` | `org.springframework.web.context.support.XmlWebApplicationContext` | Для веб-приложений (Spring MVC) | REST API, веб-интерфейсы |

## 4. Создание ApplicationContext

### 4.1. Примеры создания

#### XML (ClassPathXmlApplicationContext)
```java
ConfigurableApplicationContext context = new ClassPathXmlApplicationContext("classpath:beans.xml");
context.refresh();
```

#### Java Config (AnnotationConfigApplicationContext)
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

#### Groovy (GenericApplicationContext)
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

## 5. Жизненный цикл ApplicationContext

### 5.1. Этапы жизненного цикла

1. **Создание контекста**: Инициализация реализации
2. **Загрузка BeanDefinition**: Чтение конфигурации
3. **Настройка Environment**: Загрузка свойств и активация профилей
4. **Компонент-сканирование**: Поиск бинов с аннотациями
5. **Подготовка BeanFactory**: Выполнение BeanFactoryPostProcessor
6. **Создание и конфигурация бинов**: Инстанцирование, внедрение зависимостей
7. **Инициализация бинов**: Вызов методов инициализации
8. **Готовность контекста**: Контекст готов, бины доступны
9. **Уничтожение**: Вызов методов уничтожения

### 5.2. Пример жизненного цикла

```java
ConfigurableApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
context.refresh(); // Запуск жизненного цикла
OrderService service = context.getBean(OrderService.class);
context.close(); // Завершение
```

## Связи с другими темами

- [[Bean Lifecycle]] - Детальный жизненный цикл бинов
- [[Spring Annotations]] - Аннотации для DI
- [[Spring Configuration]] - Способы конфигурации
- [[BeanPostProcessor]] - Расширение функциональности 