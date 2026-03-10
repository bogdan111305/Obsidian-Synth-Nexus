# Compare-And-Swap (CAS) в Java

> [!QUOTE] Суть
> **CAS** (Compare-And-Swap): атомарно сравни текущее значение с ожидаемым, если совпадает — обнови. Операция одна (`cmpxchg` на x86). Основа всех lock-free структур. Проблема: **ABA** (значение изменилось с A→B→A, CAS не заметит). Решение: `AtomicStampedReference`.

> [!WARNING] Ловушка: проблема ABA
> Поток T1 читает значение A. Поток T2 меняет A→B→A. T1 делает CAS: видит A, думает ничего не изменилось — успех! Но состояние уже другое. Решение: `AtomicStampedReference` добавляет версию/штамп к значению.

**Как работает CAS пошагово:**
1. Читаем текущее значение: `current = *addr` (например, 5)
2. Сравниваем с ожидаемым: `current == expected`? (5 == 5 ✓)
3. Если совпадает — атомарно записываем новое: `*addr = newValue` (записываем 6)
4. Возвращаем `true` (успех)
5. Если не совпадает — возвращаем `false`, повторяем (spin)

**CAS (Compare-And-Swap)** — это атомарная операция, реализованная на уровне процессора, которая позволяет безопасно обновлять значение в памяти, только если оно совпадает с ожидаемым. CAS лежит в основе высокопроизводительных неблокирующих алгоритмов и структур данных в Java.

## 1. Что такое CAS?

- **Определение**: CAS — это неделимая операция, которая сравнивает текущее значение в памяти с ожидаемым и обновляет его, только если они совпадают.
- **Назначение**:
    - Обеспечивает **атомарность** без блокировок (lock-free).
    - Используется для синхронизации в многопоточных приложениях.
- **Логика работы**:
    
    ```java
    if (memoryAddress == expectedValue) {
        memoryAddress = newValue;
        return true;
    } else {
        return false;
    }
    ```
    
    - Параметры: `expectedValue`, `newValue`, `memoryAddress`.
    - Возвращает `true` при успешной замене, `false` при несовпадении.

## 2. CAS в Java

В Java CAS реализуется через класс `sun.misc.Unsafe` и высокоуровневые классы из пакета `java.util.concurrent.atomic`.

### 2.1. `sun.misc.Unsafe`

- Предоставляет низкоуровневые операции, включая CAS:
    
    ```java
    unsafe.compareAndSwapInt(object, offset, expected, update);
    ```
    
    - `object`: Объект, поле которого обновляется.
    - `offset`: Смещение поля в байтах.
    - `expected`: Ожидаемое значение.
    - `update`: Новое значение.

**Примечание**: `Unsafe` — внутренний API, не рекомендуется для прямого использования. Начиная с Java 9 (модульная система Jigsaw), доступ к `sun.misc.Unsafe` ограничен. В Java 17+ попытка прямого использования без специальных флагов (`--add-opens`) генерирует предупреждение. Вместо него с Java 9 доступен `java.lang.invoke.VarHandle`.

### 2.2. Пакет `java.util.concurrent.atomic`

- Классы, такие как `AtomicInteger`, `AtomicLong`, `AtomicReference`, используют CAS для атомарных операций.
- Пример метода `AtomicInteger.compareAndSet`:
    
    ```java
    public boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    ```
    

## 3. Пример: `AtomicInteger.incrementAndGet()`

Метод `incrementAndGet()` использует CAS для атомарного инкремента:

```java
public final int incrementAndGet() {
    for (;;) {
        int current = get();           // Читаем текущее значение (volatile)
        int next = current + 1;       // Вычисляем новое значение
        if (compareAndSet(current, next)) // Пытаемся атомарно заменить
            return next;              // Успех
    }
}
```

- **Спин-цикл**: Повторяет попытки, пока CAS не удастся.
- **Volatile**: Гарантирует видимость изменений (JMM).

## 4. Преимущества CAS

1. **Высокая производительность**:
    - Не требует перехода в режим ядра ОС (в отличие от `synchronized`).
    - Низкая задержка при низкой конкуренции.
2. **Lock-free алгоритмы**:
    - Позволяет создавать неблокирующие структуры данных (например, `ConcurrentLinkedQueue`).
    - Исключает deadlock.
