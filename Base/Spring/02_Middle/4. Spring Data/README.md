# Spring Data JPA - Полное руководство

Данная папка содержит полную документацию по Spring Data JPA, реорганизованную для лучшего понимания и изучения.

## 📚 Структура документации

### 1. Основы Spring Data JPA

**Базовые концепции и компоненты**

- **[Repository.md](1.%20Основы%20Spring%20Data%20JPA/Repository.md)** - Иерархия интерфейсов Repository, от базового до JpaRepository
- **[@Query.md](1.%20Основы%20Spring%20Data%20JPA/@Query.md)** - Аннотация @Query для кастомных запросов, JPQL, Native SQL
- **[@Modifying.md](1.%20Основы%20Spring%20Data%20JPA/@Modifying.md)** - Аннотация @Modifying для операций изменения данных

### 2. Data JPA Repositories

**Продвинутые возможности репозиториев**

#### 2.1 Основные интерфейсы
- **[Repository](Data%20JPA%20Repositories/1.%20Repository/)** - Детальное изучение интерфейса Repository
- **[RepositoryQuery](Data%20JPA%20Repositories/2.%20RepositoryQuery/)** - Механизм выполнения запросов
- **[PartTreeJpaQuery](Data%20JPA%20Repositories/3.%20PartTreeJpaQuery/)** - Автоматическое создание запросов по именам методов

#### 2.2 Аннотации для запросов
- **[@Query](Data%20JPA%20Repositories/4.%20@Query/)** - Подробное изучение аннотации @Query
- **[@Modifying](Data%20JPA%20Repositories/5.%20@Modifying/)** - Операции модификации данных

#### 2.3 Пагинация и проекции
- **[Page & Slice](Data%20JPA%20Repositories/6.%20Page%20&%20Slice/)** - Пагинация результатов запросов
- **[Projection](Data%20JPA%20Repositories/9.%20Projection/)** - Проекции для оптимизации запросов

#### 2.4 Оптимизация производительности
- **[@EntityGraph](Data%20JPA%20Repositories/7.%20@EntityGraph/)** - Управление загрузкой связанных сущностей
- **[@Lock & @QueryHints](Data%20JPA%20Repositories/8.%20@Lock%20&%20@QueryHints/)** - Блокировки и подсказки для запросов

#### 2.5 Расширенные возможности
- **[Custom Repository Implementation](Data%20JPA%20Repositories/10.%20Custom%20Repository%20Implementation/)** - Кастомная реализация репозиториев
- **[JPA Auditing](Data%20JPA%20Repositories/11.%20JPA%20Auditing/)** - Аудит изменений данных
- **[Hibernate Envers](Data%20JPA%20Repositories/12.%20Hibernate%20Envers/)** - Версионирование сущностей
- **[Querydsl](Data%20JPA%20Repositories/13.%20Querydsl/)** - Type-safe запросы

### 3. Data JPA Transactions

**Управление транзакциями в Spring Data JPA**

#### 3.1 Базовые концепции
- **[Базовые концепции](Data%20JPA%20Transactions/1.%20Базовые%20концепции/)** - Основы работы с транзакциями
- **[Продвинутые настройки](Data%20JPA%20Transactions/2.%20Продвинутые%20настройки/)** - Настройка поведения транзакций
- **[Тестирование](Data%20JPA%20Transactions/3.%20Тестирование/)** - Тестирование транзакционного кода

## 🚀 Рекомендуемый порядок изучения

### Для начинающих:
1. Начните с **[Repository.md](1.%20Основы%20Spring%20Data%20JPA/Repository.md)** - изучите иерархию интерфейсов
2. Переходите к **[@Query.md](1.%20Основы%20Spring%20Data%20JPA/@Query.md)** - научитесь писать кастомные запросы
3. Изучите **[@Modifying.md](1.%20Основы%20Spring%20Data%20JPA/@Modifying.md)** - операции изменения данных
4. Практикуйтесь с **[PartTreeJpaQuery](Data%20JPA%20Repositories/3.%20PartTreeJpaQuery/)** - автоматические запросы

