# Bean Lifecycle (Жизненный цикл бинов)

**Дата создания:** 2022-09-06  
**Теги:** #spring #springbase #dmdev #beanLifecycle

## Обзор жизненного цикла

Жизненный цикл бина в Spring Framework состоит из нескольких этапов, которые выполняются в строгом порядке.

![[Pasted image 20220906222008.png]]

## Этапы жизненного цикла

### 1. Поступление Bean Definitions
**Приходит набор Bean Definitions** в IoC Container.

### 2. BeanFactoryPostProcessors
При попадании в IoC Container начинают отрабатывать **BeanFactoryPostProcessors**.

### 3. Сортировка Bean Definitions
Bean Definitions сортируются, так как одни бины зависят от других.

### 4. Инициализация бинов
По одному с помощью for-each начинается этап инициализации:

#### 4.1 Вызов конструктора
```java
public class UserService {
    public UserService() {
        // Конструктор вызывается первым
    }
}
```

#### 4.2 Вызов сеттеров
После этого можно по сути работать с бином.

#### 4.3 BeanPostProcessor - beforeInitialization()
Bean попадает ко всем определённым **BeanPostProcessor**, где вызывается метод `beforeInitialization()`.

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Логика до инициализации
        return bean;
    }
}
```

#### 4.4 InitializationCallback
Вызывается **InitializationCallback** - это тоже BPP, который идёт последним.

```java
@Component
public class UserService implements InitializingBean {
    @Override
    public void afterPropertiesSet() throws Exception {
        // Инициализация после установки свойств
    }
}
```

#### 4.5 BeanPostProcessor - afterInitialization()
У **BeanPostProcessor** в том же самом порядке вызываются методы `afterInitialization()`.

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Логика после инициализации
        return bean;
    }
}
```

### 5. Готовые бины
Имеем готовые бины, готовые к использованию.

### 6. Уничтожение (для singleton)
Если бин singleton, то перед его стиранием вызовется метод `destroy`.

```java
@Component
public class UserService implements DisposableBean {
    @Override
    public void destroy() throws Exception {
        // Логика уничтожения
    }
}
```

## Практический пример

```java
@Component
public class UserService implements InitializingBean, DisposableBean {
    
    public UserService() {
        System.out.println("1. Конструктор");
    }
    
    @PostConstruct
    public void postConstruct() {
        System.out.println("2. @PostConstruct");
    }
    
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("3. afterPropertiesSet()");
    }
    
    @PreDestroy
    public void preDestroy() {
        System.out.println("4. @PreDestroy");
    }
    
    @Override
    public void destroy() throws Exception {
        System.out.println("5. destroy()");
    }
}
```

## Связи с другими темами

- [[Spring IoC]] - основы IoC контейнера
- [[BeanPostProcessor]] - детали работы BPP
- [[Bean Scopes]] - области видимости бинов
- [[Spring Configuration]] - конфигурация жизненного цикла 