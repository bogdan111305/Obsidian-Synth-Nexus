# Атомарность операций и Volatile

> [!QUOTE] Суть
> **Атомарность**: операция неделима (нет промежуточных состояний). `count++` — НЕ атомарна (read → increment → write). **volatile**: гарантирует видимость и запрещает reordering, но НЕ гарантирует атомарность составных операций. Для атомарных инкрементов — `AtomicInteger`.

> [!WARNING] Ловушка: volatile не заменяет synchronized
> `volatile` решает видимость (HB на write→read), но `count++` всё равно race condition. Два потока могут прочитать одно значение, оба инкрементируют, оба запишут — один инкремент потерян. Нужен `AtomicInteger` или `synchronized`.

**Атомарность** и **модификатор `volatile`** — ключевые концепции многопоточного программирования в Java, обеспечивающие корректное поведение в конкурентной среде. Атомарная операция выполняется целиком без прерываний, а `volatile` решает проблемы видимости и переупорядочивания операций.

## 1. Атомарность операций

**Атомарная операция** — это операция, которая выполняется как единое целое, без возможности прерывания другими потоками. В Java атомарность гарантируется для:

- **Чтения и записи примитивов**: `int`, `float`, `boolean`, `char`, `short`, `byte`, а также `long` и `double` (с Java 8, 64-битные операции атомарны).
- **Чтения и записи ссылок**: Например, присваивание объекта (`obj = new Object()`).
- **Операции с `volatile` переменными**: Чтение/запись с гарантией видимости.

**Неатомарные операции**:

- Составные операции, такие как `count++`, состоят из трёх шагов: чтение, изменение, запись. Они подвержены **race condition** (гонке данных).

**Пример неатомарной операции**:

```java
public class Counter {
    private int count;

    public void increment() {
        count++; // Неатомарно: read → increment → write
    }
}
```

**Проблема**:

- Поток A читает `count = 0`, поток B читает `count = 0`.
- Оба увеличивают `count` и записывают `count = 1`, вместо ожидаемого `count = 2`.

**Решение**: Использовать `synchronized`, `AtomicInteger` или другие механизмы синхронизации.

## 2. Модификатор `volatile`

`volatile` — модификатор переменной, решающий проблемы **видимости** и **переупорядочивания** в многопоточных приложениях. Пометка переменной как `volatile` заставляет JVM использовать специальные **барьеры памяти** (memory barriers) вокруг операций с этой переменной.

### 2.1. Что гарантирует `volatile`?

1. **Видимость (visibility)**:
    
    - Чтение/запись `volatile` переменной происходит напрямую в основную память, минуя локальные кэши CPU.
    - Изменения, сделанные одним потоком, сразу видны другим потокам.
2. **Запрет переупорядочивания (memory barriers)**:
    
    - JVM и CPU не могут переставлять операции с `volatile` переменной относительно других операций.
    - Барьеры памяти обеспечивают строгий порядок.

**Эти барьеры гарантируют**:

- **При записи в `volatile`**:
    - Все предыдущие операции записи из этого потока **будут завершены и записаны в основную память** перед записью в `volatile` переменную.
    - Обновление `volatile` переменной сразу **обновит кэши других ядер** (или пометит их как устаревшие), чтобы другие ядра не читали устаревшее значение.
- **При чтении `volatile`**:
    - Чтение `volatile` переменной заставляет поток **сбросить локальные кэши** или получить **свежие данные из основной памяти**.
    - Это гарантирует, что поток прочитает **актуальное значение** переменной.

**Переупорядочивание (reordering)**:

- Компилятор или CPU могут изменять порядок операций для оптимизации (например, `a = 1; b = 2;` → `b = 2; a = 1;`), если это не влияет на результат в одном потоке.
- В многопоточной среде это может привести к непредсказуемому поведению. Например:
    - Поток A пишет данные (`data = 42`) и устанавливает флаг (`flag = true`).
    - Поток B читает `flag = true`, но видит старое `data = 0` из-за переупорядочивания.
