# Batch операции в Hibernate

## Оглавление
1. [Что такое batching и зачем он нужен](#что-такое)
2. [hibernate.jdbc.batch_size](#batch-size)
3. [order_inserts и order_updates](#ordering)
4. [Почему IDENTITY-генератор отключает batching](#identity-problem)
5. [SEQUENCE + hi/lo оптимизация](#sequence)
6. [Паттерн bulk insert: flush + clear](#flush-clear)
7. [StatelessSession как альтернатива](#stateless)

---

## 1. Что такое batching и зачем он нужен <a name="что-такое"></a>

По умолчанию Hibernate отправляет каждый SQL-запрос отдельным round-trip к БД. При вставке 1000 строк — 1000 round-trips. Это медленно.

**JDBC batching** позволяет накапливать несколько SQL в буфере и отправлять их одним пакетом:

```
Без batching: [INSERT] [INSERT] [INSERT] ... × 1000 round-trips
С batching:   [INSERT, INSERT, INSERT...50 штук] × 20 round-trips
```

> [!INFO]
> На локальной БД разница незначительна. На удалённой БД (network latency ~1ms) вставка 10000 строк ускоряется с ~10 секунд до ~0.2 секунды.

---

## 2. hibernate.jdbc.batch_size <a name="batch-size"></a>

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50  # размер одного batch-пакета
```

```java
// Что происходит с batch_size=50:
for (int i = 0; i < 200; i++) {
    entityManager.persist(new User("user" + i));
}
entityManager.flush();
// JDBC отправляет:
// batch 1: INSERT × 50
// batch 2: INSERT × 50
// batch 3: INSERT × 50
// batch 4: INSERT × 50
// Итого: 4 round-trips вместо 200
```

**Почему по умолчанию выключен (`batch_size=0`)**:
- Hibernate не может знать, сколько памяти безопасно использовать под буфер
- IDENTITY-генераторы не совместимы с batching (см. ниже)
- Требует явного `flush()` + `clear()` для правильной работы

**Рекомендуемые значения**: 20-50 для INSERT-heavy, 10-20 для UPDATE-heavy.

---

## 3. order_inserts и order_updates <a name="ordering"></a>

```yaml
spring:
  jpa:
    properties:
      hibernate:
        order_inserts: true   # группировать INSERT по типу entity
        order_updates: true   # группировать UPDATE по типу entity
```

**Без ordering**: ActionQueue может содержать перемежающиеся операции:
```
INSERT User, INSERT Order, INSERT User, INSERT Order, ...
```
JDBC batch прерывается при смене типа запроса → эффективность batching падает до нуля.

**С ordering**: Hibernate перегруппирует операции:
```
INSERT User × 50, INSERT User × 50, INSERT Order × 50, ...
```
Каждый batch однородный → JDBC может эффективно батчить.

> [!WARNING]
> `order_inserts=true` без `batch_size` — бесполезно. Эти две настройки работают только в паре.

---

## 4. Почему IDENTITY-генератор отключает batching <a name="identity-problem"></a>

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

IDENTITY (`AUTO_INCREMENT` в MySQL) генерирует ID на стороне БД **во время INSERT**. Hibernate узнаёт ID только после выполнения INSERT.

**Проблема**: Hibernate требует ID сразу после `persist()`, чтобы управлять Persistence Context. Поэтому он вынужден:
1. Выполнить INSERT немедленно (не буферизовать)
2. Сразу же сделать SELECT LAST_INSERT_ID() (или аналог)

```
persist(user1) → немедленный INSERT + SELECT id  (round-trip 1)
persist(user2) → немедленный INSERT + SELECT id  (round-trip 2)
persist(user3) → немедленный INSERT + SELECT id  (round-trip 3)
// Batching полностью отключён!
```

> [!WARNING]
> `GenerationType.IDENTITY` + `hibernate.jdbc.batch_size` = batching не работает. Hibernate молча игнорирует batch_size для IDENTITY-стратегии.

---

## 5. SEQUENCE + hi/lo оптимизация <a name="sequence"></a>

`GenerationType.SEQUENCE` — правильный выбор для batching:

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(
    name = "user_seq",
    sequenceName = "user_sequence",
    allocationSize = 50  // hi/lo оптимизация: одна SELECT = 50 ID
)
private Long id;
```

**Как работает hi/lo оптимизация**:
- Hibernate делает `CALL NEXT VALUE FOR user_sequence` → получает значение `N`
- Резервирует диапазон `[N * allocationSize ... (N+1) * allocationSize - 1]`
- Следующие 50 `persist()` получают ID из этого диапазона **без обращения к БД**
- Когда диапазон исчерпан — один новый `CALL NEXT VALUE`

```
Вставка 1000 записей:
- 20 вызовов SEQUENCE (1000 / 50 = 20)
- + 20 batch INSERT-пакетов
= 40 round-trips вместо 1000
```

> [!INFO]
> `allocationSize` должен совпадать с `hibernate.jdbc.batch_size` или быть его кратным для максимальной эффективности. Если `allocationSize=50` и `batch_size=50` — идеальная синхронизация.

---

## 6. Паттерн bulk insert: flush + clear <a name="flush-clear"></a>

Без `clear()` Persistence Context растёт неограниченно → OutOfMemoryError и деградация dirty checking.

```java
@Transactional
public void bulkInsert(List<UserDto> dtos) {
    int batchSize = 50; // должен совпадать с hibernate.jdbc.batch_size
    
    for (int i = 0; i < dtos.size(); i++) {
        User user = new User();
        user.setEmail(dtos.get(i).getEmail());
        user.setFirstName(dtos.get(i).getFirstName());
        
        entityManager.persist(user);
        
        // Каждые N записей: сбрасываем в БД и очищаем PC
        if (i % batchSize == 0 && i > 0) {
            entityManager.flush(); // отправляем SQL в БД
            entityManager.clear(); // очищаем Persistence Context
            // Теперь все объекты стали detached — GC может их собрать
        }
    }
    // Последний неполный batch — flush произойдёт при commit
}
```

**Что происходит без `clear()`**:
- После 10000 `persist()` в PC находится 10000 объектов + 10000 snapshots
- При каждом flush Hibernate обходит **все 10000** для dirty checking
- Операция экспоненциально замедляется

**Что происходит с `clear()`**:
- PC содержит максимум 50 объектов одновременно
- Dirty checking — быстрый
- Память — стабильная

> [!WARNING]
> После `entityManager.clear()` все ранее загруженные объекты становятся **detached**. Не держи на них ссылок — доступ к lazy-ассоциациям вызовет LazyInitializationException.

### Bulk UPDATE через HQL (альтернатива)

```java
// Иногда проще обойтись без цикла:
int updated = entityManager.createQuery(
    "UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoff")
    .setParameter("cutoff", LocalDate.now().minusYears(1))
    .executeUpdate();
// Один SQL UPDATE — максимальная производительность
```

> [!WARNING]
> Bulk UPDATE/DELETE через HQL **не обновляет** L2 Cache и **не триггерит** lifecycle callbacks (`@PreUpdate` и т.д.). Используй только когда callbacks не нужны.

---

## 7. StatelessSession как альтернатива <a name="stateless"></a>

```java
try (StatelessSession session = sessionFactory.openStatelessSession()) {
    Transaction tx = session.beginTransaction();
    
    for (UserDto dto : dtos) {
        User user = new User();
        user.setEmail(dto.getEmail());
        session.insert(user); // немедленно в JDBC batch buffer
        // нет PC, нет flush/clear — объект не удерживается в памяти
    }
    
    tx.commit();
}
```

| Критерий | flush+clear паттерн | StatelessSession |
|----------|--------------------|--------------------|
| Lifecycle callbacks | Работают | Не работают |
| Каскадирование | Работает | Не работает |
| Сложность | Средняя | Низкая |
| Производительность | Высокая | Выше |
| Контроль памяти | Ручной (flush+clear) | Автоматический |

Подробнее: [[StatelessSession]]

---

**Связанные файлы:**
- [[StatelessSession]] — bulk операции без Persistence Context
- [[Dirty Checking и FlushMode]] — почему важен clear()
- [[Primary keys. Embedded components]] — SEQUENCE vs IDENTITY
