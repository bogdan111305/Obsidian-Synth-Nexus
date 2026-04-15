# Java Monitor: synchronized, wait/notify

> `synchronized` захватывает монитор объекта — только один поток в критической секции. `wait()`/`notify()` — кооперация потоков через монитор. Основа многопоточности в Java до появления `java.util.concurrent`.  
> На интервью: как работает Lock Inflation (thin → fat lock), зачем wait() в while, чем отличается notify от notifyAll.

## Связанные темы
[[Модель памяти Java (JMM) и барьеры памяти]], [[Lock]], [[Процессы и Потоки, Thread, Runnable, состояния потоков]]

---

## synchronized: формы применения

```java
// 1. Метод экземпляра — монитор this:
public synchronized void increment() { count++; }

// 2. Статический метод — монитор Counter.class:
public static synchronized void increment() { count++; }

// 3. Блок — явный объект-монитор (рекомендуется: меньше scope блокировки):
private final Object lock = new Object();
public void update() {
    synchronized (lock) {
        count++;
    }
}
```

**Байт-код:** `monitorenter` / `monitorexit`. Компилятор генерирует два `monitorexit` — для нормального выхода и для выхода по исключению. Это гарантирует unlock даже при exception.

**JMM:** `synchronized` вход = LoadLoad + LoadStore барьеры. Выход = StoreStore + StoreLoad. Happens-before: `unlock` HB перед следующим `lock` того же монитора.

---

## Object Header и Lock Inflation

Каждый объект в JVM содержит Object Header (12-16 байт). Первые 8 байт — **Mark Word** — кодируют состояние монитора:

```
Mark Word (64-bit JVM):

UNLOCKED:    | hashcode (31 bit) | age (4) | 0 | 01 |
THIN LOCK:   | ptr to LockRecord in thread stack (62 bit) | 00 |
HEAVY LOCK:  | ptr to ObjectMonitor in Heap (62 bit)      | 10 |
GC MARK:     | forwarding address                         | 11 |
```

**Lock Inflation — эскалация блокировки (Java 21+, без biased locks):**

```
1. UNLOCKED (01)
   → поток пытается захватить: CAS Mark Word [hashcode|01] → [ptr LockRecord|00]
   → CAS успешен → THIN LOCK (в стеке потока)
   → CAS провалился (гонка) → inflate → HEAVY LOCK

2. THIN LOCK (00): LockRecord в стеке = {displaced Mark Word + owner thread}
   → другой поток пытается захватить → adaptive spin (несколько итераций)
   → spin не помог → inflate → HEAVY LOCK

3. HEAVY LOCK (10): ObjectMonitor в Heap
   → ObjectMonitor: {owner, EntryList, WaitSet, count}
   → конкурирующие потоки → EntryList → Thread.State.BLOCKED
   → wait() → перемещает поток в WaitSet → Thread.State.WAITING
   → notify() → перемещает из WaitSet → EntryList

4. DEFLATION (Java 18+): если ObjectMonitor пуст и нет waiters →
   JVM асинхронно возвращает в unlocked состояние
```

> [!WARNING] `hashCode()` ломает thin lock
> `System.identityHashCode(obj)` записывает hash в Mark Word. После этого в Mark Word нет места для LockRecord → thin lock невозможен → сразу inflate в heavy lock! Не вызывай `identityHashCode` на объекте, который используешь как монитор.

**Biased locking удалён в Java 21** (JEP 374 deprecated в Java 15, удалён в Java 21). Требовал stop-the-world revocation при первом обращении другого потока — в продакшне с пулами потоков давал больше STW пауз, чем выигрыша.

Диагностика: `jstack <pid>`, флаг `-Xlog:monitorinflation*=debug`.

---

## wait() / notify() / notifyAll()

Механизм кооперации потоков. **Только внутри `synchronized` блока** — иначе `IllegalMonitorStateException`.

```java
// Семантика:
synchronized (obj) {
    while (!condition) {    // WHILE, не IF (защита от spurious wakeup)
        obj.wait();         // 1. освобождает монитор
                            // 2. поток → WaitSet → Thread.State.WAITING
                            // 3. при notify: WaitSet → EntryList, ждёт монитор
                            // 4. после получения монитора: продолжает с этой строки
    }
    doWork(); // condition выполнено
}

synchronized (obj) {
    condition = true;
    obj.notify();    // будит ОДИН случайный поток из WaitSet
    obj.notifyAll(); // будит ВСЕ потоки из WaitSet
}
```

