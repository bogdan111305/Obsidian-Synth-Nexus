# Lock (java.util.concurrent.locks)

> `Lock` — явная альтернатива `synchronized` с поддержкой `tryLock()`, прерывания, таймаутов и `Condition`. Основан на AQS (AbstractQueuedSynchronizer).  
> На интервью: разница fair/unfair, write starvation в RWLock, ловушки StampedLock, как работает AQS внутри.

## Связанные темы
[[Java Monitor]], [[CAS и Unsafe]], [[Модель памяти Java (JMM) и барьеры памяти]], [[Atomic (java.util.concurrent.atomic)]]

---

## Lock vs synchronized

| Возможность | `synchronized` | `ReentrantLock` |
|---|---|---|
| Автоосвобождение | ✅ | ❌ — обязателен `finally` |
| `tryLock()` / таймаут | ❌ | ✅ |
| Прерывание ожидания | ❌ | ✅ `lockInterruptibly()` |
| Несколько Condition | ❌ | ✅ `newCondition()` |
| Диагностика | ❌ | ✅ `isLocked()`, `hasQueuedThreads()` |
| Fair-режим | ❌ | ✅ `new ReentrantLock(true)` |

**Единственный обязательный паттерн:**
```java
lock.lock();
try {
    // критическая секция
} finally {
    lock.unlock(); // НЕ пропускать — иначе вечный deadlock
}
```

---

## Основные методы

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();                              // блокирует до захвата
lock.lockInterruptibly();                 // то же, но можно прервать через interrupt()
lock.tryLock();                           // захват без ожидания → boolean
lock.tryLock(500, TimeUnit.MILLISECONDS); // с таймаутом
lock.unlock();                            // освобождение (только владелец!)

// Condition — мощная замена wait/notify:
Condition notEmpty = lock.newCondition();
notEmpty.await();             // отпускает lock и ждёт сигнала
notEmpty.signal();            // будит один ожидающий поток
notEmpty.signalAll();         // будит все
notEmpty.await(1, SECONDS);  // с таймаутом
```

**Пример: producer-consumer с двумя Condition:**
```java
class BoundedBuffer<T> {
    private final Queue<T> queue = new ArrayDeque<>();
    private final int capacity;
    private final Lock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) notFull.await(); // ждём места
            queue.add(item);
            notEmpty.signal(); // сигналим consumer'у
        } finally { lock.unlock(); }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) notEmpty.await(); // ждём элемента
            T item = queue.poll();
            notFull.signal(); // сигналим producer'у
            return item;
        } finally { lock.unlock(); }
    }
}
```

---

## AQS: внутреннее устройство

`ReentrantLock`, `Semaphore`, `CountDownLatch` — все построены на `AbstractQueuedSynchronizer`.

```
AQS state (volatile int):
  0 = unlocked
  1 = locked (ReentrantLock unfair)
  N = locked N раз (reentrant, тот же поток)

CLH Queue (модифицированная, двусвязный список):
  head → [Node(SIGNAL)] → [Node(SIGNAL)] → [Node(0)] = tail
         (уже free)        (ожидает)         (новый)

Node.waitStatus:
  CANCELLED =  1  — поток прерван/timed out, узел выбросить
  SIGNAL    = -1  — при unlock() разбудить следующий узел
  CONDITION = -2  — поток ждёт на Condition.await()
  PROPAGATE = -3  — shared lock: пробуждение передать дальше
  0               — начальное состояние нового узла
```

**Последовательность `lock()` (unfair):**
```java
// Шаг 1: barging — пробуем захватить без очереди
if (compareAndSetState(0, 1))
    setExclusiveOwnerThread(currentThread()); // захвачено!
else
    acquire(1); // встаём в очередь

// Шаг 2: acquire → addWaiter → acquireQueued
final boolean acquireQueued(Node node, int arg) {
    for (;;) {
        if (node.predecessor() == head && tryAcquire(arg)) {
            setHead(node);      // стали головой
            return interrupted;
        }
        if (shouldParkAfterFailedAcquire(p, node))
            parkAndCheckInterrupt(); // LockSupport.park(this) — поток спит
    }
}
```

Нет thundering herd: при `unlock()` пробуждается только один следующий узел через `LockSupport.unpark(successor)`.

---

## Fair vs Unfair

```java
new ReentrantLock();       // unfair (по умолчанию)
new ReentrantLock(true);   // fair
```

Разница в `tryAcquire`:
```java
// Unfair: barging — новый поток может "вклиниться" перед очередью
if (state == 0 && compareAndSetState(0, 1)) { ... } // захват без проверки очереди

// Fair: строгий FIFO
if (state == 0 && !hasQueuedPredecessors() && compareAndSetState(0, 1)) { ... }
//                 ^^^ лишний барьер памяти при каждой попытке
```

| | Unfair | Fair |
|---|---|---|
| Throughput | Высокий | ~20-30% ниже |
| Latency | Непредсказуемый | Предсказуемый |
| Starvation | Возможно (редко) | Нет |

> [!WARNING] Fair ≠ равное время ожидания
> Fair гарантирует FIFO порядок *захвата*, не равное время. Используй только если предсказуемость важнее throughput (real-time, heartbeat координация).

---

## ReentrantReadWriteLock

Много читателей одновременно, но запись — эксклюзивна:

```java
ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

