---
title: "Lock (java.util.concurrent.locks)"
tags: [java, concurrency, lock, reentrantlock, aqs, stampedlock, readwritelock]
updated: 2026-03-11
---

# Lock (java.util.concurrent.locks)

> [!QUOTE] Суть
> `Lock` — гибкая альтернатива `synchronized`. `ReentrantLock`: повторный вход, `tryLock()`, `lockInterruptibly()`. `ReadWriteLock`: много читателей / один писатель. `StampedLock` (Java 8+): оптимистичное чтение без блокировки. **Всегда** освобождай lock в `finally`.

> [!WARNING] Ловушка: не освободил Lock
> В отличие от `synchronized`, `ReentrantLock` НЕ освобождается автоматически при исключении. Паттерн обязателен: `lock.lock(); try { ... } finally { lock.unlock(); }`. Пропуск `unlock()` = вечный deadlock.

`Lock` — это **интерфейс** из пакета `java.util.concurrent.locks`, предоставляющий гибкий и мощный механизм синхронизации потоков, превосходящий возможности ключевого слова `synchronized`. Он позволяет явно управлять блокировками, поддерживает прерываемость, таймауты и условные переменные.

## 1. Почему `Lock`?

Интерфейс `Lock` предлагает расширенные возможности по сравнению с `synchronized`:

- **Явный контроль**: Ручной захват (`lock()`) и освобождение (`unlock()`) блокировки.
- **Прерываемость**: Возможность прерывать ожидание блокировки (`lockInterruptibly()`).
- **Таймауты**: Попытка захвата с ограничением времени (`tryLock(timeout, unit)`).
- **Условные переменные**: Поддержка `Condition` как альтернативы `wait/notify`.
- **Гибкость**: Разделение на чтение/запись (`ReadWriteLock`) и оптимистичное чтение (`StampedLock`).
- **Диагностика**: Проверка состояния блокировки (`isLocked()`, `hasQueuedThreads()`).

## 2. Основные реализации

|Реализация|Описание|
|---|---|
|`ReentrantLock`|Повторно входимая блокировка, поддерживает рекурсивный захват|
|`ReentrantReadWriteLock`|Разделяет доступ на чтение (множественное) и запись (эксклюзивное)|
|`StampedLock` (Java 8+)|Поддерживает оптимистичное чтение, возвращает штампы для проверки версии|

## 3. Внутреннее устройство

### 3.1. AbstractQueuedSynchronizer (AQS)

- **Основа**: Все реализации `Lock` используют `AbstractQueuedSynchronizer` (AQS).
- **Механизм**:
    - Поле `state` (volatile `int`) хранит состояние блокировки (0 — свободна, >0 — занята).
    - CLH-очередь (Craig, Landin, Hagersten) управляет ожидающими потоками.
    - Атомарные операции CAS (`compareAndSetState`) обеспечивают потокобезопасность.
- **LockSupport**: Использует `park()` для блокировки и `unpark()` для пробуждения потоков.

### 3.2. Работа `ReentrantLock`

- **Захват (`lock()`)**:
    1. Если `state == 0`, CAS устанавливает `state = 1`, поток становится владельцем.
    2. Если текущий поток уже владеет блокировкой, `state` увеличивается (reentrant).
    3. Если блокировка занята другим потоком, текущий добавляется в CLH-очередь и блокируется (`LockSupport.park()`).
- **Освобождение (`unlock()`)**:
    - Уменьшает `state`. Если `state == 0`, блокировка освобождается, и следующий поток из очереди пробуждается (`LockSupport.unpark()`).

### 3.3. JVM и JMM

- **JMM**: `lock()` и `unlock()` создают **happens-before** отношения, обеспечивая видимость изменений.
- **Байт-код (упрощённый)**:
    
    ```java
    invokevirtual java/util/concurrent/locks/ReentrantLock.lock()V
    // Захват через AQS, CAS или park()
    ```
    

## 4. Основные методы `Lock`

