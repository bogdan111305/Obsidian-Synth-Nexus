# Spring IoC (Inversion of Control)

**Дата создания:** 2022-09-03  
**Теги:** #springbase #spring #dmdev

## Введение

Spring Framework во многом реализует спецификации **JavaEE** и предоставляет мощную инфраструктуру для разработки Java приложений.

## Inversion of Control (IoC)

**Inversion of Control** - это принцип программирования, при котором управление выполнением программы передаётся фреймворку, а не программисту.

### Основные концепции

#### IoC в Spring
В Spring реализацией **IoC** выступает механизм **Dependency Injection (DI)**.

#### Bean Definition
Необходимо предоставить какое-то количество **Bean Definition**, исходя из которых IoC контейнер будет создавать **Bean**.

#### Bean
**Bean** - это объект, который создан Spring контейнером (context), исходя из определения Bean Definition.

#### IoC Container
IoC container представляет собой реализацию **BeanFactory** либо **ApplicationContext**.

![[Pasted image 20220903210304.png]]

## Способы определения Bean Definitions

### 1. XML-based конфигурация
```xml
<bean id="userService" class="com.example.UserService">
    <property name="userRepository" ref="userRepository"/>
</bean>
```

### 2. Annotation-based конфигурация
```java
@Component
public class UserService {
    @Autowired
    private UserRepository userRepository;
}
```

### 3. Java-based конфигурация
```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService();
    }
}
```

## Преимущества IoC

1. **Слабая связанность** - компоненты не зависят от конкретных реализаций
2. **Тестируемость** - легко подменять зависимости для тестов
3. **Гибкость** - можно легко изменить поведение без изменения кода
4. **Переиспользование** - компоненты можно использовать в разных контекстах

## Связи с другими темами

- [[Dependency Injection]] - детальное рассмотрение DI
- [[Bean Lifecycle]] - жизненный цикл бинов
- [[Spring Configuration]] - способы конфигурации
- [[Spring Annotations]] - аннотации для IoC 