// Чтение (параллельно с другими читателями):
rwl.readLock().lock();
try { return data; }
finally { rwl.readLock().unlock(); }

// Запись (эксклюзивно):
rwl.writeLock().lock();
try { data = newData; }
finally { rwl.writeLock().unlock(); }
```

**Write starvation:** при unfair RWLock непрерывный поток читателей может голодать писателя бесконечно. Решения:
1. `new ReentrantReadWriteLock(true)` — fair, но снижает read throughput
2. Lock downgrade

**Lock downgrade (write → read):**
```java
writeLock.lock();
try {
    if (!cache.containsKey(key)) {
        cache.put(key, loadFromDB(key));
    }
    readLock.lock();       // захватываем read ДО отпуска write!
} finally {
    writeLock.unlock();    // теперь отпускаем write, но read уже держим
}
try {
    return cache.get(key);
} finally {
    readLock.unlock();
}
```

> [!WARNING] Lock upgrade (read → write) вызывает deadlock
> Нельзя захватить writeLock, держа readLock. Нужно: отпустить readLock → захватить writeLock → перечитать состояние.

---

## StampedLock

Самый производительный, но и самый сложный. Три режима:

```java
StampedLock sl = new StampedLock();

// 1. Write lock:
long stamp = sl.writeLock();
try { x = newX; }
finally { sl.unlockWrite(stamp); }

// 2. Read lock:
long stamp = sl.readLock();
try { return x; }
finally { sl.unlockRead(stamp); }

// 3. Optimistic read (без блокировки!):
long stamp = sl.tryOptimisticRead();
double localX = x, localY = y;
if (!sl.validate(stamp)) {     // проверяем — не было ли записи?
    stamp = sl.readLock();     // было → fallback на read lock
    try { localX = x; localY = y; }
    finally { sl.unlockRead(stamp); }
}
return Math.sqrt(localX * localX + localY * localY);
```

**Конвертация read → write через `tryConvertToWriteLock`:**
```java
long stamp = sl.readLock();
try {
    while (x < 0) {
        long ws = sl.tryConvertToWriteLock(stamp); // атомарная попытка
        if (ws != 0) {
            stamp = ws;
            x = newX;
            break;
        } else {
            sl.unlockRead(stamp);    // не получилось — отпускаем и берём write явно
            stamp = sl.writeLock();
        }
    }
} finally {
    sl.unlock(stamp); // универсальный unlock
}
```

| | ReentrantLock | ReentrantReadWriteLock | StampedLock |
|---|---|---|---|
| Реентрантность | ✅ | ✅ | ❌ |
| Condition | ✅ | ✅ | ❌ |
| Оптимистичное чтение | ❌ | ❌ | ✅ |
| Конвертация lock | ❌ | только downgrade | ✅ |
| Сложность | Низкая | Средняя | Высокая |

> [!WARNING] StampedLock — критические ловушки
> - **Не реентрантен**: повторный `writeLock()` в том же потоке → мгновенный deadlock
> - **Не поддерживает Condition**: `asReadWriteLock().readLock().newCondition()` → `UnsupportedOperationException`
> - **Optimistic read на 32-bit JVM**: `double` не атомарен → torn read возможен. На 64-bit x86 безопасно.

---

## LockSupport.park/unpark

Низкоуровневый примитив, на котором построены AQS и Condition:

```
Модель: каждый поток имеет ОДИН permit (0 или 1)

unpark(thread) → permit = 1 (накопление не работает — двойной unpark = один permit)
park()         → если permit=1: забирает permit и возвращает сразу
               → если permit=0: блокирует поток до unpark()

Важно: park() может вернуться spuriously — ВСЕГДА в цикле while:
```

```java
// ПРАВИЛЬНО:
while (!condition) {
    LockSupport.park(this); // this = blocker для jstack/debugger
}

// НЕПРАВИЛЬНО:
if (!condition) {
    LockSupport.park(this); // spurious wakeup → пропуск сигнала
}
```

---

## Вопросы на интервью

- Зачем `Lock` если есть `synchronized`? Назови три сценария, где `Lock` необходим.
- Что произойдёт, если забыть вызвать `unlock()`?
- Чем fair ReentrantLock отличается от unfair? Почему unfair быстрее?
- Что такое write starvation? Как решить без fair-режима?
- Почему lock upgrade (read→write) вызывает deadlock?
- Что такое AQS? Опиши CLH очередь и `waitStatus`.
- Почему StampedLock не реентрантен? Чем это опасно?
- Как работает оптимистичное чтение в StampedLock?
- Может ли `park()` вернуться без `unpark()`? Как это обработать?

## Подводные камни

- **Забытый `unlock()`** — в отличие от `synchronized`, исключение в критической секции не освобождает Lock. `finally` обязателен.
- **Lock upgrade → deadlock** — захват writeLock при удерживаемом readLock блокирует поток навсегда.
- **StampedLock не реентрантен** — один поток не может дважды взять writeLock.
- **Condition.await() без while** — spurious wakeup пройдёт мимо проверки условия. Только `while (!condition) { await(); }`.
- **Fair lock в продакшне** — `-20-30%` throughput. Включай только если точно нужно.
- **`validate()` до чтения полей** — на 32-bit JVM `double`/`long` не атомарны, оптимистичное чтение может дать torn values.
