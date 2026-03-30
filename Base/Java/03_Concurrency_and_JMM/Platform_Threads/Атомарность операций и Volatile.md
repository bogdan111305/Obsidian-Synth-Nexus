# Атомарность операций и Volatile

> `volatile` — гарантия видимости и happens-before, но NOT атомарности. Барьеры памяти на уровне JIT.
> На интервью: чем volatile отличается от synchronized, когда volatile достаточно, weak ordering (lazySet/setRelease/setOpaque), False Sharing.

## Связанные темы
[[Модель памяти Java (JMM) и барьеры памяти]], [[CAS и Unsafe]], [[Atomic (java.util.concurrent.atomic)]], [[Java Monitor]]

---

## Атомарность в JVM

**Атомарные по спецификации JVM:**
- Чтение/запись `int`, `char`, `boolean`, `byte`, `short`, `float` — атомарны
- Ссылки на объекты — атомарны (32-bit и 64-bit JVM)
- `long` и `double` — **НЕ атомарны** на 32-bit JVM (two 32-bit writes), атомарны на 64-bit

```java
volatile long value; // volatile делает long/double атомарным на 32-bit JVM тоже
```

**Неатомарные составные операции:**
```java
volatile int count = 0;
count++; // НЕ атомарно: read → increment → write (три операции)
// Если Thread-1 и Thread-2 одновременно: потеряется одно приращение
```
Решение: `AtomicInteger.incrementAndGet()` или `synchronized`.

---

## volatile: что гарантирует

**Гарантирует:**
1. **Visibility** — запись в volatile поле немедленно видна всем потокам
2. **Happens-before** — volatile write HB перед volatile read того же поля (переупорядочение запрещено)

```java
// Паттерн publish-consume:
class DataHolder {
    volatile boolean ready = false;
    int data = 0;

    void producer() {
        data = 42;      // обычная запись
        ready = true;   // volatile write: StoreStore гарантирует data=42 видна ДО ready=true
    }

    void consumer() {
        if (ready) {    // volatile read: LoadLoad гарантирует data читается ПОСЛЕ ready
            // data гарантированно 42
        }
    }
}
```

**НЕ гарантирует:**
- Атомарность составных операций (`count++`, `a += b`)
- Атомарность записи нескольких полей за раз

Полный разбор HB-правил, DCL и VarHandle ordering: [[Модель памяти Java (JMM) и барьеры памяти]]

---

## JVM-реализация: барьеры памяти

`volatile` поле в байт-коде помечается флагом `ACC_VOLATILE`. JIT вставляет барьеры памяти:

```
// Запись: flag = true
putfield Example.flag:Z   // ACC_VOLATILE
// x86:  неявный StoreLoad (TSO — stores видны в порядке)
//       + MFENCE или LOCK XCHG для StoreLoad барьера
// ARM64: STLR (store-release)

// Чтение: boolean f = flag
getfield Example.flag:Z   // ACC_VOLATILE
// x86:  обычный load (TSO — reads уже acquire-семантика)
// ARM64: LDAR (load-acquire)
```

| Барьер | volatile write | volatile read |
|--------|---------------|---------------|
| StoreStore | ✅ | — |
| LoadStore | — | ✅ |
| LoadLoad | — | ✅ |
| **StoreLoad** | ✅ **(дорогой!)** | — |

На x86: `volatile write` дороже `volatile read` из-за StoreLoad (MFENCE ~100 тактов).
На ARM64: обе операции симметрично дороги (STLR + LDAR оба дороже обычных).

---

## Weak Ordering: lazySet / setOpaque / setRelease

Когда полный volatile fence избыточен — три уровня ослабленной записи:

```java
// lazySet (AtomicXxx) = VarHandle.setRelease — StoreStore барьер, NO StoreLoad
// Гарантирует: предыдущие записи видны до этой write
// НО: нет обязательной немедленной публикации cross-thread (видимость "eventual")
// Применение: очистка ссылок в пулах, SPSC-очереди

AtomicReference<Object> ref = new AtomicReference<>();
ref.lazySet(null);  // быстрее ref.set(null) при cleanup без необходимости немедленной публикации

// setOpaque (VarHandle) — атомарно, coherent для одного поля, без HB, без ordering
// Поток видит значение последовательно (нет stale reads как у plain), но нет happens-before
// Применение: progress indicators, observability счётчики (не для синхронизации!)

// Практический паттерн: cleanup ссылки в worker
class Worker {
    private static final VarHandle TASK;
    static {
        try { TASK = MethodHandles.lookup().findVarHandle(Worker.class, "task", Runnable.class); }
        catch (Exception e) { throw new Error(e); }
    }

    volatile Runnable task;

    void complete() {
        TASK.setRelease(this, null); // быстрее volatile write, достаточно для cleanup
    }
}
```

