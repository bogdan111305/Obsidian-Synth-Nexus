# ThreadPool, Future, Callable, Executors, CompletableFuture

> Пул потоков — переиспользование потоков вместо создания нового на каждую задачу. `Future` — результат асинхронной задачи. `CompletableFuture` — неблокирующие цепочки операций.  
> На интервью: как работает ThreadPoolExecutor (логика заполнения!), отличие thenApply от thenCompose, deadlock в ForkJoinPool, когда виртуальные потоки, когда пул.

## Связанные темы
[[Synchronizers]], [[Lock]], [[Процессы и Потоки, Thread, Runnable, состояния потоков]], [[Virtual Threads — модель и архитектура]]

---

## Runnable vs Callable

```java
// Runnable: нет результата, нет checked exceptions
Runnable task = () -> processData();
executor.execute(task);

// Callable: есть результат, есть checked exceptions
Callable<String> task = () -> {
    String result = fetchFromDB(); // может бросить Exception
    return result;
};
Future<String> future = executor.submit(task);
```

---

## ThreadPoolExecutor: конструктор и логика заполнения

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                               // corePoolSize — всегда живые потоки
    8,                               // maximumPoolSize — максимум при нагрузке
    60, TimeUnit.SECONDS,            // keepAliveTime — TTL для core+ потоков
    new ArrayBlockingQueue<>(200),   // workQueue — bounded!
    threadFactory,                   // фабрика именованных потоков
    new ThreadPoolExecutor.CallerRunsPolicy() // отклонение задач
);
```

**Контринтуитивная логика `submit(task)`:**
```
1. threads < corePoolSize
   → создать новый поток, выполнить task
   (даже если в очереди уже есть задачи!)

2. threads >= corePoolSize
   → попробовать добавить в workQueue
   → queue.offer(task) == true → задача в очереди

3. queue.offer(task) == false (очередь полна!)
   → threads < maximumPoolSize → создать extra поток
   → threads >= maximumPoolSize → RejectedExecutionHandler!
```

**RejectedExecutionHandler политики:**

| Политика | Поведение |
|---|---|
| `AbortPolicy` (default) | `RejectedExecutionException` |
| `CallerRunsPolicy` | Выполнить в вызывающем потоке (backpressure) |
| `DiscardPolicy` | Молча выбросить задачу |
| `DiscardOldestPolicy` | Выбросить старейшую задачу из очереди |

**Почему `Executors.*` опасны:**
```java
Executors.newFixedThreadPool(4);
// = new ThreadPoolExecutor(4, 4, 0, SECONDS, new LinkedBlockingQueue<>())
// LinkedBlockingQueue() = НЕОГРАНИЧЕННАЯ очередь (Integer.MAX_VALUE)!
// При нагрузке → очередь растёт → OOM

Executors.newCachedThreadPool();
// = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, SECONDS, new SynchronousQueue<>())
// При spike нагрузки → неограниченное число потоков → OOM / перегрузка
```

**Правильный подход — явный конструктор:**
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 8, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),
    r -> {
        Thread t = new Thread(r, "worker-" + counter.incrementAndGet());
        t.setDaemon(true); // не блокируем JVM shutdown
        return t;
    },
    new ThreadPoolExecutor.CallerRunsPolicy()
);

// Мониторинг:
executor.getPoolSize();           // текущий размер пула
executor.getActiveCount();        // активные потоки
executor.getQueue().size();       // задачи в очереди
executor.getCompletedTaskCount(); // завершённые задачи
```

---

## Future

```java
Future<String> future = executor.submit(() -> heavyComputation());

future.isDone();         // завершена ли задача
future.isCancelled();    // отменена ли
future.cancel(true);     // отмена (true = прервать поток если выполняется)
String result = future.get();             // блокирующее получение результата
String result = future.get(5, SECONDS);  // с таймаутом → TimeoutException
```