3. **Аппаратная поддержка**:
    - Реализован через инструкции процессора:
        - x86: `cmpxchg` (Compare and Exchange).
        - ARM: `ldrex/strex` (Load-Link/Store-Conditional).
    - Гарантирует атомарность на уровне шины.

## 5. Проблемы и ограничения CAS

### 5.1. ABA-проблема

- **Описание**: Значение в памяти меняется с `A` на `B` и обратно на `A`. CAS считает операцию успешной, хотя значение изменялось.
- **Пример**:
    
    ```java
    AtomicReference<String> ref = new AtomicReference<>("A");
    // Thread-1
    String val = ref.get(); // "A"
    // Thread-2
    ref.set("B");
    ref.set("A"); // Возвращает "A"
    // Thread-1
    ref.compareAndSet("A", "X"); // Успех, но значение менялось!
    ```
    
- **Риск**: Неправильная логика в lock-free структурах (например, в стеках).

### 5.2. Решение ABA: `AtomicStampedReference`

- Хранит **значение** и **метку (stamp)** для отслеживания версий.
- Пример:
    
    ```java
    AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
    // Thread-1
    int[] stampHolder = new int[1];
    String val = ref.get(stampHolder); // val = "A", stamp = 0
    int stamp = stampHolder[0];
    // Thread-2
    ref.set("B", stamp + 1);
    ref.set("A", stamp + 2);
    // Thread-1
    ref.compareAndSet("A", "X", stamp, stamp + 1); // Провал, stamp не совпадает
    ```
    

**Методы `AtomicStampedReference`**:

|Метод|Описание|
|---|---|
|`get(stampHolder)`|Возвращает значение, заполняет stampHolder|
|`set(value, stamp)`|Устанавливает значение и версию|
|`compareAndSet(expectedVal, newVal, expectedStamp, newStamp)`|CAS с проверкой версии|
|`attemptStamp(expectedVal, newStamp)`|Обновляет версию при совпадении значения|

### 5.3. Спин-блокировка

- CAS использует спин-циклы (`for(;;)`), повторяя попытки.
- **Проблема**: Высокая конкуренция увеличивает нагрузку на CPU.
- **Решение**: Ограничивать количество попыток или использовать `Lock`.

### 5.4. Ограниченность

- CAS обновляет **одно значение** атомарно.
- Для нескольких полей требуется `synchronized`, `Lock`, или STM (Software Transactional Memory).

## 6. JVM-реализация CAS

- **Механизм**:
    - `Unsafe.compareAndSwap*` вызывает JNI для нативного кода.
    - JVM транслирует вызов в инструкцию процессора (`cmpxchg` на x86, `ldrex/strex` на ARM).
- **JMM (Java Memory Model)**:
    - CAS использует `volatile` для чтения/записи, обеспечивая видимость.
    - Устанавливает **happens-before** отношения.
- **Байт-код** (упрощённый):
    
    ```java
    invokevirtual java/util/concurrent/atomic/AtomicInteger.compareAndSet(II)Z
    // Вызывает native-метод через Unsafe
    ```
    

## 7. Практическое применение

- **Java EE / Jakarta EE**:
    - CAS в `ConcurrentHashMap` для обработки запросов.
    - Атомарные счётчики в EJB.
- **Spring**:
    - Атомарные операции в `@Service` для управления состоянием.
- **Структуры данных**:
    - `ConcurrentLinkedQueue`, `LinkedTransferQueue` используют CAS для добавления/удаления элементов.
- **Пулы потоков**:
    - `ForkJoinPool` использует CAS для управления задачами.
- **Кэши**:
    - Атомарное обновление кэшей (например, в Caffeine).

**Пример (Spring)**:

```java
@Service
public class CounterService {
    private final AtomicInteger counter = new AtomicInteger(0);

    public int incrementAndGet() {
        return counter.incrementAndGet();
    }
}
```

## 8. Современные возможности (Java 21+)

### 8.1. CAS в виртуальных потоках

Виртуальные потоки (Java 21, Project Loom) используют CAS аналогично платформенным потокам.

**Пример**:

```java
public class VirtualAtomicCounter {
    private final AtomicInteger counter = new AtomicInteger(0);

    public void increment() {
        Thread.ofVirtual().start(() -> counter.incrementAndGet());
    }

    public int get() {
        return counter.get();
    }
}
```