- `volatile` накладывает ограничения на переупорядочивание операций вокруг доступа к `volatile` переменной:
    - Операции до записи `volatile` не могут быть перенесены **после** неё.
    - Операции после чтения `volatile` не могут быть перенесены **до** неё.
- Проще говоря, `volatile` создаёт **"границу"** в потоке операций, которую компилятор и CPU не могут переставить.

**Пример**:

```java
public class VolatileExample {
    private volatile boolean flag = false;
    private int data;

    public void writer() {
        data = 42; // Операция 1
        flag = true; // Операция 2 (volatile)
    }

    public void reader() {
        if (flag) { // Чтение volatile
            System.out.println("Data: " + data); // Увидит data = 42
        }
    }
}
```

**Без `volatile`**:

- Поток B может увидеть `flag = true`, но `data = 0`, если операции переупорядочены.

**С `volatile`**:

- Гарантируется, что `data = 42` будет записано до `flag = true`, и поток B увидит актуальные значения.

### 2.2. Что не гарантирует `volatile`?

- **Не обеспечивает атомарность составных операций**:
    - Например, `volatile int count; count++;` не атомарен (чтение → увеличение → запись).
- **Не заменяет синхронизацию**:
    - Для сложных операций нужна `synchronized` или атомарные классы (`AtomicInteger`).

## 3. JVM-реализация `volatile`

> [!INFO] Теоретическая основа — в JMM
> Полный разбор барьеров памяти, happens-before правил, DCL и VarHandle ordering modes — в [[Модель памяти Java (JMM) и барьеры памяти]]. Здесь — только практический минимум.

`volatile` в байт-коде помечает поле флагом `ACC_VOLATILE`. JIT вставляет соответствующие барьеры:

```
// Запись: flag = true
putfield VolatileExample.flag:Z   // ACC_VOLATILE
// JIT на x86: LOCK XCHG (или implicit — store не может быть reordered past load)
// JIT на ARM:  STLR (store-release)

// Чтение: boolean f = flag
getfield VolatileExample.flag:Z   // ACC_VOLATILE
// JIT на x86: обычный load (x86 TSO делает reads уже acquire)
// JIT на ARM:  LDAR (load-acquire)
```

**Ключевой факт**: на x86 `volatile write` дороже `volatile read` (StoreLoad fence). На ARM обе операции одинаково дороги.

## 4. Практическое применение

### 4.1. Пример: Флаг остановки

```java
public class ShutdownExample {
    private volatile boolean running = true;

    public void worker() {
        while (running) {
            System.out.println("Working...");
            try { Thread.sleep(100); } catch (InterruptedException e) {}
        }
    }

    public void stop() {
        running = false; // Видимо всем потокам
    }

    public static void main(String[] args) throws InterruptedException {
        ShutdownExample example = new ShutdownExample();
        Thread worker = new Thread(example::worker);
        worker.start();
        Thread.sleep(500);
        example.stop();
    }
}
```

**Особенности**:

- `running` с `volatile` гарантирует, что изменение в `stop()` сразу видно в `worker()`.

### 4.2. Пример: Double-checked locking