|Метод|Описание|
|---|---|
|`lock()`|Захватывает блокировку, блокируя поток до успеха|
|`unlock()`|Освобождает блокировку|
|`tryLock()`|Пытается захватить без ожидания, возвращает `boolean`|
|`tryLock(long timeout, TimeUnit unit)`|Пытается захватить с таймаутом|
|`lockInterruptibly()`|Захватывает с возможностью прерывания|
|`newCondition()`|Создаёт объект `Condition` для условного ожидания|

## 5. Сравнение с `synchronized`

|Характеристика|`synchronized`|`Lock` (`ReentrantLock`)|
|---|---|---|
|Явный контроль|Нет|Да (`lock()`/`unlock()`)|
|Прерывание ожидания|Нет|Да (`lockInterruptibly()`)|
|Таймаут|Нет|Да (`tryLock(timeout)`)|
|Условные переменные|`wait`/`notify` (на мониторе)|`Condition`|
|Рекурсивность|Да|Да|
|Производительность|Оптимизирована JVM|Лучше при высокой конкуренции|
|Проверка состояния|Нет|Да (`isLocked()`, `hasQueuedThreads()`)|

## 6. Использование `ReentrantLock`

**Пример: Потокобезопасный счётчик**

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Counter {
    private int count = 0;
    private final Lock lock = new ReentrantLock();

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

## 7. `Condition`

`Condition` — это объект, связанный с `Lock`, предоставляющий гибкую альтернативу `wait`/`notify`. Он позволяет создавать несколько условий ожидания для одной блокировки.

### 7.1. Основные методы `Condition`

- `await()`: Блокирует поток, освобождая блокировку, пока не поступит сигнал.
- `signal()`: Пробуждает один ожидающий поток.
- `signalAll()`: Пробуждает все ожидающие потоки.

### 7.2. Пример: Буфер с одним элементом

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.Condition;

public class SingleElementBuffer {
    private int value;
    private boolean hasValue = false;
    private final Lock lock = new ReentrantLock();
    private final Condition canWrite = lock.newCondition();
    private final Condition canRead = lock.newCondition();

    public void put(int val) throws InterruptedException {
        lock.lock();
        try {
            while (hasValue) {
                canWrite.await();
            }
            value = val;
            hasValue = true;
            System.out.println("Записано: " + val);
            canRead.signal();
        } finally {
            lock.unlock();
        }
    }

    public int get() throws InterruptedException {
        lock.lock();
        try {
            while (!hasValue) {
                canRead.await();
            }
            int result = value;
            hasValue = false;
            System.out.println("Прочитано: " + result);
            canWrite.signal();
            return result;
        } finally {
            lock.unlock();
        }
    }
}
```

**Объяснение**:

- `put()`: Ждёт, пока буфер освободится, затем записывает значение и сигнализирует о готовности чтения.
- `get()`: Ждёт, пока буфер заполнится, читает значение и сигнализирует о возможности записи.
- `finally`: Гарантирует освобождение блокировки.

### 7.3. Демонстрация

```java
public class Main {
    public static void main(String[] args) {
        SingleElementBuffer buffer = new SingleElementBuffer();

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    buffer.put(i);
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    buffer.get();
                    Thread.sleep(800);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

## 8. Deadlock с `ReentrantLock`

**Пример: Возможность deadlock**

```java
import java.util.concurrent.locks.ReentrantLock;

public class DeadlockExample {
    private final ReentrantLock lock1 = new ReentrantLock();
    private final ReentrantLock lock2 = new ReentrantLock();

    public void method1() {
        lock1.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " захватил lock1");
            Thread.sleep(100);
            System.out.println(Thread.currentThread().getName() + " пытается захватить lock2");
            lock2.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " захватил lock2");
            } finally {
                lock2.unlock();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock1.unlock();
        }
    }

    public void method2() {
        lock2.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " захватил lock2");
            Thread.sleep(100);
            System.out.println(Thread.currentThread().getName() + " пытается захватить lock1");
            lock1.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " захватил lock1");
            } finally {
                lock1.unlock();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock2.unlock();
        }
    }

    public static void main(String[] args) {
        DeadlockExample example = new DeadlockExample();
        Thread t1 = new Thread(example::method1, "T1");
        Thread t2 = new Thread(example::method2, "T2");
        t1.start();
        t2.start();
    }
}
```

**Объяснение**:

- `T1` захватывает `lock1` и ждёт `lock2`.
- `T2` захватывает `lock2` и ждёт `lock1`.
- Результат: Deadlock из-за циклической зависимости.

### 8.1. Обнаружение deadlock

- **Симптомы**: Программа зависает, потоки в состоянии `BLOCKED`.
- **Инструменты**:
    - `jstack <pid>`: Дамп потоков показывает `BLOCKED` и цепочки ожиданий.
    - `ThreadMXBean`: Программно проверяет deadlock через `findDeadlockedThreads()`.

### 8.2. Предотвращение deadlock

1. **Единый порядок захвата**:
    
    ```java
    public void method2() {
        lock1.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " захватил lock1");
            Thread.sleep(100);
            System.out.println(Thread.currentThread().getName() + " пытается захватить lock2");
            lock2.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " захватил lock2");
            } finally {
                lock2.unlock();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock1.unlock();
        }
    }
    ```
    
2. **Использование `tryLock` с таймаутом**:
    
    ```java
    public void method1() {
        try {
            if (lock1.tryLock(500, TimeUnit.MILLISECONDS)) {
                try {
                    System.out.println(Thread.currentThread().getName() + " захватил lock1");
                    if (lock2.tryLock(500, TimeUnit.MILLISECONDS)) {
                        try {
                            System.out.println(Thread.currentThread().getName() + " захватил lock2");
                        } finally {
                            lock2.unlock();
                        }
                    } else {
                        System.out.println(Thread.currentThread().getName() + " не смог захватить lock2");
                    }
                } finally {
                    lock1.unlock();
                }
            } else {
                System.out.println(Thread.currentThread().getName() + " не смог захватить lock1");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    ```
    

## 9. `ReadWriteLock` и `StampedLock`

### 9.1. `ReentrantReadWriteLock`

- Разделяет доступ на чтение (множественное) и запись (эксклюзивное).
- Подходит для сценариев, где чтение преобладает.

**Пример**:

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class DataStore {
    private String data = "";
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public void write(String value) {
        rwLock.writeLock().lock();
        try {
            data = value;
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public String read() {
        rwLock.readLock().lock();
        try {
            return data;
        } finally {
            rwLock.readLock().unlock();
        }
    }
}
```

### 9.2. `StampedLock`

- Поддерживает оптимистичное чтение, возвращая штамп для проверки версии.
- Подходит для сценариев с редкими изменениями.

**Пример**:

```java
import java.util.concurrent.locks.StampedLock;

public class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();
        double currentX = x, currentY = y;
        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

## 10. Практическое применение

- **Java EE / Jakarta EE**:
    - Синхронизация доступа к общим ресурсам в сервлетах.
    - Управление транзакциями в EJB.
- **Spring**:
    - Потокобезопасные сервисы для обработки запросов.
- **Пулы потоков**:
    - `ExecutorService` для синхронизации задач.
- **Кэши**:
    - Атомарное обновление в Caffeine или Guava Cache.

**Пример (Spring)**:

```java
@Service
public class SharedResource {
    private final Lock lock = new ReentrantLock();
    private int resource;

    public void updateResource(int value) {
        lock.lock();
        try {
            resource = value;
        } finally {
            lock.unlock();
        }
    }
}
```

## 11. Современные возможности (Java 21+)

### 11.1. Виртуальные потоки

`Lock` эффективен с виртуальными потоками (Java 21, Project Loom).

**Пример**:

```java
public class VirtualCounter {
    private final Lock lock = new ReentrantLock();
    private int count;

    public void increment() {
        Thread.ofVirtual().start(() -> {
            lock.lock();
            try {
                count++;
            } finally {
                lock.unlock();
            }
        });
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

### 11.2. Альтернативы

- **Атомарные классы**: Для простых операций без блокировок.
- **StampedLock**: Для оптимистичного чтения.
- **CompletableFuture**: Для асинхронной координации.

## 12. Подводные камни

1. **Неосвобождённая блокировка**:
    - Забытый `unlock()` вызывает deadlock.
    - **Решение**: Всегда используйте `finally`.
2. **Игнорирование `InterruptedException`**:
    
    ```java
    try {
        lock.lockInterruptibly();
    } catch (InterruptedException e) {
        // Игнорирование
    }
    ```
    
    **Решение**: Восстанавливайте флаг:
    
    ```java
    Thread.currentThread().interrupt();
    ```
    
3. **Сложность отладки**:
    - Deadlock трудно обнаружить без инструментов.
    - **Решение**: Используйте `jstack` или `ThreadMXBean`.
4. **Медленный `Condition`**:
    - Частое использование `await()`/`signal()` снижает производительность.
    - **Решение**: Рассмотрите атомарные классы для простых сценариев.

## 13. Производительность

- **Преимущества**:
    - Гибкость (`tryLock`, `lockInterruptibly`) улучшает отзывчивость.
    - `ReentrantLock` может быть быстрее `synchronized` при высокой конкуренции.
- **Недостатки**:
    - Явное управление увеличивает сложность.
    - CLH-очередь может замедлиться при большом числе потоков.
- **Сравнение**:
    - `synchronized`: Простота, но хуже при конкуренции.
    - `StampedLock`: Оптимизирован для чтения.
    - `Atomic*`: Lock-free, лучше для простых операций.
- **Рекомендации**:
    - Используйте `synchronized` для простых сценариев.
    - Выбирайте `Lock` для сложной синхронизации.

## 14. Плюсы и минусы

|Плюсы|Минусы|
|---|---|
|✅ Гибкость (`tryLock`, `Condition`)|❌ Сложность управления|
|✅ Прерываемость|❌ Риск deadlock|
|✅ Поддержка виртуальных потоков|❌ Сложность отладки|
|✅ Оптимистичное чтение (`StampedLock`)|❌ Требует явного `unlock()`|

## 15. Лучшие практики

1. **Освобождайте блокировку в `finally`**:
    
    ```java
    lock.lock();
    try {
        // Критическая секция
    } finally {
        lock.unlock();
    }
    ```
    
2. **Используйте `tryLock` для предотвращения deadlock**:
    
    ```java
    if (lock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            // Работа
        } finally {
            lock.unlock();
        }
    }
    ```
    
3. **Обрабатывайте исключения**:
    
    ```java
    try {
        lock.lockInterruptibly();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    ```
    
4. **Используйте `Condition` для сложной синхронизации**:
    
    ```java
    while (conditionNotMet) {
        condition.await();
    }
    ```
    
5. **Тестируйте синхронизацию**:
    
    ```java
    @Test
    void testLock() throws InterruptedException {
        Lock lock = new ReentrantLock();
        AtomicInteger counter = new AtomicInteger();
        Thread t1 = new Thread(() -> {
            lock.lock();
            try {
                counter.incrementAndGet();
            } finally {
                lock.unlock();
            }
        });
        Thread t2 = new Thread(() -> {
            lock.lock();
            try {
                counter.incrementAndGet();
            } finally {
                lock.unlock();
            }
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        assertEquals(2, counter.get());
    }
    ```
    

## 16. Заключение

Интерфейс `Lock` предоставляет мощный и гибкий механизм синхронизации, превосходящий `synchronized` за счёт прерываемости, таймаутов и `Condition`. Реализации (`ReentrantLock`, `ReentrantReadWriteLock`, `StampedLock`) подходят для различных сценариев, от простых счётчиков до оптимистичного чтения. Основанные на AQS, они обеспечивают потокобезопасность и производительность. Поддержка виртуальных потоков (Java 21+) и лучшие практики минимизируют риски deadlock и ошибок. `Lock` — это идеальный выбор для сложной синхронизации в многопоточных приложениях.

---

## 17. Senior Insights

### 17.1. AQS CLH Queue — внутренняя архитектура

AQS (AbstractQueuedSynchronizer) использует **модифицированную CLH очередь** (Craig–Landin–Hagersten). В классической CLH каждый узел спинит на флаге **предыдущего** узла. AQS делает это через `LockSupport.park()`, избегая busy-wait.

```
AQS state (volatile int):
  0 = unlocked
  1 = locked (ReentrantLock)
  N = locked N times (reentrant, same thread)

CLH Queue (двусвязный список):
  head ─► [Node(SIGNAL)] ─► [Node(SIGNAL)] ─► [Node(0)] = tail
            (освобождён)     (ожидает)          (новый)

Node.waitStatus:
  CANCELLED =  1  — поток прерван, узел вычеркнуть
  SIGNAL    = -1  — следующий узел нужно разбудить при unlock
  CONDITION = -2  — поток ждёт на Condition.await()
  PROPAGATE = -3  — shared lock: пробуждение распространить дальше
  0               — начальное состояние
```

**Последовательность `lock()` (unfair mode):**
```java
// ReentrantLock.NonfairSync.lock()
final void lock() {
    // 1. Barging: сразу пробуем захватить без очереди
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(currentThread());
    else
        acquire(1); // → tryAcquire → addWaiter → parkAndCheckInterrupt
}

// acquire(1) — шаблонный метод AQS:
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

**`acquireQueued` — цикл ожидания:**
```java
// Поток паркуется и просыпается только когда стал головой очереди
final boolean acquireQueued(final Node node, int arg) {
    for (;;) {
        final Node p = node.predecessor();
        if (p == head && tryAcquire(arg)) {  // только если мы следующие
            setHead(node);    // стать новым head
            p.next = null;    // GC старого head
            return interrupted;
        }
        if (shouldParkAfterFailedAcquire(p, node))
            parkAndCheckInterrupt(); // LockSupport.park(this)
    }
}
```

> [!INFO] Почему CLH эффективна
> Каждый поток спинит/паркуется на своём узле. При `unlock()` пробуждается только следующий узел (`LockSupport.unpark(successor)`). Нет thundering herd эффекта в отличие от `Object.notifyAll()`.

### 17.2. Fair vs Unfair — реальные трейдоффы

```java
// Unfair (по умолчанию)
new ReentrantLock();
new ReentrantLock(false);

// Fair
new ReentrantLock(true);
```

**Разница в `tryAcquire`:**
```java
// NonfairSync — barging разрешён:
protected boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires); // CAS без проверки очереди
}

// FairSync — строгий FIFO:
protected boolean tryAcquire(int acquires) {
    if (state == 0 && !hasQueuedPredecessors() && // ← ключевое отличие
        compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
    // reentrant...
}
```

| Режим | Throughput | Latency | Starvation |
|---|---|---|---|
| **Unfair** (default) | Высокий | Непредсказуемый | Возможно |
| **Fair** | ~20-30% ниже | Предсказуемый | Нет |

> [!WARNING] Ловушка: Fair ≠ справедливый по времени
> Fair lock гарантирует FIFO порядок *захвата*, но не равное время ожидания. При высокой конкуренции `hasQueuedPredecessors()` вызывается каждый раз — это дополнительный барьер памяти. Используй fair только если задержка критичнее throughput (real-time системы, координация heartbeat).

### 17.3. ReentrantReadWriteLock — write starvation

**Проблема:** При непрерывном потоке читателей писатель может голодать бесконечно.

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock(); // unfair!

// Читатели могут входить бесконечно, пока хоть один читает:
// Reader1 → Reader2 → Reader3 → ... → WriterThread ждёт вечно
```

**Решение 1: Fair RWLock**
```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock(true); // fair
// Писатель встаёт в очередь, новые читатели за ним не пролезут
// Цена: более низкий throughput чтения
```

**Решение 2: Lock downgrade (write → read)**
```java
// Паттерн: обновить кэш под write lock, затем опубликовать под read lock
class Cache {
    private volatile Map<String, Object> cache = new HashMap<>();
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock readLock  = rwl.readLock();
    private final Lock writeLock = rwl.writeLock();

    public Object get(String key) {
        readLock.lock();
        try {
            Object val = cache.get(key);
            if (val != null) return val;
        } finally {
            readLock.unlock();
        }

        // Downgrade: захват write → захват read → отпускаем write
        writeLock.lock();
        try {
            if (!cache.containsKey(key)) {      // double-check
                cache.put(key, loadFromDB(key));
            }
            readLock.lock(); // ← захватываем read ДО отпуска write
        } finally {
            writeLock.unlock(); // отпускаем write, но read уже держим
        }
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }
}
```

> [!WARNING] Lock upgrade запрещён!
> `readLock → writeLock` (upgrade) **вызывает deadlock**. Нужно отпустить read, потом захватить write. В промежутке другой поток может изменить состояние — проверяй инвариант заново.

### 17.4. StampedLock — детали и ловушки

**StampedLock НЕ реентрантен!**
```java
StampedLock sl = new StampedLock();
long stamp = sl.writeLock();
long stamp2 = sl.writeLock(); // DEADLOCK — поток заблокирует сам себя
```

**StampedLock НЕ поддерживает Condition:**
```java
sl.asReadWriteLock().readLock().newCondition(); // UnsupportedOperationException
// Если нужен Condition — используй ReentrantReadWriteLock
```

**Полный паттерн конвертации замков (optimistic → read → write):**
```java
public double distanceFromOrigin() {
    // Фаза 1: оптимистичное чтение (без блокировки)
    long stamp = sl.tryOptimisticRead();
    double x = this.x, y = this.y;

    // Фаза 2: если было изменение — fallback на read lock
    if (!sl.validate(stamp)) {
        stamp = sl.readLock();
        try {
            x = this.x;
            y = this.y;
        } finally {
            sl.unlockRead(stamp);
        }
    }
    return Math.sqrt(x * x + y * y);
}

// Конвертация read → write:
public void conditionalUpdate(double newX) {
    long stamp = sl.readLock();
    try {
        while (x < 0) {
            long writeStamp = sl.tryConvertToWriteLock(stamp);
            if (writeStamp != 0) {
                stamp = writeStamp; // конвертация успешна
                x = newX;
                break;
            } else {
                sl.unlockRead(stamp);
                stamp = sl.writeLock(); // явный захват write
            }
        }
    } finally {
        sl.unlock(stamp); // универсальный unlock
    }
}
```

**validate() — тонкость:**
```java
// validate() читает volatile поле — это memory barrier!
// Компилятор/CPU не переставит чтения x,y ДО validate()
long stamp = sl.tryOptimisticRead();
double x = this.x; // может быть torn read на 32-bit платформах!
double y = this.y;
if (!sl.validate(stamp)) { ... }
// На x86: работает, т.к. double read атомарен на 64-bit
// На 32-bit JVM: ОПАСНО — double не атомарен
```

| | ReentrantLock | ReentrantReadWriteLock | StampedLock |
|---|---|---|---|
| Реентрантность | ✅ | ✅ | ❌ |
| Condition | ✅ | ✅ | ❌ |
| Оптимистичное чтение | ❌ | ❌ | ✅ |
| Конвертация замков | ❌ | Lock downgrade only | ✅ |
| Производительность | Средняя | Хорошая при read >> write | Отличная при read >> write |
| Сложность | Низкая | Средняя | Высокая |

### 17.5. LockSupport.park/unpark — модель permit

`park()`/`unpark()` работают на основе **одного разрешения (permit)** на поток:

```
unpark(t) → permit = 1
park()    → если permit=1: потребляет, продолжает сразу
           → если permit=0: блокирует до unpark()

Ключевые свойства:
- permit НЕ накапливается: двойной unpark() = один permit
- park() может вернуться spuriously (без причины) — всегда в цикле while!
- park(blocker) принимает blocker для диагностики (jstack показывает объект)
```

```java
// ПРАВИЛЬНО — цикл while против spurious wakeup:
while (!condition) {
    LockSupport.park(this); // this = blocker для jstack
}

// НЕПРАВИЛЬНО — if вместо while:
if (!condition) {
    LockSupport.park(this); // может вернуться раньше времени
}
```

> [!INFO] Связано
> - [[Synchronized]] — Mark Word и ObjectMonitor vs AQS CLH
> - [[CAS и Unsafe]] — compareAndSetState внутри AQS
> - [[Модель памяти Java (JMM) и барьеры памяти]] — happens-before при lock/unlock