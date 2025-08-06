# Оптимистические блокировки в JPA/Hibernate

## Оглавление
1. [Что такое оптимистическая блокировка](#что-это)
2. [Оптимистические vs пессимистические блокировки](#vs)
3. [LockModeType и аннотация @Version](#lockmode)
4. [Аннотация @OptimisticLocking](#optimisticlocking)
5. [OptimisticLockException и обработка конфликтов](#exception)
6. [Типы OptimisticLockType: ALL, DIRTY](#types)
7. [@DynamicUpdate и @DynamicInsert](#dynamic)
8. [Best practices](#best-practices)
9. [Вопросы для собеседования](#вопросы)

---

## 1. Что такое оптимистическая блокировка <a name="что-это"></a>

**Оптимистическая блокировка** — это подход к контролю конкурентного доступа, при котором предполагается, что конфликты будут редкими. Проверка конфликтов происходит только при попытке записи (commit/merge).

- Не блокирует строки в БД
- Использует версионное поле (обычно @Version)
- При конфликте выбрасывается OptimisticLockException

## 2. Оптимистические vs пессимистические блокировки <a name="vs"></a>

| Характеристика         | Оптимистическая | Пессимистическая |
|------------------------|-----------------|------------------|
| Блокировка в БД        | Нет             | Да               |
| Производительность     | Выше            | Ниже             |
| Конфликты              | Обнаруживаются при commit | Предотвращаются заранее |
| Использование          | Чтение чаще, чем запись | Высокая конкуренция |

## 3. LockModeType и аннотация @Version <a name="lockmode"></a>

- **LockModeType.OPTIMISTIC** — стандартная оптимистическая блокировка
- **LockModeType.OPTIMISTIC_FORCE_INCREMENT** — увеличивает версию даже при чтении

### Пример:
```java
@Entity
public class User {
    @Id
    private Long id;
    @Version
    private Integer version;
    // ...
}
```

### Использование:
```java
entityManager.find(User.class, id, LockModeType.OPTIMISTIC);
```

## 4. Аннотация @OptimisticLocking <a name="optimisticlocking"></a>

- Позволяет управлять стратегией оптимистической блокировки на уровне класса
- В Hibernate: @OptimisticLocking(type = OptimisticLockType.ALL)

### Пример:
```java
@Entity
@OptimisticLocking(type = OptimisticLockType.ALL)
public class Order { ... }
```

## 5. OptimisticLockException и обработка конфликтов <a name="exception"></a>

- Если при commit обнаружено, что версия изменилась — выбрасывается OptimisticLockException
- Необходимо обработать исключение (например, повторить операцию или показать ошибку пользователю)

### Пример:
```java
try {
    // ... commit ...
} catch (OptimisticLockException e) {
    // обработка конфликта
}
```

## 6. Типы OptimisticLockType: ALL, DIRTY <a name="types"></a>

- **ALL** — версия увеличивается при любом изменении
- **DIRTY** — версия увеличивается только при изменении dirty-полей

## 7. @DynamicUpdate и @DynamicInsert <a name="dynamic"></a>

- **@DynamicUpdate** — Hibernate генерирует UPDATE только по изменённым полям
- **@DynamicInsert** — INSERT только по непустым полям
- Используются для оптимизации и уменьшения конфликтов

### Пример:
```java
@Entity
@DynamicUpdate
public class Product { ... }
```

## 8. Best practices <a name="best-practices"></a>
- Используйте оптимистическую блокировку для систем с преобладанием чтения
- Всегда добавляйте поле @Version для важных сущностей
- Обрабатывайте OptimisticLockException корректно (повтор, откат, уведомление)
- Используйте @DynamicUpdate для уменьшения конфликтов
- Не используйте оптимистическую блокировку для критичных секций с высокой конкуренцией

## 9. Вопросы для собеседования <a name="вопросы"></a>
1. Как работает оптимистическая блокировка в JPA/Hibernate?
2. Для чего используется аннотация @Version?
3. Чем отличается OptimisticLockType.ALL от DIRTY?
4. Как обработать OptimisticLockException?
5. Когда лучше использовать оптимистическую, а когда — пессимистическую блокировку?

---

**Дополнительные ресурсы:**
- [JPA Locking](https://docs.oracle.com/javaee/7/tutorial/persistence-locking001.htm)
- [Hibernate: Optimistic Locking](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#locking-optimistic)
- [Spring Data JPA: Optimistic Locking](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.locking.optimistic)