> [!INFO] Подробный разбор с байт-кодом и тремя вариантами реализации — в [[Модель памяти Java (JMM) и барьеры памяти#JMM Double-Checked Locking — разбор по байт-коду]]

```java
// Корректный DCL: volatile обязателен!
public class Singleton {
    private static volatile Singleton instance; // volatile!

    public static Singleton getInstance() {
        if (instance == null) {              // 1. fast path без синхронизации
            synchronized (Singleton.class) {
                if (instance == null) {      // 2. slow path с синхронизацией
                    instance = new Singleton();
                    // volatile write = StoreStore+StoreLoad барьер
                    // конструктор ВСЕГДА завершён ДО этого барьера
                }
            }
        }
        return instance;
    }
}
// BEST: Initialization-on-demand holder (без volatile, без synchronized в fast path):
// private static class Holder { static final Singleton INSTANCE = new Singleton(); }
```

### 4.3. Применение в реальных сценариях

- **Java EE / Jakarta EE**: Управление флагами в сервлетах или CDI-бобах.
- **Spring**: Контроль состояния в `@Service` или `@Configuration` компонентах.
- **Паттерны**:
    - Флаги для активации/деактивации потоков.
    - Состояние кэшей или конфигураций.

## 5. Современные возможности (Java 17–21+)

### 5.1. VarHandle

`VarHandle` (Java 9+) предоставляет гибкий контроль доступа к переменным, включая `volatile`-семантику и атомарные операции.

**Пример**:

```java
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;

public class VarHandleExample {
    private int counter;
    private static final VarHandle COUNTER;

    static {
        try {
            COUNTER = MethodHandles.lookup().findVarHandle(VarHandleExample.class, "counter", int.class);
        } catch (Exception e) {
            throw new Error(e);
        }
    }

    public void increment() {
        COUNTER.getAndAdd(this, 1); // Атомарное увеличение
    }

    public int getCounter() {
        return (int) COUNTER.getVolatile(this); // volatile-чтение
    }
}
```

**Преимущества**:

- Атомарность для сложных операций (например, `compareAndSet`).
- `volatile`-семантика без явного модификатора.

### 5.2. Atomic Classes

Классы из `java.util.concurrent.atomic` (`AtomicInteger`, `AtomicReference`) обеспечивают атомарность и видимость.

**Пример**:

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // Атомарно
    }

    public int getCount() {
        return count.get();
    }
}
```

**Преимущества**:

- Полная атомарность для операций вроде `incrementAndGet`.
- Встроенная видимость, как у `volatile`.

### 5.3. Отложенная видимость: lazySet / setOpaque / setRelease

> [!INFO] Senior-паттерн: когда полный volatile fence избыточен
> Иногда нужна видимость "в итоге" (eventually visible), а не немедленная. Для этого существуют ослабленные операции записи — быстрее `volatile`, но со слабыми гарантиями.

```java
import java.util.concurrent.atomic.AtomicReference;
import java.lang.invoke.VarHandle;

// AtomicReference.lazySet (устарело в Java 9, но ещё используется)
// = VarHandle.setRelease — StoreStore барьер (NO StoreLoad!)
// Гарантирует: предыдущие записи видны ДО этой, но другой поток МОЖЕТ видеть
// новое значение не сразу (нет обязательной публикации cross-thread).
// Используется для: обнуления ссылок в пулах, очереди SPSC

AtomicReference<Object> ref = new AtomicReference<>();
ref.lazySet(null); // быстрее ref.set(null) для cleanup

// VarHandle.setOpaque — атомарно, coherent, без HB и ordering
// (одно поле видно последовательно — нет stale reads как у plain)
// Использование: progress indicators, счётчики для observability (не для синхронизации)

// Практический пример: очистка ссылки в thread pool worker
class Worker {
    private static final VarHandle TASK;
    static { /* lookup */ }

    volatile Object task;

    void complete() {
        TASK.setRelease(this, null); // быстрее volatile set, гарантирует видимость предыдущих writes
    }
}
```

**Сравнение производительности (нс/op, приблизительно, x86):**

| Операция | Latency | Throughput |
|----------|---------|------------|
| Plain `field = value` | ~1 нс | ~1000 мопс |
| `setOpaque` | ~2 нс | ~800 мопс |
| `setRelease` / `lazySet` | ~3 нс | ~600 мопс |
| `volatile` write | ~10–30 нс | ~50–200 мопс |
| `compareAndSet` | ~15–40 нс | ~30–80 мопс |

### 5.4. False Sharing и volatile

> [!WARNING] volatile НЕ защищает от False Sharing — это разные проблемы
> `volatile` решает видимость. False Sharing — это производительность кэша. Два volatile-поля в одной кэш-линии работают корректно, но МЕДЛЕННО.

```java
// BAD: оба поля в одной кэш-линии → каждый volatile write инвалидирует
// кэш другого ядра, хотя данные не пересекаются
class SharedState {
    volatile long reads  = 0; // обновляет Thread-1
    volatile long writes = 0; // обновляет Thread-2 (та же кэш-линия!)
}

