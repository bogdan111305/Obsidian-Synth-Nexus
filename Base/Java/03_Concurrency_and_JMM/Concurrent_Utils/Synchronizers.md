# Synchronizers: CountDownLatch, CyclicBarrier, Phaser, Semaphore

> Координационные примитивы из `java.util.concurrent` для управления порядком выполнения потоков.  
> На интервью: чем отличаются друг от друга, когда что выбирать, подводные камни каждого.

## Связанные темы
[[Lock]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]], [[Java Monitor]]

---

## CountDownLatch

**Одноразовый** счётчик: N потоков выполняют `countDown()`, другие ждут через `await()` пока счётчик не достигнет 0.

```java
CountDownLatch latch = new CountDownLatch(3);

// Рабочие потоки:
executor.submit(() -> { doWork(); latch.countDown(); });
executor.submit(() -> { doWork(); latch.countDown(); });
executor.submit(() -> { doWork(); latch.countDown(); });

// Ожидающий поток:
latch.await();                           // блокирует до count=0
latch.await(5, TimeUnit.SECONDS);        // с таймаутом → boolean
```

**Паттерн "По сигналу марш!" (параллельный старт):**
```java
CountDownLatch startSignal = new CountDownLatch(1); // сигнал старта
CountDownLatch done = new CountDownLatch(N);        // ожидание завершения

for (int i = 0; i < N; i++) {
    new Thread(() -> {
        try {
            startSignal.await();   // все ждут стартового сигнала
            doWork();
            done.countDown();
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }).start();
}

prepareEverything();
startSignal.countDown(); // старт!
done.await();            // ждём всех
```

**Внутри:** AQS shared state. `countDown()` = `releaseShared(1)`. `await()` = `acquireSharedInterruptibly` — поток паркуется пока state > 0. При state=0 все ожидающие пробуждаются одновременно через `doReleaseShared()`.

**Ограничение:** одноразовый — сбросить нельзя, только создать новый.

---

## CyclicBarrier

**Циклический** барьер: N потоков ждут друг друга, когда все достигли — все разблокируются, барьер сбрасывается для повторного использования.

```java
// Опциональный barrierAction выполняется одним из потоков ПЕРЕД разблокировкой всех:
CyclicBarrier barrier = new CyclicBarrier(3, () -> mergeResults());

// Каждый поток:
processChunk(myPart);
barrier.await();     // ждём всех → возвращает arrival index (0 = последний)
processNextPhase();  // все потоки стартуют одновременно
barrier.await();     // второй цикл — тот же барьер!
```

**Внутри:** реализован через `ReentrantLock` + `Condition` (не AQS shared state как CountDownLatch). Счётчик `count` уменьшается при каждом `await()`. Последний поток выполняет `barrierAction`, вызывает `trip.signalAll()`, создаёт новое `generation`.

**Сломанный барьер (`BrokenBarrierException`):**
```java
// Барьер ломается если любой из ожидающих потоков:
// 1. Прерван (InterruptedException)
// 2. Превысил таймаут (TimeoutException)
// Все остальные получают BrokenBarrierException

try {
    barrier.await(2, TimeUnit.SECONDS);
} catch (BrokenBarrierException e) {
    // кто-то другой сломал барьер
    if (barrier.isBroken()) barrier.reset(); // сброс (ломает текущий цикл!)
} catch (TimeoutException e) {
    // этот поток превысил таймаут — барьер сломан для всех
}
```

---

## Phaser

**Гибкий** многофазный синхронизатор: CyclicBarrier + CountDownLatch + динамическое число участников.

```java
// Основные методы:
phaser.register();              // добавить участника (можно в рантайме)
phaser.arriveAndAwaitAdvance(); // прибыть и ждать остальных → переход к след. фазе
phaser.arriveAndDeregister();   // прибыть и выйти из Phaser
phaser.arrive();                // прибыть без ожидания (асинхронно)
phaser.awaitAdvance(phase);     // ждать завершения конкретной фазы
phaser.getPhase();              // номер текущей фазы
phaser.isTerminated();          // завершён ли
```

**Пример: многофазная обработка с выходом:**
```java
Phaser phaser = new Phaser(3) {
    @Override
    protected boolean onAdvance(int phase, int registeredParties) {
        System.out.println("Фаза " + phase + " завершена");
        return phase >= 2 || registeredParties == 0; // true = завершить Phaser
    }
};

// Поток с 3 фазами:
for (int phase = 0; phase < 3 && !phaser.isTerminated(); phase++) {
    doWork(phase);
    phaser.arriveAndAwaitAdvance();
}
phaser.arriveAndDeregister(); // выходим, освобождаем слот
```

**Внутри:** `state` = `volatile long` кодирует фазу + число зарегистрированных + число прибывших. CAS для атомарного обновления. Дерево Phaser'ов для масштабирования (root Phaser не блокируется напрямую — делегирует sub-phaser'ам).

