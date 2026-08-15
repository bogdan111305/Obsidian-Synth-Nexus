# Модель памяти Java (JMM) и барьеры памяти

> JMM (Java Memory Model, JSR-133, Java 5+) определяет когда изменения одного потока видны другим. Без JMM гарантий компилятор и CPU могут переставлять инструкции и кэшировать данные локально.  
> На интервью: правила happens-before, почему DCL без volatile сломан, что такое safe publication, False Sharing.

## Связанные темы
[[Атомарность операций и Volatile]], [[Java Monitor]], [[CAS и Unsafe]], [[Atomic (java.util.concurrent.atomic)]], [[Java Memory Structure]]

---

## Почему нужен JMM

Без явной синхронизации два потока могут видеть разные значения одной переменной:
- **CPU кэши** — каждое ядро кэширует данные локально
- **Reordering** — компилятор/JIT/CPU переставляет инструкции ради оптимизации

JMM задаёт когда именно изменения становятся видимы. Инструмент — отношение **happens-before**.

---

## Правила Happens-Before

Если A happens-before B: все записи в A (и до A) видны в B. Отношение транзитивно.

| Правило | A | B |
|---|---|---|
| Программный порядок | любая операция в потоке | следующая операция в том же потоке |
| Monitor | `unlock()` монитора | `lock()` того же монитора в любом потоке |
| Volatile | запись в volatile-поле | любое последующее чтение того же поля |
| Thread.start | всё до `t.start()` | первая операция потока `t` |
| Thread.join | все операции потока `t` | возврат из `t.join()` |
| Final fields | запись final-поля в конструкторе | любое чтение объекта после публикации |
| CAS | успешный `compareAndSet` | последующие чтения через `get()` |

```java
// Без HB — видимость не гарантирована:
int value = 0;
boolean ready = false;

// Thread 1:
value = 42;   // может быть переставлено ПОСЛЕ ready=true компилятором
ready = true;

// Thread 2:
if (ready) {
    System.out.println(value); // может вывести 0 !
}
```

---

## Memory barriers

JMM реализуется через барьеры памяти — инструкции процессора, запрещающие перестановку операций:

| Барьер | Что запрещает | Стоимость на x86 |
|---|---|---|
| `LoadLoad` | перестановку двух чтений | бесплатен (TSO) |
| `LoadStore` | чтение переставить после записи | бесплатен |
| `StoreStore` | перестановку двух записей | бесплатен |
| `StoreLoad` | запись переставить до чтения | дорогой (`MFENCE` / `LOCK XCHG`) |

**Где JVM вставляет барьеры:**
- `volatile write` → StoreStore + **StoreLoad** (дорого!)
- `volatile read` → LoadLoad + LoadStore
- `synchronized` enter → LoadLoad + LoadStore
- `synchronized` exit → StoreStore + StoreLoad
- `final` поля в конструкторе → StoreStore после последней записи

---

## Volatile: safe publication

```java
// Паттерн publish/consume через volatile:
class DataHolder {
    volatile boolean ready = false;
    int data = 0;

    void producer() {
        data = 42;      // обычная запись
        ready = true;   // volatile write: StoreStore гарантирует data=42 видна ДО ready=true
    }

    void consumer() {
        if (ready) {    // volatile read: LoadLoad гарантирует data читается ПОСЛЕ ready
            System.out.println(data); // гарантированно 42
        }
    }
}
```

Volatile НЕ гарантирует атомарность составных операций: `count++` = три шага, volatile не поможет.

---

## Double-Checked Locking

```java
// СЛОМАНО без volatile:
class BrokenSingleton {
    private static BrokenSingleton instance;

    public static BrokenSingleton getInstance() {
        if (instance == null) {                    // 1. проверка без синхронизации
            synchronized (BrokenSingleton.class) {
                if (instance == null) {
                    instance = new BrokenSingleton(); // JIT/CPU может переставить:
                    // 1) выделить память
                    // 2) записать ссылку в instance  ← другой поток видит не-null!
                    // 3) вызвать конструктор          ← объект не инициализирован
                }
            }
        }
        return instance; // может вернуть частично инициализированный объект!
    }
}

// ИСПРАВЛЕНО: volatile создаёт HB между записью и чтением instance
class CorrectSingleton {
    private static volatile CorrectSingleton instance;

    public static CorrectSingleton getInstance() {
        if (instance == null) {
            synchronized (CorrectSingleton.class) {
                if (instance == null) {
                    instance = new CorrectSingleton();
                    // volatile write: конструктор ВСЕГДА завершается до записи в instance
                }
            }
        }
        return instance;
    }
}

// ЛУЧШИЙ СПОСОБ: Initialization-on-demand holder (no volatile, no sync overhead)
class BestSingleton {
    private static class Holder {
        static final BestSingleton INSTANCE = new BestSingleton();
        // Класс загружается при первом обращении — ClassLoader гарантирует thread-safety
        // static final = happens-before через ClassLoader lock
    }

    public static BestSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## Final fields и безопасная публикация

JMM специально гарантирует: запись в `final` поле в конструкторе HB любому чтению объекта — **если объект опубликован корректно**.

```java
class ImmutablePoint {
    final int x;
    final int y;

    ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
        // JVM вставляет StoreStore здесь — x и y видны после публикации
    }
}

