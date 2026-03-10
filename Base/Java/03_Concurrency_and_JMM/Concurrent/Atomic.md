# Atomic (java.util.concurrent.atomic)

> [!QUOTE] Суть
> **Atomic-классы** — lock-free операции через **CAS** (Compare-And-Swap), реализованный на уровне процессора. Без `synchronized`, без блокировок. `AtomicInteger`, `AtomicLong`, `AtomicReference`, `LongAdder` (лучше для high-contention). Java 9+: `VarHandle` вместо `Unsafe`.

Пакет `java.util.concurrent.atomic` предоставляет **атомарные классы**, которые позволяют выполнять **безблокировочные (lock-free)** операции над переменными (инкремент, установка, замена) без использования `synchronized`. Эти классы основаны на низкоуровневых атомарных инструкциях процессора (CAS, Compare-And-Swap) через `sun.misc.Unsafe` или `VarHandle` (Java 9+). Они обеспечивают высокую производительность и масштабируемость в многопоточных приложениях. Эта статья подробно рассматривает атомарные классы, их реализацию, примеры, преимущества, ограничения, производительность и современные возможности (Java 21+).

## 1. Что такое атомарные классы?

- **Определение**: Классы из пакета `java.util.concurrent.atomic` предоставляют API для атомарных операций над переменными без явных блокировок.
- **Назначение**:
    - Обеспечивают **атомарность** операций (например, инкремент, замена).
    - Поддерживают **lock-free** и **wait-free** алгоритмы.
    - Гарантируют видимость изменений через **volatile** (Java Memory Model, JMM).
- **Основа**: Используют CAS (Compare-And-Swap), реализованный на уровне процессора (`cmpxchg` на x86, `ldrex/strex` на ARM).

## 2. Основные классы

|Класс|Тип данных|Особенности|
|---|---|---|
|`AtomicInteger`|`int`|Инкремент, декремент, установка, замена|
|`AtomicLong`|`long`|Аналог `AtomicInteger` для 64-битных чисел|
|`AtomicBoolean`|`boolean`|Атомарные операции над флагами (например, переключение состояния)|
|`AtomicReference<V>`|Любой объект|Атомарная замена ссылок на объекты|
|`AtomicStampedReference<V>`|Объект + `int`|Решает ABA-проблему с помощью версий (stamp)|
|`AtomicMarkableReference<V>`|Объект + `boolean`|Альтернатива для простых меток (например, пометка удаления)|
|`AtomicIntegerArray`|`int[]`|Атомарные операции над массивом целых чисел|
|`AtomicLongArray`|`long[]`|Атомарные операции над массивом длинных чисел|
|`AtomicReferenceArray<V>`|Объект[]|Атомарные операции над массивом ссылок|

**Пример использования**:

```java
AtomicInteger counter = new AtomicInteger(0);

// Инкремент
counter.incrementAndGet(); // ++counter
counter.getAndIncrement(); // counter++

// Установка
counter.set(10);

// Атомарная замена
boolean success = counter.compareAndSet(10, 20); // Успех, если текущее значение 10
```

## 3. Основные операции

|Метод|Описание|
|---|---|
|`get()`|Получает текущее значение (volatile read)|
|`set(newValue)`|Устанавливает новое значение (volatile write)|
|`lazySet(newValue)`|Устанавливает значение с возможной задержкой (оптимизация для слабой консистентности)|
|`getAndSet(value)`|Устанавливает новое значение, возвращает старое|
|`incrementAndGet()`|Увеличивает на 1, возвращает новое значение|
|`getAndIncrement()`|Возвращает старое значение, затем увеличивает на 1|
|`addAndGet(delta)`|Прибавляет delta, возвращает новое значение|
|`compareAndSet(expected, update)`|Атомарно заменяет значение, если текущее равно expected|
|`getAndUpdate(UnaryOperator)`|Применяет функцию к текущему значению, возвращает старое (Java 8+)|
|`updateAndGet(UnaryOperator)`|Применяет функцию, возвращает новое значение (Java 8+)|

**Пример**:

```java
AtomicInteger counter = new AtomicInteger(0);
counter.updateAndGet(x -> x * 2 + 1); // Умножает текущее значение на 2 и прибавляет 1
```

## 4. Как работает CAS в атомарных классах?

- **Основа**: Атомарные классы используют CAS через `sun.misc.Unsafe` (до Java 9) или `VarHandle` (Java 9+).
- **Логика CAS**:
    
    ```java
    if (memoryAddress == expectedValue) {
        memoryAddress = newValue;
        return true;
    } else {
        return false;
    }
    ```
    
