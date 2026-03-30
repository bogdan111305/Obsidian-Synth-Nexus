# Процессы и Потоки в Java

> **Поток** — единица выполнения внутри JVM-процесса. Platform threads — нативные OS потоки (~1-2MB stack). **Virtual threads** (Java 21) — легковесные (~KB), миллионы одновременно, маппятся на OS threads через continuation механизм.
> На интервью: состояния потоков, разница BLOCKED vs WAITING, virtual thread internals, pinning и как его избежать.

## Связанные темы
[[Java Monitor]], [[Lock]], [[Прерывание потока в Java]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]], [[Structured Concurrency (Java 21)]]

---

## Создание потоков

```java
// 1. extends Thread (не рекомендуется — смешивает задачу и механизм)
new Thread(() -> doWork()).start();

// 2. Runnable / лямбда (предпочтительно для platform threads)
Runnable task = () -> doWork();
new Thread(task).start();

// 3. ExecutorService (производственный код)
ExecutorService exec = Executors.newFixedThreadPool(4);
exec.submit(() -> doWork());

// 4. Virtual thread (Java 21+)
Thread.ofVirtual().start(() -> doWork());
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    exec.submit(() -> handleRequest(req)); // один поток на задачу
}
```

---

## Состояния потока (Thread.State)

```
NEW → RUNNABLE ⇄ BLOCKED       (ждёт монитор synchronized)
              ⇄ WAITING        (wait(), join(), park() — без таймаута)
              ⇄ TIMED_WAITING  (sleep(ms), wait(ms), join(ms))
    → TERMINATED
```

| Состояние | Причина | Выход |
|-----------|---------|-------|
| **NEW** | `new Thread()`, `start()` не вызван | `start()` |
| **RUNNABLE** | выполняется или ждёт CPU | OS-планировщик |
| **BLOCKED** | ждёт монитор (`synchronized`) | другой поток вышел из synchronized |
| **WAITING** | `wait()`, `join()`, `park()` | `notify()`, thread завершился, `unpark()` |
| **TIMED_WAITING** | `sleep(ms)`, `wait(ms)`, `join(ms)` | таймаут или notify |
| **TERMINATED** | `run()` вернул / выбросил | нельзя перезапустить |

**Критичное отличие BLOCKED vs WAITING:**
- **BLOCKED** — ожидает войти в `synchronized` блок. `interrupt()` НЕ прерывает.
- **WAITING/TIMED_WAITING** — ожидает сигнала. `interrupt()` → `InterruptedException`.

```java
// Диагностика состояний:
Thread t = ...;
t.getState(); // Thread.State enum

// В jstack:
// "Thread-0" BLOCKED on <0x...> (a java.lang.Object)
// "Thread-1" WAITING on <0x...> (object.wait())
```

---

## Методы управления

```java
thread.start();                  // запустить поток (NEW → RUNNABLE)
thread.join();                   // ждать завершения → WAITING
thread.join(5000);               // с таймаутом → TIMED_WAITING
Thread.sleep(1000);              // пауза → TIMED_WAITING
thread.interrupt();              // установить флаг прерывания
thread.isAlive();                // не в NEW и не TERMINATED?
thread.setDaemon(true);         // daemon-поток: JVM не ждёт его завершения
thread.setPriority(1..10);      // подсказка планировщику (платформозависимо)

Thread.currentThread();          // текущий поток
Thread.currentThread().isVirtual(); // Java 21+: виртуальный?
```

**Daemon потоки:** JVM завершается когда все non-daemon потоки закончились. GC, JIT — daemon. Сервисные потоки приложения — обычно non-daemon.

---

## Virtual Threads: внутреннее устройство

> [!INFO] Virtual Thread = Continuation + ForkJoinPool Scheduler + Carrier Thread

```
Platform Thread:
  JVM Thread ──── OS Thread (kernel) ──── CPU Core
  Stack: ~1-2 MB (нативная память ОС)
  Создание: ~1 ms, Context switch: ~1-10 µs (kernel syscall)
  Максимум: thousands

Virtual Thread:
  VirtualThread ──── Continuation (Heap) ──── Carrier (FJP Worker)
  Stack: ~KB (Java Heap, serialized continuation)
  Создание: ~1-10 µs, "Switch": ~100-300 ns (JVM, no syscall)
  Максимум: millions
```

**Жизненный цикл при блокировке:**
```
1. Virtual thread вызывает blocking I/O / sleep / LockSupport.park()
2. VirtualThread.yield() — стек сериализуется в Heap (continuation)
3. Carrier thread освобождается → берёт другой virtual thread
4. I/O завершён → continuation восстанавливается на (любом) carrier
```