// Способы безопасной публикации (любой из):
// 1. volatile поле:          volatile ImmutablePoint p = new ImmutablePoint(1, 2);
// 2. synchronized блок
// 3. final поле контейнера:  class C { final ImmutablePoint p = ...; }
// 4. static инициализатор:   static ImmutablePoint P = new ImmutablePoint(1, 2);
// 5. concurrent collection:  queue.put(new ImmutablePoint(1, 2));
```

> [!WARNING] Escaped reference — объект опубликован до завершения конструктора
> ```java
> class Unsafe {
>     static Unsafe instance;
>     Unsafe() {
>         instance = this; // ЛОВУШКА! Другой поток видит instance до окончания конструктора
>         expensiveInit(); // если упадёт — instance указывает на неинициализированный объект
>     }
> }
> ```

---

## False Sharing и @Contended

Два volatile-поля в одной кэш-линии (64 байта) → запись в одно инвалидирует кэш для другого:

```java
// ПЛОХО: counter0 и counter1 в одной 64-байт cache line
class BadCounters {
    volatile long counter0; // байты 8-15 в объекте
    volatile long counter1; // байты 16-23 — та же cache line!
}
// Thread-1 пишет counter0 → инвалидирует cache line на ядре Thread-2
// Thread-2 промахивается в L1 при чтении counter1 — хотя его поле не менялось!

// ХОРОШО: @Contended добавляет padding до 128 байт вокруг поля
class GoodCounters {
    @jdk.internal.vm.annotation.Contended
    volatile long counter0;

    @jdk.internal.vm.annotation.Contended
    volatile long counter1;
}
// Требует JVM-флаг: -XX:-RestrictContended
```

В продакшне встречается в: счётчиках запросов, флагах состояния потоков, LongAdder (Cell[] с @Contended внутри Striped64).

Диагностика: `async-profiler -e cache-misses`, `perf stat -e cache-misses,cache-references`.

---

## VarHandle: режимы ordering

Java 9+ — тонкий контроль над гарантиями памяти:

| Режим | Read | Write | Барьеры | Применение |
|---|---|---|---|---|
| **Plain** | `get()` | `set()` | нет | однопоточный код |
| **Opaque** | `getOpaque()` | `setOpaque()` | атомарность, нет HB | индикаторы прогресса |
| **Acquire/Release** | `getAcquire()` | `setRelease()` | LoadLoad / StoreStore | publish-consume |
| **Volatile** | `getVolatile()` | `setVolatile()` | полный fence | общий случай |

**Паттерн Acquire/Release — дешевле volatile на ARM:**
```java
// Producer:
DATA.setRelease(this, value); // StoreStore: все записи ДО видны consumer'у

// Consumer:
int v = (int) DATA.getAcquire(this); // LoadLoad: все последующие чтения корректны
// Если getAcquire видит значение после setRelease → все записи producer'а видны
```

На x86 разницы нет (TSO гарантирует порядок store). На ARM (`AWS Graviton`): `setRelease` = `stlr`, volatile write = `stlr + dmb ish` — ощутимый выигрыш в tight loops.

---

## JMM и Virtual Threads (Java 21+)

Виртуальные потоки полностью соблюдают JMM — все happens-before правила применяются одинаково.

**Нюанс: pinning при `synchronized`** — виртуальный поток прикрепляется к carrier thread на время удержания монитора. Если carrier thread заблокирован — он недоступен для других виртуальных потоков. `volatile` и `ReentrantLock` не вызывают pinning.

**Unmount/mount стека** — при блокировке виртуального потока его стек сохраняется в Heap, при возобновлении — восстанавливается. JMM гарантирует корректную видимость через happens-before на операции mount.

---

## Вопросы на интервью

- Что такое happens-before? Перечисли основные правила.
- Почему Double-Checked Locking без volatile сломан? Что именно может пойти не так?
- Что гарантирует `volatile` — атомарность или видимость? Чем они отличаются?
- Что такое safe publication? Назови 4 способа безопасно опубликовать объект.
- Что такое False Sharing? Как диагностировать и исправить?
- Чем `setRelease`/`getAcquire` отличается от `volatile`? На каких архитектурах это важно?
- Как JMM работает с виртуальными потоками? Что такое pinning?
- Может ли `final` поле наблюдаться в некорректном состоянии? При каких условиях?

## Подводные камни

- **`volatile` не атомарен** — `count++` это три операции. Нужен `AtomicInteger`.
- **DCL без volatile** — в Java до 5 сломан, с Java 5+ нужен `volatile`. Лучше — holder-паттерн.
- **Escaped reference в конструкторе** — публикация `this` до окончания конструктора: другой поток видит частично инициализированный объект.
- **False sharing** — два volatile-поля рядом в одном объекте могут разрушить производительность. Проверяй layout через JOL.
- **Pinning в виртуальных потоках** — `synchronized` + блокирующий I/O = carrier thread заблокирован. Используй `ReentrantLock`.
