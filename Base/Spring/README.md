# Spring Framework

Полное руководство по Spring Framework, включая основы, продвинутые темы, безопасность, работу с данными и практические примеры.

## Содержание

### 1. Основы Spring ✅
- **[README.md](./1. Основы Spring/README.md)** - Обзор основ Spring
- **1.1 IoC и Dependency Injection** - Принципы IoC и DI
- **1.2 Аннотации Spring** - Основные аннотации Spring
- **1.3 Конфигурация** - Способы конфигурации приложений
- **1.4 Spring Boot** - Быстрая разработка с Spring Boot

### 2. Продвинутые темы Spring 📋
- **[README.md](./2. Продвинутые темы Spring/README.md)** - Обзор продвинутых тем
- **2.1 BeanPostProcessor** - Кастомизация жизненного цикла бинов
- **2.2 Event Listeners** - Система событий Spring
- **2.3 Interceptors** - Перехватчики запросов
- **2.4 SpEL и Environment** - Expression Language и окружение

### 3. Spring Security 📋
- **[README.md](./3. Spring Security/README.md)** - Обзор безопасности
- **3.1 Основы безопасности** - Аутентификация и авторизация
- **3.2 JWT и сессии** - Работа с токенами
- **3.3 OAuth2** - OAuth2 интеграция

### 4. Spring Data и кэширование 📋
- **[README.md](./4. Spring Data и кэширование/README.md)** - Обзор работы с данными
- **4.1 JPA и Hibernate** - ORM и работа с БД
- **4.2 Repository Pattern** - Паттерн репозитория
- **4.3 Transactions** - Управление транзакциями
- **4.4 Second Level Cache** - Кэширование второго уровня

### 5. Практика и паттерны 📋
- **[README.md](./5. Практика и паттерны/README.md)** - Обзор практик
- **5.1 Spring Patterns** - Паттерны проектирования
- **5.2 Интеграция** - Интеграция с внешними системами
- **5.3 Практика** - Практические примеры

## Статус разработки

### ✅ Завершено
- **1. Основы Spring** - Полностью завершен
  - IoC и Dependency Injection (4 файла)
  - Аннотации Spring (5 файлов)
  - Конфигурация (3 файла)
  - Spring Boot (2 файла)

### 🔄 В процессе
- Дополнительные примеры и практические задания
- Интеграция с другими модулями

### 📋 Планируется
- **2. Продвинутые темы Spring**
- **3. Spring Security**
- **4. Spring Data и кэширование**
- **5. Практика и паттерны**

## Ключевые концепции

### IoC (Inversion of Control)
Принцип инверсии управления, при котором контроль над созданием и управлением объектами передается фреймворку.

### Dependency Injection
Реализация IoC, при которой зависимости объекта предоставляются извне.

### Spring Boot
Фреймворк для быстрого создания Spring приложений с минимальной конфигурацией.

### Spring Security
Фреймворк для обеспечения безопасности в Spring приложениях.

### Spring Data
Упрощение работы с различными источниками данных.

## Быстрый старт

### Создание Spring Boot приложения

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

### Основная конфигурация

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {
    
    @Bean
    public UserService userService() {
        return new UserService();
    }
}
```

### Использование аннотаций

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Value("${app.name}")
    private String appName;
}
```

## Связанные разделы

- [Java](../Java/README.md) - Основы Java
- [Hibernate](../Hibernate/README.md) - ORM фреймворк
- [JDBC](../JDBC/README.md) - Работа с базами данных 