**Производительность (приблизительно, x86, нс/op):**

| Операция | Latency | Применение |
|----------|---------|------------|
| `plain field = value` | ~1 нс | Однопоточный код |
| `setOpaque` | ~2 нс | Observability, progress |
| `setRelease` / `lazySet` | ~3 нс | Cleanup, SPSC-очереди |
| `volatile` write | ~10–30 нс | Общий случай |
| `compareAndSet` | ~15–40 нс | Lock-free обновление |

---

## False Sharing и volatile

`volatile` НЕ защищает от False Sharing — это разные проблемы:
- `volatile` = корректность (visibility)
- False Sharing = производительность (кэш)

```java
// BAD: оба volatile-поля в одной 64-байт кэш-линии
// Запись Thread-1 в reads инвалидирует кэш ядра Thread-2 (который пишет writes)
class SharedStats {
    volatile long reads  = 0;  // обновляет Thread-1
    volatile long writes = 0;  // обновляет Thread-2 — та же кэш-линия!
}

// FIX: @Contended padding (128 байт вокруг поля)
class PaddedStats {
    @jdk.internal.vm.annotation.Contended volatile long reads  = 0;
    @jdk.internal.vm.annotation.Contended volatile long writes = 0;
    // JVM-флаг: -XX:-RestrictContended
}
```

В стандартной библиотеке: `LongAdder` использует `@Contended` внутри `Striped64.Cell[]`.
Диагностика: `async-profiler -e cache-misses`, `perf stat -e cache-misses,cache-references`.

---

## Decision Matrix: что выбрать?

| Ситуация | Инструмент | Почему |
|----------|-----------|--------|
| Флаг остановки потока | `volatile boolean` | Только visibility, нет contention |
| Счётчик с одним писателем | `volatile long` | Один writer = нет race condition |
| Счётчик с несколькими писателями | `AtomicLong` | CAS для атомарных инкрементов |
| Счётчик с высоким contention | `LongAdder` | Cell striping снижает contention |
| Публикация объекта | `volatile` ref или `final` поле | Happens-before при публикации |
| Очистка ссылки в пуле | `VarHandle.setRelease()` | Дешевле volatile, достаточно для cleanup |
| Observability счётчик | `VarHandle.setOpaque()` | Без overhead StoreLoad барьера |
| Инвариант между несколькими полями | `synchronized` / `Lock` | volatile не даёт атомарность составных операций |

---

## Вопросы на интервью

- Что гарантирует `volatile`? Что НЕ гарантирует?
- Почему `count++` небезопасен даже для volatile переменной?
- Чем `volatile` отличается от `synchronized` с точки зрения JMM?
- Какие барьеры памяти вставляет JIT для `volatile` write и read? Почему write дороже на x86?
- Что такое `lazySet`? Чем отличается от `volatile set`?
- Чем `setRelease` отличается от `setOpaque`? Когда что применять?
- Почему два volatile-поля в одном объекте могут быть медленнее одного? Как исправить?
- Как проверить корректность JMM-зависимого кода? (jcstress)

---

## Подводные камни

- **`volatile` ≠ атомарность** — `count++` на volatile-переменной не потокобезопасен. Race condition на read-modify-write. Нужен `AtomicInteger` или `synchronized`.
- **`volatile` не блокирует** — не подходит для "критических секций" где нужна атомарность нескольких полей. Только для publish/consume паттерна.
- **`long`/`double` без `volatile` на 32-bit JVM** — torn read: можно прочитать верхние 32 бита от одного write и нижние от другого. На 64-bit JVM атомарны, но `volatile` всё равно нужен для visibility.
- **False Sharing** — два volatile-поля рядом в объекте: корректно, но медленно. Проверяй layout через JOL.
- **Pinning в virtual threads** — `volatile` не вызывает pinning. Но `synchronized`, даже для simple volatile write+check, — вызывает. Используй `ReentrantLock`.
- **`lazySet`/`setRelease` без понимания** — без StoreLoad барьера другой поток может видеть старое значение дольше ожидаемого. Безопасно только для cleanup (обнуление ссылок), не для передачи данных producer→consumer.