- **Спин-цикл**: Если CAS не удался, операция повторяется:
    
    ```java
    int current = get();
    int next = current + 1;
    while (!compareAndSet(current, next)) {
        current = get();
        next = current + 1;
    }
    ```
    

## 5. JVM-реализация

- **CAS**:
    - Вызывает нативные инструкции процессора через `Unsafe` или `VarHandle`.
    - Примеры: `cmpxchg` (x86), `ldrex/strex` (ARM).
- **JMM**:
    - Использует `volatile` для чтения/записи, обеспечивая видимость.
    - Устанавливает **happens-before** отношения.
- **Байт-код (упрощённый)**:
    
    ```java
    invokevirtual java/util/concurrent/atomic/AtomicInteger.compareAndSet(II)Z
    // Вызывает native-метод через Unsafe/VarHandle
    ```
    
- **VarHandle (Java 9+)**:
    - Замена `Unsafe` для безопасного низкоуровневого доступа.
    - Пример:
        
        ```java
        VarHandle handle = MethodHandles.lookup().in(AtomicInteger.class)
            .findVarHandle(AtomicInteger.class, "value", int.class);
        handle.compareAndSet(counter, expected, update);
        ```
        

## 6. Преимущества атомарных классов

1. **Высокая производительность**:
    - Lock-free операции минимизируют задержки.
    - Не требуют системных вызовов (в отличие от `synchronized`).
2. **Отсутствие deadlock**:
    - CAS исключает взаимные блокировки.
3. **Простота**:
    - Удобный API для атомарных операций.
4. **Аппаратная поддержка**:
    - Оптимизировано под процессорные инструкции.

## 7. Проблемы и ограничения

### 7.1. ABA-проблема

- **Описание**: Значение меняется с `A` на `B` и обратно на `A`, CAS считает операцию успешной.
- **Решение**: Используйте `AtomicStampedReference`:
    
    ```java
    AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
    int[] stampHolder = new int[1];
    String val = ref.get(stampHolder);
    ref.compareAndSet("A", "X", stampHolder[0], stampHolder[0] + 1);
    ```
    

### 7.2. Конкуренция

- Спин-циклы нагружают CPU при высокой конкуренции.
- **Решение**: Используйте `LongAdder` для счётчиков.

### 7.3. Ограниченность

- Атомарность только для одного значения.
- **Решение**: Комбинируйте с `synchronized` или `Lock` для сложных операций.

## 8. Расширенные классы

### 8.1. `LongAdder` и `DoubleAdder`

- Оптимизированы для высокой конкуренции.
- Разбивают счётчик на ячейки (cells), снижая конфликты.
- **Пример**:
    
    ```java
    LongAdder adder = new LongAdder();
    adder.increment();
    long sum = adder.sum(); // Суммирует все ячейки
    ```
    

### 8.2. `LongAccumulator` и `DoubleAccumulator`

- Обобщение `LongAdder` для произвольных операций.
- **Пример**:
    
    ```java
    LongAccumulator acc = new LongAccumulator(Long::max, 0);
    acc.accumulate(10); // Сохраняет максимум
    ```
    

### 8.3. `Atomic*Array`

- Атомарные операции над массивами.
- **Пример**:
    
    ```java
    AtomicIntegerArray array = new AtomicIntegerArray(10);
    array.compareAndSet(0, 0, 1);
    ```
    

## 9. Практическое применение

- **Java EE / Jakarta EE**:
    - Атомарные счётчики для метрик запросов.
    - Управление состоянием в EJB.
- **Spring**:
    - Атомарные операции в `@Service` для кэшей или счётчиков.
- **Структуры данных**:
    - `ConcurrentHashMap`, `ConcurrentLinkedQueue` используют CAS.
- **Пулы потоков**:
    - `ForkJoinPool` для управления задачами.
- **Кэши**:
    - Атомарное обновление в Caffeine или Guava Cache.

**Пример (Spring)**:

```java
@Service
public class MetricsService {
    private final LongAdder requestCounter = new LongAdder();

    public void recordRequest() {
        requestCounter.increment();
    }

    public long getRequestCount() {
        return requestCounter.sum();
    }
}
```

## 10. Современные возможности (Java 21+)

### 10.1. Виртуальные потоки

Атомарные классы эффективны в виртуальных потоках (Java 21, Project Loom).

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

### 10.2. Сравнение с другими механизмами

- **`ReentrantLock`**: Гибкость, но требует явного управления.
- **`StampedLock`**: Оптимистические блокировки для чтения.
- **STM**: Атомарность для нескольких значений (экспериментально).

## 11. Подводные камни

1. **ABA-проблема**:
    - Игнорирование промежуточных изменений.
    - **Решение**: Используйте `AtomicStampedReference`.
