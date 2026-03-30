# Virtual Threads — модель и архитектура

> Virtual Thread (Java 21, JEP 444) — лёгкий поток, управляемый JVM, а не ОС. Реализован через **continuation** (сохранение стека вызовов в heap) + **ForkJoinPool scheduler**. Позволяет создавать миллионы потоков без накладных расходов платформенных потоков.

## Связанные темы
[[Процессы и Потоки, Thread, Runnable, состояния потоков]], [[Virtual Threads vs Platform Threads]], [[Carrier Threads и Pinning]], [[Happens-Before в контексте Virtual Threads]], [[Structured Concurrency]], [[Scoped Values (Java 21, JEP 446)]]

---

## Архитектура

```
Virtual Thread (VT)          Platform Thread (PT)
   ↓ управляется JVM             ↓ управляется ОС
   ↓ стек в Heap                 ↓ стек в OS memory (~1MB)
   ↓ миллионы экземпляров        ↓ тысячи максимум

VT монтируется на PT (carrier thread) →
    carrier выполняет байткод VT →
    при блокирующей операции VT демонтируется →
    carrier освобождается для другого VT
```

**Ключевые компоненты:**

| Компонент | Роль |
|---|---|
| **Continuation** | Объект в heap, хранящий стек вызовов VT. Создаётся при park/unpark |
| **Carrier Thread** | Платформенный поток (из ForkJoinPool), исполняющий VT |
| **Scheduler** | `ForkJoinPool` в work-stealing режиме. По умолчанию parallelism = CPU count |
| **Mount/Unmount** | Монтирование VT на carrier при возобновлении, демонтирование при блокировке |

---

## Жизненный цикл

```
NEW → RUNNABLE → RUNNING (на carrier) →
    ┌──────────────────────────────────┐
    │ Блокирующий вызов (I/O, sleep)   │
    │   ↓                              │
    │ VT демонтируется (unmount)       │
    │ carrier освобождается            │
    │ continuation сохраняется в heap  │
    │   ↓                              │
    │ I/O завершается                  │
    │ VT встаёт в очередь scheduler    │
    │   ↓                              │
    │ VT монтируется на (возможно)     │
    │ другой carrier                   │
    └──────────────────────────────────┘
→ TERMINATED
```

---

## Создание Virtual Threads

```java
// 1. Thread.ofVirtual() — явное создание
Thread vt = Thread.ofVirtual()
    .name("my-vt")
    .start(() -> System.out.println("Hello from VT"));

// 2. Thread.startVirtualThread() — shortcut
Thread.startVirtualThread(() -> System.out.println("VT"));

// 3. ExecutorService с virtual threads (рекомендуемый в production):
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofMillis(100)); // блокирует VT, не carrier!
            return "done";
        });
    }
} // автоматический shutdown при выходе из try-with-resources
```

---

## Continuation — внутреннее устройство

**Continuation** — объект в heap, представляющий "замороженный" стек выполнения:

```java
// Псевдо-реализация (упрощённо):
class VirtualThread extends Thread {
    private Continuation continuation; // стек вызовов в heap

    void park() {
        // Сохраняем текущий стек выполнения в continuation
        continuation.yield(); // возврат управления carrier thread
        // carrier выполняет другой VT
    }

    void unpark() {
        // Ставим VT в очередь scheduler для возобновления
        scheduler.submit(() -> continuation.run());
    }
}
```

**Размер стека continuation:**
- Начальный: ~1KB (против 512KB–1MB у платформенного)
- Растёт по мере глубины стека вызовов
- Хранится в heap → GC управляет памятью

---

## Scheduler — ForkJoinPool

По умолчанию: `ForkJoinPool.commonPool()` с `parallelism = Runtime.getRuntime().availableProcessors()`

```java
// Настройка:
System.setProperty("jdk.virtualThreadScheduler.parallelism", "8");
System.setProperty("jdk.virtualThreadScheduler.maxPoolSize", "256");
System.setProperty("jdk.virtualThreadScheduler.minRunnable", "1");

// Кастомный scheduler (редко нужен):
var scheduler = Executors.newFixedThreadPool(4);
Thread vt = Thread.ofVirtual()
    .scheduler(scheduler)  // не публичный API, только через reflection
    .start(task);
```

**Work-stealing:** carrier threads крадут задачи из очередей других carrier-ов → равномерная нагрузка.

---

## Мониторинг Virtual Threads

```bash
# JFR (Java Flight Recorder):
jcmd <pid> JFR.start
jcmd <pid> JFR.dump filename=vt.jfr
# Посмотреть через JMC: события VirtualThreadPinned, VirtualThreadSubmitFailed

# Thread dump (показывает VT):
jcmd <pid> Thread.dump_to_file -format=json vt-dump.json

# JVM флаги для диагностики:
-Djdk.tracePinnedThreads=full    # логировать pinning с stack trace
-Djdk.tracePinnedThreads=short   # только имена методов
```

---

## Вопросы на интервью

- Что такое Virtual Thread? Чем он отличается от платформенного?
- Что такое continuation? Где хранится стек виртуального потока?
- Что такое carrier thread? Какой пул потоков используется по умолчанию?
- Что происходит с carrier при блокирующей операции VT?
- Сколько памяти занимает один Virtual Thread?
- Как создать Virtual Thread? Какой способ предпочтителен в production?
- Почему VT не подходят для CPU-intensive задач?

## Подводные камни

- **CPU-bound задачи** — VT не ускоряют вычисления: если 4 carrier thread и 1M VT выполняют CPU-intensive код, работают только 4 параллельно. VT помогает только при I/O-bound нагрузке.
- **ThreadLocal в VT** — работает, но дорого: каждый из 1M VT создаёт свой ThreadLocal. При большом количестве VT это memory overhead. Предпочитай [[Scoped Values (Java 21, JEP 446)]].
- **`synchronized` блокирует carrier** — пока VT в `synchronized` блоке, carrier не освобождается → pinning. Подробнее: [[Carrier Threads и Pinning]].
- **Нельзя задать приоритет** — `Thread.setPriority()` для VT игнорируется.
- **daemon-статус по умолчанию** — VT всегда daemon thread. JVM не будет ждать их завершения при остановке. Используй `structured concurrency` или явное ожидание.