```java
// Scheduler (ForkJoinPool): parallelism = availableProcessors() по умолчанию
// Проверить:
Thread.currentThread().isVirtual();

// Кастомный scheduler (редко нужен):
ExecutorService scheduler = Executors.newFixedThreadPool(4);
Thread.ofVirtual().scheduler(scheduler).start(task);
```

---

## Pinning — главная ловушка Virtual Threads

Виртуальный поток **прикрепляется (pinned)** к carrier и не может размонтироваться при:
1. **`synchronized` блоке** — carrier блокируется вместе с virtual thread
2. **Native method / JNI** — JVM не может демонтировать continuation

```java
// ПЛОХО: carrier thread заблокирован на время всей synchronized операции
class PinnedService {
    synchronized void slowDbQuery() { // PINNING!
        // 200ms запрос к БД — carrier thread занят, не может взять другие virtual threads
    }
}

// ХОРОШО: ReentrantLock использует LockSupport.park() → virtual thread unmounts!
class UnpinnedService {
    private final ReentrantLock lock = new ReentrantLock();

    void slowDbQuery() {
        lock.lock();
        try {
            // 200ms запрос к БД — virtual thread yields при park(), carrier свободен
        } finally {
            lock.unlock();
        }
    }
}

// Диагностика:
// JVM флаг:  -Djdk.tracePinnedThreads=full
// JFR Event: jdk.VirtualThreadPinned
// jcmd <pid> JFR.start duration=30s filename=pinning.jfr
```

> [!INFO] Java 24 (JEP 491): `synchronized` больше не вызывает pinning в большинстве случаев. Pinning остаётся только для JNI.

---

## Platform Thread vs Virtual Thread

| | Platform Thread | Virtual Thread |
|---|---|---|
| Управление | ОС (kernel) | JVM (user-space) |
| Stack Memory | ~1-2 MB нативной памяти | ~KB Java Heap |
| Создание | ~1 ms | ~1-10 µs |
| Context switch | ~1-10 µs (syscall) | ~100-300 ns |
| Максимальное кол-во | ~thousands | millions |
| CPU-bound задачи | ✅ | ❌ нет преимущества |
| I/O-bound задачи | ❌ блокирует OS thread | ✅ unmounts при блокировке |
| ThreadLocal | ✅ | ⚠️ лучше ScopedValue |
| `synchronized` | ✅ | ⚠️ pinning (до Java 24) |

**Когда NOT использовать Virtual Threads:**
```java
// CPU-intensive — нет I/O, нет yield, нет выгоды:
// BAD:
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    exec.submit(() -> heavyMatrixComputation()); // просто занимает carrier
}
// GOOD:
ForkJoinPool.commonPool().submit(() -> heavyMatrixComputation());

// Старые JDBC drivers/connection pools — могут использовать synchronized → pinning
// ThreadLocal с большими объектами — millions виртуальных потоков × ThreadLocal = OOM
// Решение: ScopedValue (Java 21+) вместо ThreadLocal
```

---

## Вопросы на интервью

- Назови все состояния потока и переходы между ними.
- Чем BLOCKED отличается от WAITING? Можно ли прервать поток в BLOCKED?
- Что такое daemon-поток? Когда JVM завершается?
- Что такое virtual thread? Как работает continuation?
- Что такое pinning? Почему `synchronized` вызывает pinning? Как исправить?
- Почему `Thread.sleep()` в virtual thread не блокирует carrier?
- Когда virtual threads не дают преимущества?
- Чем `Thread.ofVirtual()` отличается от `Executors.newVirtualThreadPerTaskExecutor()`?

---

## Подводные камни

- **`thread.run()` вместо `thread.start()`** — выполняется в текущем потоке, нового не создаётся. Компилятор не предупредит.
- **BLOCKED при interrupt** — `interrupt()` не прерывает поток в `synchronized` ожидании. Только WAITING/TIMED_WAITING реагируют. Для прерываемого ожидания блокировки: `lock.lockInterruptibly()`.
- **`Thread.stop()` удалён** — принудительно завершает поток, не освобождая мониторы → inconsistent state. Используй interrupt + кооперативный флаг.
- **Pinning + virtual threads** — `synchronized` на virtual thread = carrier thread заблокирован. Под нагрузкой все carrier threads могут быть pinned → деградация до platform thread поведения. Используй `ReentrantLock`.
- **ThreadLocal в virtual threads** — при миллионах потоков каждый копирует ThreadLocal-значение → высокое потребление памяти. Мигрируй на `ScopedValue`.
- **Daemon thread + результаты** — JVM не ждёт daemon потоков при выходе. Незавершённые I/O операции, незафлюшенные буферы могут быть потеряны.
- **`setPriority()` ненадёжен** — платформозависимо. На Linux с CFS-планировщиком приоритеты игнорируются почти полностью.