// FIX: @Contended или padding
class PaddedState {
    @jdk.internal.vm.annotation.Contended volatile long reads  = 0;
    @jdk.internal.vm.annotation.Contended volatile long writes = 0;
    // Требует -XX:-RestrictContended для пользовательских классов
}
```

Подробнее: [[Модель памяти Java (JMM) и барьеры памяти#False Sharing и @Contended]]

### 5.5. Виртуальные потоки (Java 21)

`volatile` работает так же для виртуальных потоков, как и для платформенных. Барьеры памяти JMM применяются одинаково. Однако следует помнить: `volatile` не защищает от **пинирования** — для избежания пинирования при синхронизации используйте `ReentrantLock` вместо `synchronized`.

## 6. Подводные камни

1. **Неатомарность составных операций**:
    
    ```java
    volatile int count;
    count++; // Неатомарно
    ```
    
    **Решение**: Используйте `AtomicInteger` или `synchronized`.
    
2. **Избыточное использование `volatile`**:
    
    - `volatile` для редко изменяемых переменных увеличивает накладные расходы.
    - **Решение**: Ограничивайте `volatile` флагами состояния.
3. **Неправильное ожидание синхронизации**:
    
    - `volatile` не блокирует потоки, как `synchronized`.
    - **Решение**: Используйте `Lock` или `synchronized` для критических секций.
4. **Сериализация**:
    
    - `volatile` поля требуют явной обработки при сериализации.
    - **Решение**: Используйте `transient volatile` или кастомные `writeObject`/`readObject`.

## 7. Производительность

- **Чтение/запись `volatile`**:
    - Дороже обычных операций из-за барьеров памяти (O(1), но с накладными расходами).
    - Медленнее локального кэша CPU.
- **Сравнение**:
    - `volatile`: Подходит для простых операций (флаги, счетчики).
    - `synchronized`: Выше накладные расходы из-за блокировок.
    - `Atomic` классы: Оптимизированы для CAS (Compare-And-Swap), часто быстрее `synchronized`.
- **Рекомендации**:
    - Используйте `volatile` для флагов и простых операций.
    - Для сложных операций выбирайте `AtomicInteger` или `VarHandle`.

## 8. Decision Matrix: что выбрать?

| Ситуация | Инструмент | Почему |
|----------|-----------|--------|
| Флаг остановки потока | `volatile boolean` | Только visibility, нет contention |
| Счётчик с одним писателем | `volatile long` | Один writer = нет race condition |
| Счётчик с несколькими писателями | `AtomicLong` / `LongAdder` | CAS для атомарных инкрементов |
| Счётчик с высоким contention | `LongAdder` | Cell striping снижает contention |
| Публикация объекта | `volatile` ref или `final` поле | Happens-before для безопасной публикации |
| Очистка ссылки в пуле | `VarHandle.setRelease()` | Дешевле volatile, достаточно для cleanup |
| Observability счётчик | `VarHandle.setOpaque()` | Без overhead барьеров |
| Синхронизация нескольких полей | `synchronized` / `Lock` | volatile не гарантирует атомарность составных операций |

## 9. Лучшие практики

1. **Правило одного писателя**: `volatile` безопасен когда запись делает только один поток, читать могут многие.
2. **volatile ≠ synchronized**: для инвариантов между несколькими полями нужна синхронизация.
3. **Предпочитай `VarHandle` вместо `Unsafe`**: для lock-free структур используй `VarHandle` — безопасно, поддерживается JDK.
4. **Тест на видимость сложен**: используй `jcstress` (Java Concurrency Stress) для correctness-тестирования JMM-зависимого кода.

```java
// jcstress тест на видимость volatile:
@JCStressTest
@Outcome(id = "0, 1", expect = FORBIDDEN, desc = "Флаг без данных — нарушение JMM!")
@State
public class VolatileVisibilityTest {
    int data;
    volatile boolean flag;

    @Actor void writer() { data = 1; flag = true; }
    @Actor void reader(II_Result r) { r.r1 = flag ? 1 : 0; r.r2 = data; }
}
```