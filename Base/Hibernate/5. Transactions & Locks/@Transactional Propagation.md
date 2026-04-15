# @Transactional Propagation

## Оглавление
1. [Что такое Propagation](#что-такое)
2. [Все 7 уровней propagation](#уровни)
3. [REQUIRED vs REQUIRES_NEW — реальный сценарий](#required-vs-new)
4. [Почему NESTED не работает с JPA](#nested)
5. [Self-invocation проблема](#self-invocation)
6. [Transaction-scoped vs Extended EntityManager](#entitymanager-scope)

---

## 1. Что такое Propagation <a name="что-такое"></a>

**Propagation** определяет, что произойдёт с транзакцией при вызове метода, помеченного `@Transactional`, если транзакция уже существует (или отсутствует).

```java
@Transactional(propagation = Propagation.REQUIRED) // по умолчанию
public void myMethod() { ... }
```

Spring реализует propagation через AOP-прокси. При вызове `@Transactional`-метода прокси:
1. Проверяет наличие транзакции в `TransactionSynchronizationManager` (ThreadLocal)
2. Принимает решение на основе настроенного `propagation`
3. Начинает/использует/приостанавливает транзакцию

---

## 2. Все 7 уровней propagation <a name="уровни"></a>

### REQUIRED (по умолчанию)

```java
@Transactional(propagation = Propagation.REQUIRED)
public void method() { ... }
```

- Если транзакция **есть** → присоединяется к ней
- Если транзакции **нет** → создаёт новую

**На уровне БД**: одна физическая транзакция. Rollback внутреннего метода = rollback всей транзакции.

```java
// ВАЖНО: inner и outer — ОДНА транзакция
outerService.outerMethod(); // начинает TX
  innerService.innerMethod(); // ПРИСОЕДИНЯЕТСЯ к той же TX
  // если innerMethod бросает исключение → вся TX откатится
```

---

### REQUIRES_NEW

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void method() { ... }
```

- **Всегда** создаёт новую транзакцию
- Текущая транзакция (если есть) **приостанавливается** до завершения нового метода

**На уровне БД**: две отдельные физические транзакции. Rollback внутренней не затрагивает внешнюю.

---

### SUPPORTS

```java
@Transactional(propagation = Propagation.SUPPORTS)
public void method() { ... }
```

- Если транзакция **есть** → участвует в ней
- Если транзакции **нет** → выполняется без транзакции

**Когда использовать**: read-only методы, которые корректны как с транзакцией, так и без.

---

### NOT_SUPPORTED

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void method() { ... }
```

- Если транзакция **есть** → приостанавливает её, выполняется без транзакции
- Если транзакции **нет** → выполняется без транзакции

**Когда использовать**: операции, которые нельзя выполнять в транзакции (например, вызов внешнего API с автокоммитом).

---

### NEVER

```java
@Transactional(propagation = Propagation.NEVER)
public void method() { ... }
```

- Если транзакция **есть** → выбрасывает `IllegalTransactionStateException`
- Если транзакции **нет** → выполняется без транзакции

**Когда использовать**: принудительное требование отсутствия транзакции. Используется редко.

---

### MANDATORY

```java
@Transactional(propagation = Propagation.MANDATORY)
public void method() { ... }
```

- Если транзакция **есть** → присоединяется к ней
- Если транзакции **нет** → выбрасывает `IllegalTransactionStateException`

**Когда использовать**: метод-хелпер, который должен вызываться только внутри уже существующей транзакции.

---

### NESTED

```java
@Transactional(propagation = Propagation.NESTED)
public void method() { ... }
```

- Если транзакция **есть** → создаёт вложенную транзакцию через JDBC Savepoint
- Если транзакции **нет** → создаёт новую (как REQUIRED)

**На уровне БД**: Savepoint внутри существующей транзакции. Rollback внутреннего метода откатывает только до Savepoint, внешняя транзакция продолжается.

> [!WARNING]
> **NESTED не работает с JPA/Hibernate**! JPA не поддерживает Savepoint в своей абстракции. `NESTED` работает только с чистым JDBC через `DataSourceTransactionManager`. При использовании с `JpaTransactionManager` поведение либо эквивалентно `REQUIRED`, либо выбрасывается исключение.

---

### Сводная таблица

| Propagation | Есть TX | Нет TX | Отдельная TX в БД |
|-------------|---------|--------|-------------------|
| REQUIRED | Присоединяется | Создаёт | Нет |
| REQUIRES_NEW | Создаёт новую (suspend текущей) | Создаёт | Да |
| SUPPORTS | Присоединяется | Нет TX | Нет |
| NOT_SUPPORTED | Suspend, нет TX | Нет TX | Нет |
| NEVER | Exception | Нет TX | Нет |
| MANDATORY | Присоединяется | Exception | Нет |
| NESTED | Savepoint (JDBC only) | Создаёт | Частично |

---

## 3. REQUIRED vs REQUIRES_NEW — реальный сценарий <a name="required-vs-new"></a>

**Задача**: логировать все действия пользователя в `AuditLog`. Запись в лог должна сохраниться **даже если основная операция откатилась**.

```java
@Service
public class UserService {
    
    @Autowired
    private AuditService auditService;
    
    @Transactional // REQUIRED — основная транзакция
    public void updateUser(Long id, String newEmail) {
        User user = entityManager.find(User.class, id);
        user.setEmail(newEmail);
        
        auditService.logAction("UPDATE_USER", id); // должен сохраниться даже при rollback!
        
        if (newEmail.contains("banned")) {
            throw new BusinessException("Email not allowed"); // основная TX откатится
            // Без REQUIRES_NEW: auditLog тоже откатится → нет записи в логе
        }
    }
}

@Service
public class AuditService {
    
    // REQUIRES_NEW: своя независимая транзакция
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAction(String action, Long entityId) {
        AuditLog log = new AuditLog(action, entityId, LocalDateTime.now());
        entityManager.persist(log);
        // эта транзакция закоммитится НЕЗАВИСИМО от основной
    }
}
```

**Что происходит**:
1. `updateUser` начинает TX1
2. `logAction` **приостанавливает TX1**, начинает TX2
3. TX2 коммитится — `AuditLog` сохранён в БД
4. TX1 возобновляется, бросает исключение → TX1 откатывается
5. В БД: `User` не изменён, `AuditLog` сохранён ✓

> [!INFO]
> `REQUIRES_NEW` создаёт **отдельное соединение с БД** (берёт новое из пула). Учитывай это при настройке размера connection pool.

---

## 4. Почему NESTED не работает с JPA <a name="nested"></a>

JPA `EntityManager` управляет Persistence Context — он не знает о Savepoint-ах на уровне JDBC. Hibernate не может "откатить часть" PC к состоянию на момент Savepoint.

```java
// Это НЕ работает как ожидается:
@Transactional(propagation = Propagation.NESTED)
public void innerMethod() {
    entityManager.persist(someEntity);
    throw new Exception(); // Savepoint rollback?
    // Hibernate с JpaTransactionManager:
    // либо откатит всю транзакцию (как REQUIRED)
    // либо выбросит исключение при попытке создать Savepoint
}
```

**Что использовать вместо NESTED**:
- `REQUIRES_NEW` — если нужна независимость (но это отдельная TX)
- Ручное управление через JDBC `connection.setSavepoint()` (покидаем JPA-абстракцию)

---

## 5. Self-invocation проблема <a name="self-invocation"></a>

`@Transactional` работает через AOP-прокси. Вызов метода **внутри того же бина** обходит прокси → `@Transactional` не применяется.

```java
@Service
public class OrderService {
    
    @Transactional
    public void processOrder(Long orderId) {
        // Бизнес-логика
        saveAuditLog(orderId); // вызов через 'this' — обходит прокси!
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW) // НЕ РАБОТАЕТ!
    public void saveAuditLog(Long orderId) {
        // Эта аннотация игнорируется при self-invocation
        // метод выполнится в той же транзакции, что и processOrder
    }
}
```

**Решения**:

```java
// 1. Вынести метод в отдельный бин (предпочтительно)
@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditLog(Long orderId) { ... }
}

// 2. Инжектировать себя (анти-паттерн, но работает)
@Service
public class OrderService {
    @Autowired
    private OrderService self; // Spring инжектирует прокси
    
    public void processOrder(Long orderId) {
        self.saveAuditLog(orderId); // теперь через прокси
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditLog(Long orderId) { ... }
}

// 3. Получить прокси через ApplicationContext (анти-паттерн)
@Service
public class OrderService implements ApplicationContextAware {
    private ApplicationContext context;
    
    public void processOrder(Long orderId) {
        OrderService proxy = context.getBean(OrderService.class);
        proxy.saveAuditLog(orderId);
    }
}
```

> [!WARNING]
> Self-invocation — самая частая причина "почему @Transactional не работает". Первый вопрос при отладке: вызывается ли метод через прокси или напрямую?

---

## 6. Transaction-scoped vs Extended EntityManager <a name="entitymanager-scope"></a>

### Transaction-scoped (по умолчанию в Spring)

```java
@PersistenceContext
private EntityManager em; // прокси, создаёт реальный EM при начале TX
```

- `EntityManager` живёт в рамках одной транзакции
- При commit: flush + close
- Persistence Context очищается в конце каждого `@Transactional` метода

### Extended EntityManager

```java
@PersistenceContext(type = PersistenceContextType.EXTENDED)
private EntityManager em; // только в @Stateful EJB или специальный бин
```

- `EntityManager` переживает несколько транзакций
- Сущности остаются managed между транзакциями
- В Spring: доступно через `@Scope("conversation")` или специальную конфигурацию

> [!INFO]
> В большинстве Spring-приложений используется Transaction-scoped EM. Extended EM нужен для "conversational persistence" — сценариев, где один бизнес-процесс охватывает несколько запросов/транзакций.

---

**Связанные файлы:**
- [[JPA Transactions]] — управление транзакциями в JPA
- [[Locks. Optimistic]] — оптимистичные блокировки и транзакции
- [[Locks. Pessimistic]] — пессимистичные блокировки
