# CAS и Unsafe

> CAS (Compare-And-Swap) — атомарная инструкция процессора: сравни текущее значение с ожидаемым, если совпадает — замени. Основа всех lock-free структур в Java.  
> На интервью: как работает на разных архитектурах, чем `compareAndExchange` лучше `compareAndSet`, зачем `weakCompareAndSet`, что такое False Sharing.

## Связанные темы
[[Atomic (java.util.concurrent.atomic)]], [[Модель памяти Java (JMM) и барьеры памяти]], [[Lock]], [[Java Monitor]]

---

## Как работает CAS

```java
// Логика (атомарно — не расщепляется между потоками):
if (memoryAddress == expected) {
    memoryAddress = newValue;
    return true; // или возвращает oldValue
} else {
    return false; // или возвращает current (witness)
}
```

Спин-цикл вокруг CAS — основа всех атомарных операций:
```java
// Реализация incrementAndGet внутри AtomicInteger:
int current, next;
do {
    current = get();       // volatile read
    next = current + 1;
} while (!compareAndSet(current, next)); // повтор если другой поток успел раньше
return next;
```

---

## CPU-уровень: x86 vs ARM

```
x86: CMPXCHG — одна атомарная инструкция, spurious failure невозможен
  if (EAX == [mem]) { [mem] = reg; ZF=1; }
  else              { EAX = [mem]; ZF=0; }

ARM64: LDXR + STXR — Load-Exclusive / Store-Conditional
  1. LDXR x0, [addr]       — читаем + помечаем cache line как "exclusive"
  2. (вычисления)
  3. STXR w1, x2, [addr]   — записываем ТОЛЬКО если cache line не инвалидирована
     → w1 = 0 (success) или 1 (failure)

Spurious failure на ARM: STXR может вернуть failure, даже если наше значение
не менялось — достаточно чтобы другой поток записал ЧТО УГОДНО в ту же cache line.
Поэтому на ARM retry-loop обязателен даже для сильного CAS.
```

**Практическое следствие:** `weakCompareAndSet` на x86 = `compareAndSet` (JIT компилирует одинаково). На ARM — `weakCompareAndSet` без retry дешевле, но может ложно вернуть `false`.

---

## compareAndSet vs compareAndExchange vs weakCompareAndSet

| Метод | Возвращает | Spurious failure? | Memory order | Когда использовать |
|---|---|---|---|---|
| `compareAndSet(e, v)` | `boolean` | Нет | volatile (full) | Общий случай |
| `compareAndExchange(e, v)` | witness value | Нет | volatile (full) | Когда нужно фактическое значение без лишнего `get()` |
| `compareAndExchangeAcquire` | witness | Нет | acquire | Начало critical section |
| `compareAndExchangeRelease` | witness | Нет | release | Конец critical section |
| `weakCompareAndSet(e, v)` | `boolean` | **Да** | opaque | Плотный спин-цикл на ARM |

**Пример: `compareAndExchange` экономит одно volatile-чтение при промахе:**
```java
// compareAndSet — при промахе нужен дополнительный get():
Node expected = head.get();
while (!head.compareAndSet(expected, newNode)) {
    expected = head.get(); // лишнее volatile-чтение
}

// compareAndExchange — witness IS новый expected:
Node expected = head.get();
Node witness;
while ((witness = HEAD_VH.compareAndExchange(this, expected, newNode)) != expected) {
    expected = witness; // фактическое значение вернул сам CAS
}
```

---

## VarHandle vs Unsafe

`sun.misc.Unsafe` — нестабильный internal API. С Java 9 требует `--add-opens`, с Java 17 — предупреждения при доступе через рефлексию. **Не использовать в новом коде.**

`VarHandle` (Java 9+) — стабильная замена:

```java
public class LockFreeCounter {
    private volatile int count = 0;

    private static final VarHandle COUNT;
    static {
        try {
            COUNT = MethodHandles.lookup()
                .findVarHandle(LockFreeCounter.class, "count", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    // Все режимы ordering — недоступны через Atomic-классы напрямую:
    public int getPlain()    { return (int) COUNT.get(this); }          // нет барьера
    public int getOpaque()   { return (int) COUNT.getOpaque(this); }    // атомарность, без HB
    public int getAcquire()  { return (int) COUNT.getAcquire(this); }   // LoadLoad барьер
    public int getVolatile() { return (int) COUNT.getVolatile(this); }  // полный fence

    public boolean cas(int expected, int update) {
        return COUNT.compareAndSet(this, expected, update);
    }

    // compareAndExchange — возвращает witnessed value:
    public int cae(int expected, int update) {
        return (int) COUNT.compareAndExchange(this, expected, update);
    }
}
```

