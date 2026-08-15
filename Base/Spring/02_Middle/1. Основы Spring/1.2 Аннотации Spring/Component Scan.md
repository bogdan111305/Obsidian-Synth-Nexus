# Component Scan

**Дата создания:** 2022-09-27  
**Теги:** #spring #springbase #dmdev #springXML

## Обзор

Component Scan — это механизм Spring Framework для автоматического обнаружения и регистрации бинов с аннотациями (`@Component`, `@Service`, `@Repository`, `@Controller`) в указанных пакетах.

![[Pasted image 20220927220235.png]]

## Основные концепции

### Необходимость указания пакета
Необходимо указать пакет, где будут сканироваться бины.

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {
    // Конфигурация
}
```

## Bean Definition Readers

Классы, которые занимаются считыванием bean definitions в зависимости от типа конфигурации:

![[Pasted image 20220927220428.png]]

По сути они и определяют 3 вида конфигурации: XMLBased, AnnotationBased и JavaBased.

### Основные читатели

**BeanDefinitionReader** реализуют 3 основных класса:

- **XMLBeanDefinitionReader** - парсит XML файл и считывает definitions бинов
- **GroovyBeanDefinitionReader** - парсит Groovy скрипт и считывает его definitions
- **PropertiesBeanDefinitionReader** - *deprecated* раньше бины можно было прописывать даже в property файле

### Сканеры и читатели

**ClassPathBeanDefinitionScanner** и **AnnotatedBeanDefinitionReader**:

- Благодаря сканеру появляется возможность указать шаблон пути к нашим классам, которые будут bean definitions
- С помощью AnnotatedBeanDefinitionReader можно зарегистрировать бины вручную

**ConfigurationClassBeanDefinitionReader**:

- Конфигурация с помощью аннотаций `@Configuration` и `@Bean`

## Механизм работы

ComponentScan использует под капотом не только рефлексию, но и IO, так как у рефлексии нет функционала поиска классов по пакетам. Тут включается поиск с помощью IO на жёстком диске по указанному шаблону classpath, который мы передали.

### Внутренняя реализация

```java
@Override  
@Nullable  
public BeanDefinition parse(Element element, ParserContext parserContext) {  
   String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);  
   basePackage = parserContext.getReaderContext()
       .getEnvironment().resolvePlaceholders(basePackage);  
   String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,  
         ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);  
  
   // Actually scan for bean definitions and register them.  
   ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);  
   Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);  
   registerComponents(parserContext.getReaderContext(), beanDefinitions, element);  
  
   return null;
}
```

## TypeFilters

![[Pasted image 20220929215245.png]]

### Примеры фильтров

Пример добавления фильтров. Для эксперимента мы отключили дефолтный фильтр:

![[Pasted image 20220929215804.png]]

#### Типы фильтров

- **annotation** - создали фильтр, который забирает классы, аннотированные `@Component` (по сути вернули дефолтный)
- **assignable** - указали интерфейс, реализации которого будет искать фильтр
- **regex** - указали регулярку, по которой будут искаться классы фильтром

#### Scoped Proxy

*scoped-proxy* - создавать ли proxy на основании классов (по умолчанию no)

## Примеры использования

### 1. Базовое сканирование

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {
}
```

### 2. Сканирование с фильтрами

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @Filter(type = FilterType.ANNOTATION, classes = Service.class),
    excludeFilters = @Filter(type = FilterType.ANNOTATION, classes = Repository.class)
)
public class AppConfig {
}
```

### 3. Сканирование по типу

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @Filter(type = FilterType.ASSIGNABLE_TYPE, classes = OrderService.class)
)
public class AppConfig {
}
```

### 4. Кастомный фильтр

```java
public class ServiceTypeFilter implements TypeFilter {
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory factory) {
        return metadataReader.getClassMetadata().getClassName().contains("Service");
    }
}

@Configuration
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @Filter(type = FilterType.CUSTOM, classes = ServiceTypeFilter.class)
)
public class AppConfig {
}
```

### 5. Сканирование с исключениями

```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = {
        @Filter(type = FilterType.REGEX, pattern = ".*Test.*"),
        @Filter(type = FilterType.ANNOTATION, classes = Configuration.class)
    }
)
public class AppConfig {
}
```

## Аннотации для сканирования

### @ComponentScan
Основная аннотация для включения сканирования компонентов.

```java
@ComponentScan(
    basePackages = "com.example",
    basePackageClasses = {UserService.class, OrderService.class},
    nameGenerator = CustomBeanNameGenerator.class,
    scopeResolver = CustomScopeResolver.class,
    scopedProxy = ScopedProxyMode.TARGET_CLASS
)
```

### @SpringBootApplication
Комбинирует `@Configuration`, `@EnableAutoConfiguration` и `@ComponentScan`.

```java
@SpringBootApplication(scanBasePackages = "com.example")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## Лучшие практики

1. **Указывайте конкретные пакеты** - избегайте сканирования всего classpath
2. **Используйте фильтры** - для точного контроля над сканированием
3. **Исключайте тестовые классы** - используйте excludeFilters
4. **Группируйте компоненты** - по функциональности в пакеты
5. **Используйте basePackageClasses** - для типобезопасности

## Производительность

- **Ленивое сканирование** - Spring Boot оптимизирует сканирование
- **Кэширование** - результаты сканирования кэшируются
- **Параллельное сканирование** - в Spring Boot 2.0+

## Связи с другими темами

- [[Spring Annotations]] - Аннотации для компонентов
- [[Spring Configuration]] - Способы конфигурации
- [[Spring Boot]] - Автоконфигурация
- [[BeanPostProcessor]] - Расширение функциональности 