**Особенности**:

- CAS эффективен для I/O-интенсивных задач в виртуальных потоках.
- Минимизирует накладные расходы благодаря лёгким потокам.

### 8.2. Сравнение с другими механизмами

- **ReentrantLock**: Гибкость, но требует явного управления.
- **StampedLock**: Оптимистические блокировки для чтения.
- **STM (Software Transactional Memory)**: Атомарное обновление нескольких значений (экспериментально).

## 9. Подводные камни

1. **ABA-проблема**:
    
    - Игнорирование промежуточных изменений.
    - **Решение**: Используйте `AtomicStampedReference`.
2. **Высокая конкуренция**:
    
    - Спин-циклы увеличивают нагрузку на CPU.
    - **Решение**: Ограничивайте попытки или используйте `Lock`.
3. **Ограниченность CAS**:
    
    - Не подходит для сложных транзакций.
    - **Решение**: Комбинируйте с `synchronized` или `Lock`.
4. **Неправильное использование `Unsafe`**:
    
    - Прямой доступ к `Unsafe` опасен.
    - **Решение**: Используйте `Atomic*` классы.

## 10. Производительность

- **Преимущества**:
    - CAS быстрее `synchronized` при низкой конкуренции.
    - Не требует системных вызовов (в отличие от fat monitors).
- **Недостатки**:
    - Спин-циклы нагружают CPU при высокой конкуренции.
    - ABA-проблема требует дополнительных проверок.
- **Сравнение**:
    - `synchronized`: Простота, но высокие затраты при конкуренции.
    - `ReentrantLock`: Гибкость, поддержка `tryLock`.
    - `Atomic*`: Высокая производительность для простых операций.
- **Рекомендации**:
    - Используйте CAS для коротких операций.
    - Переходите на `Lock` при сложной логике.

## 11. Плюсы и минусы

|Плюсы|Минусы|
|---|---|
|✅ Высокая производительность|❌ ABA-проблема|
|✅ Lock-free алгоритмы|❌ Нагрузка CPU при конкуренции|
|✅ Аппаратная атомарность|❌ Ограничен одним значением|
|✅ Поддержка в виртуальных потоках|❌ Сложность отладки ABA|

## 12. Лучшие практики

1. **Используйте `Atomic*` классы**:
    
    ```java
    AtomicInteger counter = new AtomicInteger();
    counter.incrementAndGet();
    ```
    
2. **Применяйте `AtomicStampedReference` для ABA**:
    
    ```java
    ref.compareAndSet(val, newVal, stamp, stamp + 1);
    ```
    
3. **Ограничивайте спин-циклы**:
    
    ```java
    int retries = 0;
    while (!atomic.compareAndSet(current, next) && retries++ < MAX_RETRIES);
    ```
    
4. **Комбинируйте с `Lock` для сложных операций**:
    
    ```java
    lock.lock();
    try {
        // Обновление нескольких полей
    } finally {
        lock.unlock();
    }
    ```
    
5. **Тестируйте атомарность**:
    
    ```java
    @Test
    void testAtomicCounter() throws InterruptedException {
        AtomicInteger counter = new AtomicInteger(0);
        Thread t1 = new Thread(() -> { for (int i = 0; i < 1000; i++) counter.incrementAndGet(); });
        Thread t2 = new Thread(() -> { for (int i = 0; i < 1000; i++) counter.incrementAndGet(); });
        t1.start(); t2.start();
        t1.join(); t2.join();
        assertEquals(2000, counter.get());
    }
    ```
    

## 13. Пример: Потокобезопасный стек

```java
import java.util.concurrent.atomic.AtomicReference;

public class LockFreeStack<T> {
    private final AtomicReference<Node<T>> head = new AtomicReference<>();

    private static class Node<T> {
        T value;
        Node<T> next;

        Node(T value, Node<T> next) {
            this.value = value;
            this.next = next;
        }
    }

    public void push(T value) {
        Node<T> newNode = new Node<>(value, null);
        for (;;) {
            Node<T> currentHead = head.get();
            newNode.next = currentHead;
            if (head.compareAndSet(currentHead, newNode)) {
                return;
            }
        }
    }

    public T pop() {
        for (;;) {
            Node<T> currentHead = head.get();
            if (currentHead == null) {
                return null;
            }
            if (head.compareAndSet(currentHead, currentHead.next)) {
                return currentHead.value;
            }
        }
    }
}
```

