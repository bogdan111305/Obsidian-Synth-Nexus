# @Autowired, @Value и @Resource

**Дата создания:** 2022-09-26  
**Теги:** #spring #springbase #dmdev

## Обзор аннотаций для Dependency Injection

В Spring Framework существует несколько аннотаций для внедрения зависимостей. Каждая имеет свои особенности и случаи применения.

![[Pasted image 20220926221949.png]]

## @Resource

![[Pasted image 20220926223202.png]]

### Описание
Аннотация `@Resource` из пакета `javax.annotation`, который не входит в Spring Core.

### Применение
Можно ставить над:
- **Полем**
- **Методом**

**Важно:** нельзя поставить перед конструктором!

### Отличие от @Autowired
Может инжектить в зависимости от названия бина, что может быть удобно при неопределённости.

### Пример использования
```java
@Resource(name = "myBean")  
private SomeBean someBean;
```

### Назначение
Эта аннотация нужна только для того, чтобы удовлетворять Java EE спецификации **JSR-250**.

**На практике чаще используется именно Spring аннотация `@Autowired`.**

## @Autowired

### Особенности
- Можно ставить даже над другой аннотацией
- При инжекте полагается на **тип поля**, а не на его имя

### При неопределённости
- Поле `required` - фэйлить ли контекст если не найдётся бин
- Для указания конкретного бина придётся указать `@Qualifier`
- `@Autowired` заинжектит если id бина и имя поля совпадает

### Примеры

#### Базовое использование
```java
@Autowired
private UserService userService;
```

#### С required = false
```java
@Autowired(required = false)  
private SomeBean someBean;
```

#### При двух бинах SomeBean
```java
// При двух бинах SomeBean @Autowired заинжектит тот, id которого someBean  
@Autowired(required = false)  
private SomeBean someBean;
```

#### Инжект коллекций
`@Autowired` может инжектить целые коллекции:

```java
@Autowired
private List<UserService> userServices;

@Autowired
private Map<String, UserService> userServiceMap;
```

#### Инжект в конструктор
```java
@Component
public class UserController {
    private final UserService userService;
    
    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

## @Value

### Описание
Аннотация для внедрения значений из properties файлов или SpEL выражений.

### Примеры использования

#### Простое значение
```java
@Value("${someValue}")  
private String name;
```

#### SpEL выражение
```java
@Value("#{systemProperties['user.home']}")
private String userHome;

@Value("#{userService.getDefaultUser()}")
private User defaultUser;
```

#### Значение по умолчанию
```java
@Value("${app.name:DefaultAppName}")
private String appName;
```

#### Числовые значения
```java
@Value("${server.port:8080}")
private int serverPort;

@Value("${cache.timeout:300}")
private long cacheTimeout;
```

## Сравнение аннотаций

| Аннотация | Источник | Тип поиска | JSR-250 | Spring |
|-----------|----------|------------|---------|--------|
| `@Autowired` | Spring | По типу | ❌ | ✅ |
| `@Resource` | JSR-250 | По имени | ✅ | ❌ |
| `@Value` | Spring | Properties/SpEL | ❌ | ✅ |

## Лучшие практики

1. **Используйте `@Autowired`** для Spring-специфичного кода
2. **Используйте `@Resource`** только для совместимости с JSR-250
3. **Используйте `@Value`** для конфигурационных значений
4. **Всегда указывайте `required = false`** для опциональных зависимостей
5. **Используйте `@Qualifier`** для разрешения конфликтов

## Связи с другими темами

- [[Spring IoC]] - основы IoC контейнера
- [[Bean Lifecycle]] - жизненный цикл бинов
- [[Spring Configuration]] - конфигурация приложения
- [[Properties и YAML]] - работа с конфигурацией 