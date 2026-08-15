# Atomic (java.util.concurrent.atomic)

> Lock-free операции над переменными через CAS на уровне процессора — без `synchronized`, без блокировок.  
> На интервью важно понимать: как работает CAS, чем `LongAdder` лучше `AtomicLong`, что такое ABA, и зачем нужен `VarHandle`.

## Связанные темы
[[CAS и Unsafe]], [[Модель памяти Java (JMM) и барьеры памяти]], [[Lock]], [[Java Monitor]]

---

## Основные классы

| Класс | Тип | Назначение |
|---|---|---|
| `AtomicInteger` / `AtomicLong` | `int` / `long` | Счётчики, флаги, CAS-обновления |
| `AtomicBoolean` | `boolean` | Атомарное переключение состояния |
| `AtomicReference<V>` | любой объект | Атомарная замена ссылки |
| `AtomicStampedReference<V>` | объект + `int` | Защита от ABA через версию (stamp) |
| `AtomicMarkableReference<V>` | объект + `boolean` | Атомарная пометка (например, "удалён") |
| `AtomicIntegerArray` / `AtomicLongArray` | массив | Атомарные операции над элементами массива |
| `LongAdder` / `DoubleAdder` | `long` / `double` | Счётчики при высокой конкуренции |
| `LongAccumulator` / `DoubleAccumulator` | `long` / `double` | Произвольная ассоциативная операция (max, min, ...) |

---

## Ключевые операции

```java
AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();           // ++counter (возвращает новое)
counter.getAndIncrement();           // counter++ (возвращает старое)
counter.addAndGet(5);                // counter += 5
counter.compareAndSet(10, 20);       // if (counter == 10) counter = 20

// Java 8+: функциональное обновление (реализовано через CAS loop)
counter.updateAndGet(x -> x * 2);   // возвращает новое значение
counter.accumulateAndGet(5, Integer::max); // counter = max(counter, 5)
```

| Метод | Что делает |
|---|---|
| `get()` | volatile-чтение |
| `set(v)` | volatile-запись |
| `lazySet(v)` | setOpaque — запись без StoreLoad барьера (оптимизация для publish в producer) |
| `compareAndSet(e, v)` | сильный CAS: atomically if (==e) set(v) |
| `weakCompareAndSetPlain(e, v)` | слабый CAS — может ложно вернуть false (для спин-циклов) |

---

## Как работает CAS

CAS (Compare-And-Swap) — атомарная инструкция процессора (`CMPXCHG` на x86, `LDXR`+`STXR` на ARM), на которой построены все методы этого пакета: спин-цикл читает текущее значение и пытается заменить его, повторяя попытку при конфликте. Полный разбор механизма, различий x86/ARM (spurious failure) и `compareAndExchange` — в [[CAS и Unsafe]].

```java
// Логика incrementAndGet (упрощённо) — спин-цикл вокруг CAS:
int current, next;
do {
    current = get();          // volatile read
    next = current + 1;
} while (!compareAndSet(current, next));  // CAS — если значение изменилось — повторить
return next;
```

Связь с JMM: успешный `compareAndSet` имеет семантику volatile — устанавливает happens-before между потоками (неуспешный — нет).

---

## LongAdder vs AtomicLong: Striped64 под капотом

`LongAdder` быстрее `AtomicLong` при высокой конкуренции за счёт разбивки на ячейки:

```
Striped64:
  base (volatile long)   — используется, пока нет конкуренции
  Cell[] cells           — каждый @Contended (64 байт padding, своя cache line)

increment():
  → попытка CAS на base
  → если конкуренция → пишем в cells[threadProbe % cells.length]
  → при частых промахах — массив cells удваивается (до числа CPU)

sum() = base + cells[0] + cells[1] + ... (НЕ атомарна!)
```

```java
LongAdder adder = new LongAdder();
adder.increment();          // ~15 нс при 32 потоках
// vs
AtomicLong atomic = new AtomicLong();
atomic.incrementAndGet();   // ~200 нс при 32 потоках (все крутятся на одной ячейке)
```

> [!WARNING] `LongAdder.sum()` не атомарна
> Суммирует base + cells без блокировки. Подходит для метрик (приближённое значение).  
> Для точного значения после завершения всех операций — корректна. Для rolling-window с конкурентным чтением — нужна внешняя синхронизация или `AtomicLong`.

`LongAccumulator` — обобщение `LongAdder` для произвольной ассоциативной операции:
```java
LongAccumulator maxAcc = new LongAccumulator(Long::max, 0);
maxAcc.accumulate(42); // потокобезопасный running max
```

---

## AtomicReference для lock-free state machines

`AtomicReference` + иммутабельный record = переходы состояний без блокировок:

