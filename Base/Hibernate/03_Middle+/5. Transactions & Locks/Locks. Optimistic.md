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

## 10. Senior-insights: детали и подводные камни <a name="senior"></a>

### OptimisticLockException при merge() detached entity

Самый частый сценарий в REST API: загрузить entity → вернуть клиенту → получить изменённый объект → применить.

```java
// Шаг 1: загрузка (TX1)
Product product = productService.findById(1L);
// product.version = 5

// Шаг 2: клиент редактирует (время прошло, другой пользователь изменил product)
// В БД: product.version = 6

// Шаг 3: сохранение (TX2)
@Transactional
public Product updateProduct(Product detachedProduct) {
    // detachedProduct.version = 5, в БД version = 6
    return entityManager.merge(detachedProduct);
    // Hibernate: UPDATE ... WHERE id=1 AND version=5
    // БД: затронуто 0 строк (version != 5)
    // → OptimisticLockException!
}
```

**Правильная обработка**:

```java
@Service
public class ProductService {

    @Transactional
    public Product updateProduct(ProductUpdateRequest request) {
        try {
            Product product = entityManager.find(Product.class, request.getId());
            product.setName(request.getName());
            product.setPrice(request.getPrice());
            return product; // dirty checking + version check при commit
        } catch (OptimisticLockException | ObjectOptimisticLockingFailureException e) {
            throw new ConcurrentModificationException("Product was concurrently modified", e);
        }
    }
}
```

### Паттерн retry-loop при оптимистичных блокировках

```java
@Service
public class OrderService {

    private static final int MAX_RETRIES = 3;

    public Order updateOrderWithRetry(Long orderId, OrderUpdateRequest request) {
        int attempts = 0;
        while (true) {
            try {
                return doUpdate(orderId, request);
            } catch (OptimisticLockException | ObjectOptimisticLockingFailureException e) {
                attempts++;
                if (attempts >= MAX_RETRIES) {
                    throw new MaxRetriesExceededException(
                        "Failed to update order after " + MAX_RETRIES + " attempts", e);
                }
                // Задержка с jitter перед повтором
                try {
                    Thread.sleep(50L * attempts + (long)(Math.random() * 50));
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(ie);
                }
            }
        }
    }

    @Transactional
    private Order doUpdate(Long orderId, OrderUpdateRequest request) {
        Order order = entityManager.find(Order.class, orderId);
        order.setStatus(request.getStatus());
        return order;
    }
}
```

> [!WARNING]
> Retry-loop имеет смысл только при **низкой частоте конфликтов**. Постоянные конфликты — сигнал к переходу на пессимистичные блокировки.

### @Version с Long vs Timestamp: почему Timestamp ненадёжен

```java
// ПЛОХО: Timestamp в @Version
@Entity
public class Product {
    @Version
    private Timestamp lastModified;
    // Проблема 1: MySQL DATETIME точность до секунды (до версии 5.6)
    // Два UPDATE в одну секунду → одинаковый timestamp → конфликт НЕ обнаруживается!
    // Проблема 2: clock drift в кластере
    // Проблема 3: сложнее читать в логах и API-ответах
}

// ХОРОШО: Long-версия
@Entity
public class Product {
    @Version
    private Long version; // монотонно возрастает, нет проблем с точностью
    // Hibernate: UPDATE ... SET version = version + 1 WHERE version = ?
}
```

### @OptimisticLocking без @Version (для legacy-таблиц)

**ALL — сравнивать все поля при UPDATE**:
```java
@Entity
@OptimisticLocking(type = OptimisticLockType.ALL)
@DynamicUpdate // обязателен совместно с ALL/DIRTY
public class LegacyProduct {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    // Нет @Version!
    // UPDATE ... WHERE id=? AND name='old' AND price=99.0
    // Конфликт если хоть одно поле изменилось другим пользователем
}
```

**DIRTY — сравнивать только изменённые поля**:
```java
@Entity
@OptimisticLocking(type = OptimisticLockType.DIRTY)
@DynamicUpdate // обязателен!
public class LegacyProduct {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    // Изменили только price:
    // UPDATE ... SET price=? WHERE id=? AND price=99.0
    // Если два пользователя меняют РАЗНЫЕ поля — конфликта НЕ будет
}
```

> [!INFO]
> `@DynamicUpdate` обязателен при `ALL`/`DIRTY`: без него Hibernate включает все поля в UPDATE-запрос, что нарушает логику версионного сравнения.

### Когда оптимистичные лучше пессимистичных

| Сценарий | Рекомендация |
|----------|-------------|
| Читают часто, меняют редко (чтение > 90%) | Оптимистичные |
| Высокая конкуренция за одни и те же записи | Пессимистичные |
| Короткие транзакции (< 1 сек) | Оптимистичные |
| Long-running workflows (минуты) | Оптимистичные (нет смысла держать DB lock) |
| Финансовые операции, inventory | Пессимистичные |
| REST API без состояния | Оптимистичные (версия передаётся в запросе) |

---

**Дополнительные ресурсы:**
- [JPA Locking](https://docs.oracle.com/javaee/7/tutorial/persistence-locking001.htm)
- [Hibernate: Optimistic Locking](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#locking-optimistic)
- [Spring Data JPA: Optimistic Locking](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.locking.optimistic)

**Связанные файлы:**
- [[@Transactional Propagation]] — транзакции и их поведение
- [[Locks. Pessimistic]] — пессимистичные блокировки