### Для продвинутых:
1. **[Custom Repository Implementation](Data%20JPA%20Repositories/10.%20Custom%20Repository%20Implementation/)** - кастомная логика
2. **[@EntityGraph](Data%20JPA%20Repositories/7.%20@EntityGraph/)** - оптимизация загрузки данных
3. **[JPA Auditing](Data%20JPA%20Repositories/11.%20JPA%20Auditing/)** - аудит изменений
4. **[Data JPA Transactions](Data%20JPA%20Transactions/)** - управление транзакциями

## 🔍 Быстрая навигация

### По типу операций:

**Чтение данных:**
- [Repository](1.%20Основы%20Spring%20Data%20JPA/Repository.md) - базовые методы поиска
- [@Query](1.%20Основы%20Spring%20Data%20JPA/@Query.md) - кастомные запросы на чтение
- [Page & Slice](Data%20JPA%20Repositories/6.%20Page%20&%20Slice/) - пагинация
- [Projection](Data%20JPA%20Repositories/9.%20Projection/) - проекции

**Изменение данных:**
- [@Modifying](1.%20Основы%20Spring%20Data%20JPA/@Modifying.md) - операции UPDATE/DELETE
- [Repository](1.%20Основы%20Spring%20Data%20JPA/Repository.md) - методы save/delete

**Оптимизация:**
- [@EntityGraph](Data%20JPA%20Repositories/7.%20@EntityGraph/) - управление загрузкой
- [@Lock & @QueryHints](Data%20JPA%20Repositories/8.%20@Lock%20&%20@QueryHints/) - блокировки и подсказки

**Продвинутые техники:**
- [Custom Repository Implementation](Data%20JPA%20Repositories/10.%20Custom%20Repository%20Implementation/)
- [Querydsl](Data%20JPA%20Repositories/13.%20Querydsl/)

### По уровню сложности:

**🟢 Начальный уровень:**
- Repository.md
- @Query.md (базовые примеры)
- PartTreeJpaQuery (автоматические запросы)

**🟡 Средний уровень:**
- @Modifying.md
- Page & Slice
- @EntityGraph
- JPA Auditing

**🔴 Продвинутый уровень:**
- Custom Repository Implementation
- Data JPA Transactions
- Querydsl
- @Lock & @QueryHints

## 📋 Практические примеры

Каждый документ содержит:
- ✅ Теоретическое объяснение
- ✅ Практические примеры кода
- ✅ Лучшие практики
- ✅ Типичные ошибки и их решения
- ✅ Настройки производительности

## 🔧 Настройка проекта

Для работы с примерами из документации, добавьте в ваш проект:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Базовая конфигурация в `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/testdb
    username: user
    password: password
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
```

## ❓ Часто задаваемые вопросы

**Q: В чем разница между CrudRepository и JpaRepository?**
A: Смотрите подробное объяснение в [Repository.md](1.%20Основы%20Spring%20Data%20JPA/Repository.md#иерархия-интерфейсов-repository)

**Q: Когда использовать @Query, а когда derived queries?**
A: Руководство в [@Query.md](1.%20Основы%20Spring%20Data%20JPA/@Query.md) и [PartTreeJpaQuery](Data%20JPA%20Repositories/3.%20PartTreeJpaQuery/)

**Q: Как оптимизировать производительность запросов?**
A: Изучите [@EntityGraph](Data%20JPA%20Repositories/7.%20@EntityGraph/), [@Lock & @QueryHints](Data%20JPA%20Repositories/8.%20@Lock%20&%20@QueryHints/) и [пагинацию](Data%20JPA%20Repositories/6.%20Page%20&%20Slice/)

## 📞 Дополнительные ресурсы

- [Официальная документация Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Hibernate Documentation](https://hibernate.org/orm/documentation/)
- [JPA Specification](https://jakarta.ee/specifications/persistence/)

---

## 📈 История изменений

**Последнее обновление**: Реорганизация структуры документации
- ✅ Объединены дублирующиеся файлы
- ✅ Создана логическая структура обучения  
- ✅ Добавлена детальная навигация
- ✅ Расширены практические примеры

**Принципы организации документации:**
- Максимальная детализация и практические примеры [[memory:4107629]]
- Сохранение всей полезной информации без потерь [[memory:4107636]]
- Логическое группирование по сложности и функциональности
- Четкая навигация для эффективного изучения