```java
record ServiceState(int connections, boolean healthy, Instant lastCheck) {}

class ServiceMonitor {
    private final AtomicReference<ServiceState> state = new AtomicReference<>(
        new ServiceState(0, true, Instant.now())
    );

    public void recordConnection() {
        // updateAndGet реализован через CAS loop — повторяет при конфликте:
        state.updateAndGet(s ->
            new ServiceState(s.connections() + 1, s.healthy(), s.lastCheck())
        );
    }

    public void markUnhealthy() {
        state.updateAndGet(s ->
            new ServiceState(s.connections(), false, Instant.now())
        );
    }
}
```

Почему лучше `synchronized`: нет блокировки → нет deadlock, нет lock inflation, ABA не страшна (record equality по значению, не по ссылке).

---

## ABA-проблема и AtomicStampedReference

Проблема: поток читает A, другой меняет A→B→A, первый делает CAS(A,C) — успешно, хотя структура уже изменилась.

Критично в lock-free стеках и очередях, где узел может быть переиспользован — конкретный сценарий с interleaving двух потоков разобран в [[CAS и Unsafe]].

```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);

// Чтение с версией:
int[] stamp = new int[1];
String value = ref.get(stamp);          // stamp[0] = текущая версия

// CAS с проверкой версии:
ref.compareAndSet("A", "C", stamp[0], stamp[0] + 1);
// Успех только если значение == "A" И stamp == stamp[0]
```

---

## VarHandle (Java 9+)

`VarHandle` — стабильная замена `Unsafe` (миграция и статус `Unsafe` в Java 17+ — [[CAS и Unsafe]]). Помимо `get`/`set`/`compareAndSet` даёт тонкий контроль над memory ordering — режимы Plain/Opaque/Acquire-Release/Volatile и их барьеры разобраны в [[Модель памяти Java (JMM) и барьеры памяти]]. Используется для кастомных lock-free структур, когда Atomic-классов недостаточно (например, CAS над полем произвольного объекта без обёртки в `AtomicReference`):

```java
class Node {
    volatile int state;

    private static final VarHandle STATE;
    static {
        try {
            STATE = MethodHandles.lookup()
                .findVarHandle(Node.class, "state", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    void demo() {
        STATE.setVolatile(this, 1);                                // полный барьер, эквивалент volatile-поля
        STATE.compareAndSet(this, 0, 1);                            // CAS
        int witnessed = (int) STATE.compareAndExchange(this, 0, 1); // CAS, возвращает observed вместо boolean
    }
}
```

---

## Матрица выбора

| Сценарий | Инструмент |
|---|---|
| Счётчик, низкая конкуренция | `AtomicInteger` / `AtomicLong` |
| Счётчик, высокая конкуренция | `LongAdder` |
| Running max/min по нескольким потокам | `LongAccumulator` |
| Атомарный swap ссылки | `AtomicReference.compareAndSet()` |
| Иммутабельный state machine | `AtomicReference<Record>` + `updateAndGet()` |
| Lock-free стек/очередь с повторными узлами | `AtomicStampedReference` |
| Кастомный lock-free алгоритм | `VarHandle` с нужным ordering mode |
| Несколько полей атомарно | `synchronized` или `Lock` |

---

## Вопросы на интервью

- Чем `LongAdder` отличается от `AtomicLong`? Когда что использовать?
- Что такое ABA-проблема? Приведи пример, когда она ломает корректность.
- Что делает `compareAndSet` на уровне процессора?
- Почему `LongAdder.sum()` может вернуть не точное значение?
- Чем `VarHandle` лучше `Unsafe`? Что такое режимы Acquire/Release?
- Атомарны ли `updateAndGet` / `accumulateAndGet`? Как реализованы?
- Можно ли использовать `AtomicReference` для атомарного обновления двух полей?
- Что такое `lazySet` / `setOpaque`? Где применяется?

## Подводные камни

- **`LongAdder.sum()` не атомарна** — в момент вызова другие потоки могут менять ячейки. Не используй как синхронизационный примитив.
- **`lazySet` / `setOpaque` не устанавливают happens-before** — consumer может не увидеть значение сразу. Используй только в producer-only сценариях.
- **CAS loop при высокой конкуренции** — спиннинг нагружает CPU. Если конкуренция высокая — `LongAdder` или `Lock`.
- **ABA в lock-free структурах** — `AtomicReference` не защищает. При повторном использовании узлов обязательно `AtomicStampedReference`.
- **`VarHandle` создаётся дорого** — инициализируй в `static final`, не в каждом вызове.
- **`updateAndGet` не подходит для операций с side-effects** — функция может быть вызвана несколько раз при повторе CAS.