`get()` оборачивает исключения из задачи в `ExecutionException`:
```java
try {
    String result = future.get();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // восстановить флаг
} catch (ExecutionException e) {
    Throwable cause = e.getCause();     // оригинальное исключение из Callable
    handleError(cause);
} catch (TimeoutException e) {
    future.cancel(true);
}
```

---

## ExecutorService: массовые операции

```java
List<Callable<String>> tasks = List.of(
    () -> fetchA(), () -> fetchB(), () -> fetchC()
);

// invokeAll — ждёт ВСЕ задачи, возвращает List<Future>
List<Future<String>> results = executor.invokeAll(tasks);
List<Future<String>> results = executor.invokeAll(tasks, 5, SECONDS); // с таймаутом

// invokeAny — возвращает результат ПЕРВОЙ успешной задачи, отменяет остальные
String first = executor.invokeAny(tasks);

// Shutdown:
executor.shutdown();                      // не принимает новые задачи, ждёт текущие
executor.awaitTermination(60, SECONDS);   // ждём завершения
executor.shutdownNow();                   // прерывает текущие, возвращает очередь
```

---

## ScheduledExecutorService

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// Однократно с задержкой:
scheduler.schedule(() -> doWork(), 5, SECONDS);

// scheduleAtFixedRate — интервал ОТ НАЧАЛА предыдущей задачи:
scheduler.scheduleAtFixedRate(() -> heartbeat(), 0, 1, SECONDS);
// Если задача выполняется 3 секунды при периоде 1с → нет наложений, но drift!

// scheduleWithFixedDelay — интервал ПОСЛЕ ЗАВЕРШЕНИЯ предыдущей:
scheduler.scheduleWithFixedDelay(() -> pollDB(), 0, 1, SECONDS);
// Задержка = время задачи + delay → равномерная пауза между запусками
```

---

## CompletableFuture: паттерны и ловушки

```java
// Создание:
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> fetch(), executor);
CompletableFuture<Void>   cf = CompletableFuture.runAsync(() -> doWork(), executor);

// thenApply (sync) vs thenApplyAsync (async):
cf.thenApply(s -> s.toUpperCase())          // sync: в потоке, завершившем cf
  .thenApplyAsync(s -> enrich(s), cpuPool)  // async: в cpuPool
  .thenApply(s -> format(s));               // sync: в cpuPool потоке

// thenCompose = flatMap (когда следующий шаг тоже асинхронный):
CompletableFuture<User> getUser(long id) { ... }
CompletableFuture<Order> getOrder(User user) { ... }

// НЕПРАВИЛЬНО: thenApply даёт CompletableFuture<CompletableFuture<Order>>
// ПРАВИЛЬНО:
CompletableFuture<Order> order = getUser(userId)
    .thenCompose(user -> getOrder(user));

// Обработка ошибок:
cf.exceptionally(ex -> "default")     // recover: только при ошибке
  .handle((result, ex) -> {           // всегда: может изменить результат
      return ex != null ? fallback() : result;
  })
  .whenComplete((result, ex) -> {     // всегда: не может изменить результат
      metrics.record(ex == null ? "ok" : "error");
  });

// Комбинирование:
CompletableFuture.allOf(f1, f2, f3)  // ждать все (тип Void)
    .thenRun(() -> {
        String r1 = f1.join(); // безопасно после allOf
        String r2 = f2.join();
    });

CompletableFuture.anyOf(f1, f2, f3); // первый завершённый (Object)

// Timeout (Java 9+):
cf.orTimeout(5, SECONDS);                    // TimeoutException при превышении
cf.completeOnTimeout("default", 5, SECONDS); // default value при превышении
```

**ЛОВУШКА: `join()` в потоке ForkJoinPool → deadlock:**
```java
// BAD: join() блокирует FJP worker thread, который нужен для внутренней задачи
CompletableFuture.supplyAsync(() -> {
    return CompletableFuture.supplyAsync(() -> "inner").join(); // DEADLOCK!
}, ForkJoinPool.commonPool());