> [!WARNING] VarHandle создавать дорого
> Создание `VarHandle` через `MethodHandles.lookup()` требует рефлексии. Всегда объявляй в `static final`, не в методе.

---

## ABA-проблема

CAS видит значение A и не знает, что между его чтением и CAS-ом было A→B→A.

**Где критично:** lock-free стеки и очереди, где pop() берёт узел, а потом тот же узел push()-ается обратно. CAS по ссылке увидит старую ссылку и посчитает, что ничего не изменилось — хотя `next` у узла уже другой.

```java
// Lock-free стек — классическая ABA:
// Thread 1: читает head = A (A.next = B)
// Thread 2: pop A, pop B, push A обратно (теперь A.next = null)
// Thread 1: CAS(head, A, B) — успех! Но B уже не в стеке, head.next = null → потеря данных

// Решение: AtomicStampedReference
AtomicStampedReference<Node> head = new AtomicStampedReference<>(initialNode, 0);

int[] stamp = new int[1];
Node current = head.get(stamp);
Node next = current.next;
// CAS успешен только если И ссылка совпадает И stamp совпадает:
head.compareAndSet(current, next, stamp[0], stamp[0] + 1);
```

---

## False Sharing в lock-free структурах

Несколько volatile/atomic полей в одном объекте могут разделять cache line (64 байта) — тогда CAS одного поля инвалидирует кэш для всех остальных:

```java
// ПЛОХО: counter1 и counter2 в одной cache line
class BadCounters {
    volatile long counter1; // байты 0-7
    volatile long counter2; // байты 8-15 — та же 64-байт cache line!
}
// Поток 1 инкрементирует counter1 → инвалидирует cache line для потока 2
// Поток 2 вынужден заново загружать counter2 из памяти → деградация

// ХОРОШО: @Contended добавляет padding до 128 байт
@jdk.internal.vm.annotation.Contended
class PaddedCounter {
    volatile long value;
}
// Требует JVM-флаг: -XX:-RestrictContended

// В продакшне: LongAdder использует @Contended на каждой Cell внутри Striped64
```

---

## JMM и CAS: happens-before

Успешный `compareAndSet` устанавливает happens-before:

```
Все действия потока A до успешного CAS
    happens-before
Все действия потока B, который наблюдает результат этого CAS через get()
```

**Важно:** неуспешный CAS НЕ создаёт happens-before.

```java
// Safe publication через AtomicReference:
AtomicReference<Config> ref = new AtomicReference<>(null);

// Thread A (publisher):
Config cfg = buildConfig();          // все записи в cfg...
ref.compareAndSet(null, cfg);        // ...видны после успешного CAS

// Thread B (consumer):
Config seen = ref.get();
if (seen != null) {
    seen.getValue(); // HB гарантирует — все поля cfg инициализированы корректно
}
```

---

## Вопросы на интервью

- Что такое spurious failure? На каких архитектурах возникает и почему?
- Чем `compareAndExchange` лучше `compareAndSet`? Приведи пример.
- Почему `weakCompareAndSet` быстрее на ARM?
- Почему нельзя использовать `Unsafe` в Java 17+? Чем заменить?
- Что такое False Sharing? Как `LongAdder` с ним борется?
- Создаёт ли неуспешный CAS happens-before?
- В чём ABA-проблема в lock-free стеке? Как исправить?
- Что такое witness value в `compareAndExchange`?

## Подводные камни

- **Spurious failure на ARM** — `weakCompareAndSet` может вернуть `false` без реальной конкуренции. Всегда в цикле.
- **False Sharing** — `volatile long a; volatile long b;` рядом = одна cache line = деградация производительности. Используй `@Contended` или `LongAdder`.
- **Неуспешный CAS не создаёт HB** — нельзя использовать как барьер при промахе.
- **ABA в структурах с переиспользованием узлов** — `AtomicReference` не защищает. Нужен `AtomicStampedReference`.
- **`Unsafe` в Java 17+** — требует `--add-opens` и генерирует warning. В Java 21+ планируется к ограничению. Мигрируй на `VarHandle`.