**Spurious wakeup** — поток может проснуться из `wait()` без вызова `notify()`. Это нормальное поведение JVM/ОС. Поэтому `wait()` всегда в `while (!condition)`.

**notify vs notifyAll:**

| | `notify()` | `notifyAll()` |
|---|---|---|
| Будит | 1 случайный поток | все потоки из WaitSet |
| Производительность | Лучше | Хуже при большом WaitSet |
| Риск | Starvation если разные условия | Нет |

**Правило:** если все ожидающие потоки могут продолжить — `notifyAll()`. Если только один — `notify()` (но убедись что все ждут одного и того же условия).

**Producer-Consumer:**
```java
class Buffer {
    private final Queue<Integer> queue = new ArrayDeque<>();
    private final int capacity;
    private final Object lock = new Object();

    public void put(int item) throws InterruptedException {
        synchronized (lock) {
            while (queue.size() == capacity) lock.wait(); // ждём места
            queue.add(item);
            lock.notifyAll(); // сигналим consumer'ам
        }
    }

    public int take() throws InterruptedException {
        synchronized (lock) {
            while (queue.isEmpty()) lock.wait(); // ждём элемента
            int item = queue.poll();
            lock.notifyAll(); // сигналим producer'ам
            return item;
        }
    }
}
```

**wait(timeout):**
```java
synchronized (lock) {
    long deadline = System.currentTimeMillis() + 5000;
    while (!condition) {
        long remaining = deadline - System.currentTimeMillis();
        if (remaining <= 0) throw new TimeoutException();
        lock.wait(remaining); // с таймаутом, но в while!
    }
}
```

---

## synchronized vs Lock

| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Автоосвобождение при exception | ✅ | ❌ (нужен finally) |
| `tryLock()` | ❌ | ✅ |
| Прерывание ожидания | ❌ | ✅ |
| Несколько Condition | ❌ | ✅ |
| Fair-режим | ❌ | ✅ |
| Производительность | Оптимизирована JIT (thin lock) | Сравнима при contention |

**Когда `synchronized` предпочтительнее:** простые критические секции, нет нужды в tryLock/interrupt, код должен быть читаемым.

---

## Deadlock

```
Необходимые условия (Coffman):
1. Взаимное исключение — ресурс у одного потока
2. Удержание и ожидание — держу один, жду другой
3. Нет принудительного освобождения — отобрать нельзя
4. Круговое ожидание — A ждёт B, B ждёт A
```

**Профилактика:** всегда захватывай мониторы в одном порядке по всему коду.

**Диагностика:** `jstack <pid>` → `Found one Java-level deadlock` + цепочка `waiting to lock`.

```java
// Программное обнаружение:
ThreadMXBean tmx = ManagementFactory.getThreadMXBean();
long[] deadlocked = tmx.findDeadlockedThreads();
if (deadlocked != null) {
    ThreadInfo[] infos = tmx.getThreadInfo(deadlocked, true, true);
    // логируем/алертируем
}
```

---

## Вопросы на интервью

- Объясни Lock Inflation: что такое thin lock, heavy lock, когда и зачем эскалация?
- Почему `wait()` всегда в `while`, не в `if`?
- Что такое spurious wakeup? Где это учитывается в JVM?
- Чем `notify()` отличается от `notifyAll()`? Когда что использовать?
- Что произойдёт если вызвать `wait()` вне `synchronized` блока?
- Как hashCode влияет на thin lock?
- Почему biased locking убрали в Java 21?
- Какие happens-before гарантии даёт `synchronized`?

## Подводные камни

- **`wait()` в `if`** — spurious wakeup пропустит проверку условия.
- **`notify()` при нескольких разных условиях** — разбудит случайный поток который, возможно, ждёт другого условия → starvation. Используй `notifyAll()` или `Condition` из `Lock`.
- **`synchronized(this)`** — если `this` доступен снаружи, внешний код может захватить тот же монитор → непредсказуемые блокировки. Используй приватный `Object lock = new Object()`.
- **`identityHashCode` на monitor-объекте** — убивает thin lock, принудительная инфляция.
- **`synchronized` в virtual threads** — pinning: virtual thread прикреплён к carrier thread на всё время удержания монитора. Под нагрузкой это блокирует carrier threads. Используй `ReentrantLock`.