// GOOD: thenCompose
CompletableFuture.supplyAsync(() -> "outer")
    .thenCompose(outer -> CompletableFuture.supplyAsync(() -> outer + "inner"));

// GOOD: использовать отдельный executor, не FJP
CompletableFuture.supplyAsync(() -> {
    return CompletableFuture.supplyAsync(() -> "inner", myExecutor).join();
}, myExecutor2); // разные executor — нет deadlock
```

**ЛОВУШКА: без explicit executor → ForkJoinPool.commonPool():**
```java
// BAD для I/O: блокирует FJP workers
cf.thenApplyAsync(data -> httpClient.send(data)); // FJP!

// GOOD: явный executor для I/O
cf.thenApplyAsync(data -> httpClient.send(data), ioExecutor);
```

---

## ForkJoinPool

Оптимизирован для divide-and-conquer через work-stealing:

```java
ForkJoinPool pool = new ForkJoinPool(nThreads);
// ForkJoinPool.commonPool() — глобальный пул (используется parallel streams, CompletableFuture)

pool.invoke(new RecursiveTask<Long>() {
    @Override
    protected Long compute() {
        if (data.length <= THRESHOLD) return sequentialSum(data);
        // Fork:
        var left  = new SumTask(leftHalf).fork();
        var right = new SumTask(rightHalf);
        return right.compute() + left.join(); // compute right inline, join left
    }
});
```

Work-stealing: idle потоки крадут задачи из хвоста очереди других потоков → эффективно при неравномерном разбиении.

---

## ThreadPool + Virtual Threads (Java 21+)

```java
// I/O-bound → virtual threads (один поток на задачу, нет смысла в пуле)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Response>> futures = requests.stream()
        .map(req -> executor.submit(() -> httpClient.send(req)))
        .toList();
    // при I/O virtual thread unmounts — не занимает carrier thread
}

// CPU-bound → platform thread pool = N ядер
ExecutorService cpuPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

// Mixed (I/O + CPU) → раздельные пулы:
ExecutorService ioPool  = Executors.newVirtualThreadPerTaskExecutor();
ExecutorService cpuPool = Executors.newFixedThreadPool(nCores);

CompletableFuture.supplyAsync(() -> fetchData(url), ioPool)  // virtual thread
    .thenApplyAsync(data -> process(data), cpuPool);          // platform thread
```

---

## Вопросы на интервью

- Объясни логику заполнения ThreadPoolExecutor: когда создаются extra потоки?
- Почему `Executors.newFixedThreadPool()` опасен в продакшне?
- Чем `thenApply` отличается от `thenApplyAsync`? Какой executor используется по умолчанию?
- Чем `thenCompose` отличается от `thenApply`? Покажи пример.
- Как обрабатывать исключения в CompletableFuture? Чем отличаются `handle`, `exceptionally`, `whenComplete`?
- Почему `join()` внутри `supplyAsync` с ForkJoinPool может вызвать deadlock?
- Когда виртуальные потоки лучше пула? Когда хуже?
- Разница между `scheduleAtFixedRate` и `scheduleWithFixedDelay`?

## Подводные камни

- **`Executors.newFixedThreadPool`** — `LinkedBlockingQueue` без лимита → OOM при нагрузке.
- **`Executors.newCachedThreadPool`** — неограниченные потоки → OOM/перегрузка при spike.
- **`future.get()` без таймаута** — если задача зависла → поток ждёт вечно.
- **Исключения в Runnable/Callable** — без `get()` или `whenComplete` проглатываются молча.
- **`thenApplyAsync` без executor** — ForkJoinPool.commonPool(). Для I/O — катастрофа.
- **`join()` в FJP-потоке** — deadlock при вложенных CompletableFuture на одном пуле.
- **Virtual threads + synchronized** → pinning на carrier thread → теряем преимущества. Используй `ReentrantLock`.