**Динамическое добавление участников:**
```java
Phaser phaser = new Phaser(1); // 1 участник — сам координатор

for (Task task : tasks) {
    phaser.register(); // +1 участник перед стартом задачи
    executor.submit(() -> {
        try { task.run(); }
        finally { phaser.arriveAndDeregister(); } // -1 при завершении
    });
}
phaser.arriveAndDeregister(); // координатор тоже выходит → Phaser завершится
```

---

## Semaphore

**Счётчик разрешений:** `acquire()` уменьшает (блокирует если 0), `release()` увеличивает. Ограничивает параллельный доступ к ресурсу.

```java
Semaphore sem = new Semaphore(3);       // 3 разрешения
Semaphore fairSem = new Semaphore(3, true); // fair = FIFO очередь

sem.acquire();                     // захватить 1 разрешение (или заблокироваться)
sem.acquire(2);                    // захватить 2
sem.tryAcquire();                  // без блокировки → boolean
sem.tryAcquire(500, MILLISECONDS); // с таймаутом
sem.release();                     // вернуть 1
sem.release(2);                    // вернуть 2
sem.availablePermits();            // текущее число
```

**Пример: пул соединений:**
```java
class ConnectionPool {
    private final Semaphore permits = new Semaphore(MAX_CONNECTIONS);
    private final Queue<Connection> pool = new ConcurrentLinkedQueue<>(createConnections());

    public Connection acquire() throws InterruptedException {
        permits.acquire();          // ждём разрешения
        return pool.poll();         // берём соединение
    }

    public void release(Connection conn) {
        pool.offer(conn);
        permits.release();          // возвращаем разрешение
    }
}
```

**Semaphore(1) ≠ ReentrantLock:**
```java
Semaphore mutex = new Semaphore(1);
mutex.acquire();
// ... другой поток может вызвать release() !
mutex.release(); // ReentrantLock так не позволит — только владелец может unlock()
```

Semaphore не имеет понятия владельца — любой поток может вызвать `release()`. Это позволяет паттерн producer-consumer: producer делает `release()`, consumer делает `acquire()`.

**Внутри:** AQS shared state. `acquire(N)` = `acquireSharedInterruptibly(N)`. `release(N)` = `releaseShared(N)`. Fair vs Unfair — та же механика что у ReentrantLock.

---

## Сравнение

| | `CountDownLatch` | `CyclicBarrier` | `Phaser` | `Semaphore` |
|---|---|---|---|---|
| Повторное использование | ❌ | ✅ | ✅ | ✅ |
| Динамические участники | ❌ | ❌ | ✅ | ✅ |
| Многофазность | ❌ | ✅ (вручную) | ✅ (встроено) | ❌ |
| barrierAction / onAdvance | ❌ | ✅ | ✅ | ❌ |
| Семантика | N-to-M | N-to-N | N-to-N + управление | Resource limit |
| Под капотом | AQS shared | RLock + Condition | AQS + CAS long | AQS shared |

**Когда что выбирать:**
- `CountDownLatch` — одноразовое ожидание инициализации / завершения N задач
- `CyclicBarrier` — фиксированная группа потоков, многоэтапные вычисления (matrix ops, game loop)
- `Phaser` — динамическое число задач, сложная многофазная координация
- `Semaphore` — ограничение параллелизма, пул ресурсов, rate limiting

---

## Вопросы на интервью

- Чем CountDownLatch отличается от CyclicBarrier? Когда что выбрать?
- Что происходит с остальными потоками если один из ожидающих в CyclicBarrier получит InterruptedException?
- Как Phaser позволяет динамически добавлять и удалять участников?
- Чем `Semaphore(1)` отличается от `ReentrantLock`?
- Что такое `onAdvance` в Phaser? Как завершить Phaser?
- Может ли `release()` семафора вызвать поток, который не делал `acquire()`?
- Как реализованы CountDownLatch и Semaphore внутри — в чём разница?

## Подводные камни

- **CountDownLatch — нельзя сбросить** — если countDown() вызван лишний раз (меньше 0 не уйдёт), но если вызван недостаточно — `await()` никогда не вернётся.
- **CyclicBarrier ломается при таймауте/прерывании** — все ожидающие получают `BrokenBarrierException`. После `reset()` текущий цикл сломан — потоки уже в `BrokenBarrierException`.
- **Phaser.arriveAndDeregister() без arrive** — если забыть вызвать при завершении задачи, Phaser никогда не перейдёт к следующей фазе.
- **Semaphore(1) не реентрантен** — поток, уже сделавший `acquire()`, при повторном вызове заблокируется навсегда.
- **Semaphore.release() без acquire()** — увеличивает счётчик выше начального значения. Можно случайно "создать" разрешения.
- **fair=true у Semaphore** — защищает от starvation, но снижает throughput. По умолчанию unfair.