2. **Высокая конкуренция**:
    - Спин-циклы нагружают CPU.
    - **Решение**: Используйте `LongAdder` или ограничьте попытки.
3. **Ограниченность**:
    - CAS не подходит для сложных транзакций.
    - **Решение**: Комбинируйте с `Lock`.
4. **Неправильное использование `lazySet`**:
    - Может нарушить видимость.
    - **Решение**: Используйте `set()` для строгой консистентности.

## 12. Производительность

- **Преимущества**:
    - CAS быстрее `synchronized` при низкой конкуренции.
    - `LongAdder` масштабируется при высокой конкуренции.
- **Недостатки**:
    - Спин-циклы нагружают CPU.
    - ABA-проблема требует дополнительных проверок.
- **Сравнение**:
    - `synchronized`: Простота, высокие затраты при конкуренции.
    - `ReentrantLock`: Гибкость, поддержка `tryLock`.
    - `LongAdder`: Оптимизирован для счётчиков.
- **Рекомендации**:
    - Используйте `Atomic*` для простых операций.
    - Переходите на `LongAdder` при высокой конкуренции.

## 13. Плюсы и минусы

|Плюсы|Минусы|
|---|---|
|✅ Высокая производительность|❌ ABA-проблема|
|✅ Lock-free алгоритмы|❌ Нагрузка CPU при конкуренции|
|✅ Простота API|❌ Ограниченность одним значением|
|✅ Поддержка виртуальных потоков|❌ Сложность отладки ABA|

## 14. Лучшие практики

1. **Используйте `Atomic*` для простых операций**:
    
    ```java
    AtomicInteger counter = new AtomicInteger();
    counter.incrementAndGet();
    ```
    
2. **Применяйте `LongAdder` при высокой конкуренции**:
    
    ```java
    LongAdder adder = new LongAdder();
    adder.increment();
    ```
    
3. **Используйте `AtomicStampedReference` для ABA**:
    
    ```java
    ref.compareAndSet(val, newVal, stamp, stamp + 1);
    ```
    
4. **Ограничивайте спин-циклы**:
    
    ```java
    int retries = 0;
    while (!counter.compareAndSet(current, next) && retries++ < 100);
    ```
    
5. **Тестируйте атомарность**:
    
    ```java
    @Test
    void testAtomicCounter() throws InterruptedException {
        LongAdder adder = new LongAdder();
        Thread t1 = new Thread(() -> { for (int i = 0; i < 1000; i++) adder.increment(); });
        Thread t2 = new Thread(() -> { for (int i = 0; i < 1000; i++) adder.increment(); });
        t1.start(); t2.start();
        t1.join(); t2.join();
        assertEquals(2000, adder.sum());
    }
    ```
    

## 15. Пример: Потокобезопасная очередь

```java
import java.util.concurrent.atomic.AtomicReference;

public class LockFreeQueue<T> {
    private static class Node<T> {
        T value;
        AtomicReference<Node<T>> next = new AtomicReference<>(null);

        Node(T value) {
            this.value = value;
        }
    }

    private final AtomicReference<Node<T>> head = new AtomicReference<>();
    private final AtomicReference<Node<T>> tail = new AtomicReference<>();

    public void enqueue(T value) {
        Node<T> newNode = new Node<>(value);
        for (;;) {
            Node<T> currentTail = tail.get();
            if (currentTail == null) {
                if (head.compareAndSet(null, newNode) && tail.compareAndSet(null, newNode)) {
                    return;
                }
            } else {
                if (currentTail.next.compareAndSet(null, newNode) && tail.compareAndSet(currentTail, newNode)) {
                    return;
                }
            }
        }
    }

    public T dequeue() {
        for (;;) {
            Node<T> currentHead = head.get();
            if (currentHead == null) {
                return null;
            }
            Node<T> next = currentHead.next.get();
            if (head.compareAndSet(currentHead, next)) {
                return currentHead.value;
            }
        }
    }
}
```

**Особенности**:

- Использует CAS для атомарного обновления `head` и `tail`.
- Подвержен ABA-проблеме (решение: добавить `AtomicStampedReference`).

## 16. Заключение

Пакет `java.util.concurrent.atomic` предоставляет мощные инструменты для безблокировочных операций в многопоточных приложениях. Основанные на CAS, атомарные классы (`AtomicInteger`, `AtomicReference`, `LongAdder`) обеспечивают высокую производительность и отсутствие deadlock. Однако ABA-проблема и конкуренция требуют осторожности. Современные возможности, такие как `VarHandle` и виртуальные потоки (Java 21+), расширяют их применимость. Следование лучшим практикам и тестирование позволяют создавать надёжные и эффективные приложения.