**Особенности**:

- Использует CAS для атомарного обновления `head`.
- Подвержен ABA-проблеме (решение: добавить `AtomicStampedReference`).

## Senior Insights

### VarHandle вместо Unsafe — почему это важно

```java
// BAD: sun.misc.Unsafe — нестабильный internal API
import sun.misc.Unsafe;
private static final Unsafe UNSAFE;
static {
    try {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        UNSAFE = (Unsafe) f.get(null);
    } catch (Exception e) { throw new ExceptionInInitializerError(e); }
}
// Работает до Java 8, в Java 9+ требует --add-opens, в Java 17+ предупреждения

// SENIOR WAY: java.lang.invoke.VarHandle (Java 9+)
import java.lang.invoke.*;

public class ModernAtomicCounter {
    private volatile int count = 0;

    private static final VarHandle COUNT;
    static {
        try {
            COUNT = MethodHandles.lookup()
                .findVarHandle(ModernAtomicCounter.class, "count", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    // CAS:
    public boolean cas(int expected, int update) {
        return COUNT.compareAndSet(this, expected, update);
    }

    // Атомарный инкремент:
    public int getAndIncrement() {
        return (int) COUNT.getAndAdd(this, 1);
    }

    // Гибкий memory ordering (нет в Unsafe!):
    public int getAcquire() { return (int) COUNT.getAcquire(this); }
    public void setRelease(int v) { COUNT.setRelease(this, v); }
    public int getPlain() { return (int) COUNT.get(this); } // нет барьера
}
```

**Преимущества VarHandle vs Unsafe:**
- Стабильный публичный API (`java.lang.invoke`)
- Типобезопасность (проверяется при создании VarHandle)
- Гибкие memory ordering: plain, opaque, acquire/release, volatile
- JIT инлайнит как нативные операции
- Работает с Java Security Manager

### JMM happens-before: тонкости для CAS

```
CAS и happens-before (JSR-133):
- Успешный compareAndSet() на AtomicInteger устанавливает happens-before:
  Все действия ДО успешного CAS в потоке A
  happens-before
  Все действия ПОСЛЕ успешного CAS того же объекта в потоке B

Важно: НЕУСПЕШНЫЙ CAS не создаёт happens-before!

Пример:
Thread A: if (ref.compareAndSet(null, value)) { /* публикуем value */ }
Thread B: Object v = ref.get(); // если видит value → CAS был успешен → HB гарантирует видимость value

Это основа safe publication через AtomicReference!
```

### False Sharing в CAS-based структурах

```java
// BAD: Счётчики рядом → false sharing на одной cache line
class SharedCounters {
    volatile long counter1; // cache line 0-7 байт
    volatile long counter2; // cache line 8-15 байт — В ТОЙ ЖЕ 64-байт cache line!
    // Thread-1 инкрементирует counter1 → инвалидирует всю cache line
    // Thread-2 промахивается в cache для counter2 → жутко медленно!
}

// SENIOR WAY: LongAdder использует @Contended внутри
LongAdder adder1 = new LongAdder();
LongAdder adder2 = new LongAdder();
// Каждый LongAdder держит свои ячейки на отдельных cache lines

// Или явное padding:
@jdk.internal.vm.annotation.Contended
class PaddedCounter {
    volatile long value; // JVM добавляет padding до 128 байт вокруг
}
// Требует: -XX:-RestrictContended
```

## 14. Заключение

CAS (Compare-And-Swap) — мощный инструмент для создания высокопроизводительных неблокирующих алгоритмов в Java. Он обеспечивает атомарность через аппаратные инструкции и лежит в основе классов `java.util.concurrent.atomic`. Несмотря на преимущества (производительность, отсутствие deadlock), CAS имеет ограничения, такие как ABA-проблема и нагрузка на CPU при конкуренции. Использование `AtomicStampedReference`, высокоуровневых примитивов и виртуальных потоков (Java 21+) позволяет создавать надёжные и эффективные многопоточные приложения. Следование лучшим практикам и тестирование обеспечивают корректность и производительность.