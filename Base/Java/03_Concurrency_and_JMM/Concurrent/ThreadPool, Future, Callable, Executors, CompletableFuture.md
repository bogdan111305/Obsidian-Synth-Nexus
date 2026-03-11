# ThreadPool, Future, Callable, Executors, CompletableFuture

> [!QUOTE] Суть
> **ThreadPool** — пул переиспользуемых потоков. `Callable<T>` = `Runnable` с результатом и checked exceptions. `Future<T>` = обещание результата (блокирующий `get()`). `CompletableFuture` — неблокирующие цепочки (`.thenApply`, `.thenCompose`, `.exceptionally`). Avoid `Executors.newFixedThreadPool` без мониторинга очереди.

> [!WARNING] Ловушка: newCachedThreadPool под нагрузкой
> `Executors.newCachedThreadPool()` создаёт **неограниченное** число потоков. При spike нагрузки — OOM или тысячи потоков. Предпочитай `ThreadPoolExecutor` с явными `corePoolSize`, `maxPoolSize` и bounded queue.

**Пул потоков** (`Thread Pool`) — это набор заранее созданных потоков, повторно используемых для выполнения задач. Вместо создания нового потока для каждой задачи, задача отправляется в пул, что снижает накладные расходы и улучшает управление ресурсами.

## 1. Преимущества пулов потоков

- **Производительность**: Уменьшение затрат на создание и уничтожение потоков.
- **Контроль параллелизма**: Ограничение числа одновременно выполняемых задач.
- **Управление ресурсами**: Предотвращение перегрузки системы.
- **Гибкость**: Поддержка асинхронных, периодических и рекурсивных задач.

## 2. Интерфейсы для задач: `Runnable` и `Callable`

### 2.1. `Runnable`

- **Интерфейс**: Один метод `void run()`.
- **Особенности**:
    - Не возвращает результат.
    - Не выбрасывает проверяемые исключения.
    - Подходит для простых задач без возвращаемого значения.
- **Пример**:
    
    ```java
    Runnable task = () -> System.out.println("Задача выполняется");
    ```
    

### 2.2. `Callable<V>`

- **Интерфейс**: `@FunctionalInterface` с методом `V call() throws Exception`.
- **Особенности**:
    - Возвращает результат типа `V` (включая `Void`).
    - Поддерживает проверяемые исключения.
    - Используется с `ExecutorService` через `submit()`, возвращает `Future<V>`.
    - Поддерживает лямбда-выражения.
- **Пример**:
    
    ```java
    Callable<String> callableTask = () -> {
        Thread.sleep(1000);
        return "Результат вычисления";
    };
    ```
    

### 2.3. Сравнение `Runnable` и `Callable`

|Характеристика|`Runnable`|`Callable<V>`|
|---|---|---|
|Метод|`void run()`|`V call()`|
|Возвращаемое значение|Нет|Есть (`V`)|
|Проверяемые исключения|Нет|Да|
|Поддержка лямбд|Да|Да|
|Использование|`execute()` или `submit()`|Только `submit()`|

### 2.4. Использование `Callable`

```java
import java.util.concurrent.*;

public class CallableExample {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        Callable<String> callableTask = () -> {
            Thread.sleep(1000);
            return "Результат вычисления";
        };
        Future<String> future = executor.submit(callableTask);
        try {
            String result = future.get();
            System.out.println("Полученный результат: " + result);
        } finally {
            executor.shutdown();
        }
    }
}
```

**Когда использовать `Callable`?**

- Задачи с возвращаемым значением.
- Задачи, требующие обработки исключений.
- Интеграция с `Future` или `CompletableFuture`.

## 3. Интерфейсы `Executor` и `ExecutorService`

### 3.1. `Executor`

- **Описание**: Базовый интерфейс для выполнения задач.
- **Метод**: `void execute(Runnable command)`.
- **Пример**:
    
    ```java
    Executor executor = Executors.newSingleThreadExecutor();
    executor.execute(() -> System.out.println("Hello from thread"));
    ```
    

### 3.2. `ExecutorService`

- **Описание**: Расширяет `Executor`, добавляя управление задачами и их результатами.
- **Методы**:
    - `submit(Runnable/Callable)`: Запускает задачу, возвращает `Future`.
    - `invokeAll/invokeAny`: Выполнение коллекции задач.
    - `shutdown/shutdownNow`: Завершение пула.
- **Пример**:
    
    ```java
    ExecutorService executor = Executors.newFixedThreadPool(4);
    executor.submit(() -> System.out.println("Задача выполняется"));
    ```
    

## 4. Работа с результатами: `Future<T>`

- **Описание**: Представляет результат асинхронной задачи.
- **Методы**:
    - `get()`: Блокирует до получения результата.
    - `cancel(boolean)`: Отменяет задачу.
    - `isDone()`: Проверяет завершение.
    - `isCancelled()`: Проверяет отмену.
- **Пример**:
    
    ```java
    Future<String> future = executor.submit(() -> "Результат");
    String result = future.get(); // Блокирует
    ```
    

**Примечание**: `get()` блокирует поток, используйте с осторожностью.

## 5. Создание пулов: `Executors`

|Метод|Описание|
|---|---|
|`newFixedThreadPool(n)`|Пул с фиксированным числом потоков|
|`newCachedThreadPool()`|Динамический пул, удаляет простаивающие потоки|
|`newSingleThreadExecutor()`|Один поток для последовательного выполнения|
|`newScheduledThreadPool(n)`|Пул для периодических задач|

## 6. Внутреннее устройство: `ThreadPoolExecutor`

- **Основа**: Все методы `Executors` возвращают `ThreadPoolExecutor`.
- **Конструктор**:
    
    ```java
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
        int corePoolSize,
        int maximumPoolSize,
        long keepAliveTime,
        TimeUnit unit,
        BlockingQueue<Runnable> workQueue,
        ThreadFactory threadFactory,
        RejectedExecutionHandler handler
    );
    ```
    
- **Параметры**:
    - `corePoolSize`: Минимальное число активных потоков.
    - `maximumPoolSize`: Максимальное число потоков.
    - `keepAliveTime`: Время простоя для лишних потоков.
    - `workQueue`: Очередь задач (`LinkedBlockingQueue`, `ArrayBlockingQueue`, `SynchronousQueue`).
    - `handler`: Обработка переполнения (например, `AbortPolicy`, `CallerRunsPolicy`).

**Пример**:

```java
ExecutorService executor = new ThreadPoolExecutor(
    2, 4, 60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(10),
    Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy()
);
```

### 6.1. Очереди задач

- `LinkedBlockingQueue`: Неограниченная очередь (риск `OutOfMemoryError`).
- `ArrayBlockingQueue`: Ограниченная очередь.
- `SynchronousQueue`: Прямая передача задач без хранения.

### 6.2. JVM и JMM

- **AQS**: `ThreadPoolExecutor` использует AQS для управления очередями через `LockSupport` и CAS.
- **JMM**: `submit()` и `get()` обеспечивают **happens-before** для видимости результатов.
- **Байт-код (упрощённый)**:
    
    ```java
    invokevirtual java/util/concurrent/ExecutorService.submit(Ljava/util/concurrent/Callable;)Ljava/util/concurrent/Future;
    ```
    

## 7. Планирование задач: `ScheduledExecutorService`

- **Описание**: Пул для выполнения задач по расписанию.
- **Создание**:
    
    ```java
    ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
    ```
    
- **Методы**:
    - `schedule(Runnable/Callable, delay, TimeUnit)`: Однократное выполнение.
    - `scheduleAtFixedRate(Runnable, initialDelay, period, TimeUnit)`: Периодический запуск с фиксированным интервалом.
    - `scheduleWithFixedDelay(Runnable, initialDelay, delay, TimeUnit)`: Периодический запуск после завершения задачи.

**Пример**:

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.scheduleAtFixedRate(() -> System.out.println("Tick: " + LocalTime.now()), 0, 1, TimeUnit.SECONDS);
```

**Разница**:

- `scheduleAtFixedRate`: Интервал от начала предыдущей задачи.
- `scheduleWithFixedDelay`: Интервал после завершения предыдущей задачи.

## 8. Массовое выполнение: `invokeAll` и `invokeAny`

### 8.1. `invokeAll`

- Выполняет все задачи из коллекции, возвращает `List<Future<T>>`.
- **Пример**:
    
    ```java
    List<Callable<String>> tasks = Arrays.asList(
        () -> "task1", () -> "task2", () -> "task3"
    );
    List<Future<String>> results = executor.invokeAll(tasks);
    for (Future<String> f : results) {
        System.out.println(f.get());
    }
    ```
    

### 8.2. `invokeAny`

- Возвращает результат первой успешно завершённой задачи.
- **Пример**:
    
    ```java
    String result = executor.invokeAny(tasks);
    System.out.println("Первый результат: " + result);
    ```
    

**Таймаут**:

```java
executor.invokeAll(tasks, 2, TimeUnit.SECONDS);
```

## 9. Завершение пула

- `shutdown()`: Завершает новые задачи, ждёт выполнения текущих.
- `shutdownNow()`: Прерывает все задачи.
- `awaitTermination(timeout, unit)`: Ожидает завершения пула.
- **Пример**:
    
    ```java
    executor.shutdown();
    executor.awaitTermination(60, TimeUnit.SECONDS);
    ```
    

## 10. Современное асинхронное программирование: `CompletableFuture`

- **Описание**: Аналог `Promise`, поддерживает цепочки асинхронных операций.
- **Создание**:
    
    ```java
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "hello");
    ```
    
- **Цепочки**:
    
    ```java
    cf.thenApply(String::length)
      .thenAccept(System.out::println);
    ```
    
- **Обработка ошибок**:
    
    ```java
    cf.exceptionally(ex -> "default");
    cf.handle((res, ex) -> ex != null ? "fallback" : res);
    ```
    
- **Комбинация**:
    
    ```java
    CompletableFuture.allOf(cf1, cf2).join();
    ```
    

## 11. Рекурсивные задачи: `ForkJoinPool`

- **Описание**: Пул для задач с разбиением на подзадачи (divide-and-conquer).
- **Пример**:
    
    ```java
    ForkJoinPool pool = new ForkJoinPool();
    pool.invoke(new RecursiveTask<Integer>() {
        @Override
        protected Integer compute() {
            return 42; // Разбиение задачи
        }
    });
    ```
    

## 12. Практическое применение

- **Java EE / Jakarta EE**:
    - Асинхронная обработка запросов в сервлетах.
    - Управление задачами в EJB.
- **Spring**:
    - Асинхронные методы с `@Async` и `TaskExecutor`.
- **Тестирование**:
    - Параллельное выполнение тестов.
- **Параллельные вычисления**:
    - Обработка больших данных, машинное обучение.

**Пример (Spring)**:

```java
@Service
public class AsyncService {
    private final ExecutorService executor = Executors.newFixedThreadPool(4);

    @Async
    public CompletableFuture<String> processData(int id) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(1000);
                return "Результат " + id;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return "Ошибка";
            }
        }, executor);
    }
}
```

## 13. Современные возможности (Java 21+)

### 13.1. Виртуальные потоки

Пулы потоков эффективны с виртуальными потоками (Java 21, Project Loom).

**Пример**:

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> System.out.println("Задача в виртуальном потоке"));
```

**Особенности**:

- Лёгкие потоки снижают затраты на создание.
- Подходит для I/O-интенсивных задач.

### 13.2. Альтернативы

- **`CompletableFuture`**: Асинхронная композиция.
- **`StructuredTaskScope`**: Структурированная конкуренция (Java 21+).
- **`ForkJoinPool`**: Для рекурсивных задач.

## 14. Подводные камни

1. **Переполнение очереди**:
    - `LinkedBlockingQueue` без ограничений может вызвать `OutOfMemoryError`.
    - **Решение**: Используйте `ArrayBlockingQueue`.
2. **Игнорирование исключений**:
    
    ```java
    try {
        future.get();
    } catch (Exception e) {
        // Игнорирование
    }
    ```
    
    **Решение**: Обрабатывайте:
    
    ```java
    try {
        future.get();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } catch (ExecutionException e) {
        handleError(e.getCause());
    }
    ```
    
3. **Неправильное завершение**:
    - Забытый `shutdown()` оставляет пул активным.
    - **Решение**: Всегда вызывайте `shutdown()`.
4. **Высокая конкуренция**:
    - Слишком большой `corePoolSize` перегружает систему.
    - **Решение**: Настройте параметры пула.

## 15. Производительность

- **Преимущества**:
    - Минимизация затрат на создание потоков.
    - Эффективное управление через AQS.
- **Недостатки**:
    - Неправильная настройка очередей замедляет выполнение.
    - `scheduleAtFixedRate` может вызвать наложение задач.
- **Сравнение**:
    - `FixedThreadPool`: Стабильность, но ограниченный параллелизм.
    - `CachedThreadPool`: Гибкость, но риск перегрузки.
    - `CompletableFuture`: Асинхронность, меньше накладных расходов.
    - `ForkJoinPool`: Оптимизирован для рекурсии.
- **Рекомендации**:
    - Используйте ограниченные очереди.
    - Тестируйте параметры пула.

## 16. Плюсы и минусы

|Плюсы|Минусы|
|---|---|
|✅ Высокая производительность|❌ Риск переполнения очереди|
|✅ Гибкость управления|❌ Сложность настройки|
|✅ Поддержка виртуальных потоков|❌ Требует обработки исключений|
|✅ Асинхронные возможности|❌ Риск неправильного завершения|

## 17. Лучшие практики

1. **Используйте ограниченные очереди**:
    
    ```java
    new ThreadPoolExecutor(2, 4, 60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(10));
    ```
    
2. **Обрабатывайте исключения**:
    
    ```java
    Future<String> future = executor.submit(callableTask);
    try {
        future.get();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } catch (ExecutionException e) {
        handleError(e.getCause());
    }
    ```
    
3. **Завершайте пул**:
    
    ```java
    executor.shutdown();
    executor.awaitTermination(60, TimeUnit.SECONDS);
    ```
    
4. **Тестируйте производительность**:
    
    ```java
    @Test
    void testThreadPool() throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        AtomicInteger counter = new AtomicInteger();
        executor.submit(() -> counter.incrementAndGet());
        executor.submit(() -> counter.incrementAndGet());
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        assertEquals(2, counter.get());
    }
    ```
    
5. **Используйте `CompletableFuture` для асинхронности**:
    
    ```java
    CompletableFuture.supplyAsync(() -> "Результат", executor).thenAccept(System.out::println);
    ```
    

## 18. Пример: Параллельная обработка данных

```java
import java.util.concurrent.*;
import java.util.stream.IntStream;

public class DataProcessing {
    private final ExecutorService executor;

    public DataProcessing(int poolSize) {
        this.executor = new ThreadPoolExecutor(
            poolSize, poolSize, 0, TimeUnit.SECONDS, new ArrayBlockingQueue<>(100)
        );
    }

    public List<Future<Integer>> processData(int[] data) {
        return IntStream.range(0, data.length)
            .mapToObj(i -> executor.submit(() -> processItem(data[i])))
            .toList();
    }

    private int processItem(int item) {
        try {
            Thread.sleep(500);
            return item * 2;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return 0;
        }
    }

    public void shutdown() {
        executor.shutdown();
    }
}
```

**Объяснение**:

- Пул обрабатывает массив данных параллельно.
- Ограниченная очередь (`ArrayBlockingQueue`) предотвращает перегрузку.
- `Future` собирает результаты.

## Senior Insights

### ThreadPoolExecutor: логика заполнения (очень важно!)

> [!WARNING] Контринтуитивная логика: `ThreadPoolExecutor` сначала заполняет ОЧЕРЕДЬ, потом создаёт дополнительные потоки

```
Логика обработки submit(task) в ThreadPoolExecutor:

1. if (running threads < corePoolSize):
     → создать новый thread, выполнить task (даже если очередь пустая!)
     → corePool threads всегда живут (keepAliveTime не применяется, если не allowCoreThreadTimeOut)

2. if (running threads >= corePoolSize):
     → попробовать добавить task в workQueue
     → если queue.offer(task) == true: задача в очереди, потоки её заберут
     → если queue.offer(task) == false (очередь полна):

3. if (queue full AND running threads < maximumPoolSize):
     → создать новый thread (выше corePoolSize), выполнить task
     → эти потоки живут keepAliveTime после простоя

4. if (queue full AND running threads >= maximumPoolSize):
     → RejectedExecutionHandler!
     → Default: AbortPolicy → throw RejectedExecutionException
     → CallerRunsPolicy → выполнить task в вызывающем потоке (backpressure!)
     → DiscardPolicy → молча отбросить
     → DiscardOldestPolicy → выбросить самую старую задачу из очереди
```

```java
// ПРИМЕР: почему Executors.newFixedThreadPool опасен
Executors.newFixedThreadPool(4);
// Внутри: new ThreadPoolExecutor(4, 4, 0, SECONDS, new LinkedBlockingQueue<>())
// LinkedBlockingQueue() = НЕОГРАНИЧЕННАЯ очередь (Integer.MAX_VALUE capacity)!
// При нагрузке: очередь растёт бесконечно → OOM

// ПРАВИЛЬНО: явные параметры с bounded queue
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                              // corePoolSize
    8,                              // maximumPoolSize
    60, TimeUnit.SECONDS,           // keepAliveTime для core+ threads
    new ArrayBlockingQueue<>(200),  // bounded queue!
    new ThreadFactory() {
        AtomicInteger count = new AtomicInteger();
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "worker-" + count.incrementAndGet());
            t.setDaemon(true); // не блокировать JVM shutdown
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // backpressure вместо отбрасывания
);

// Мониторинг:
executor.getPoolSize();        // текущий размер пула
executor.getActiveCount();     // активные (выполняющие) потоки
executor.getQueue().size();    // очередь
executor.getCompletedTaskCount(); // завершённые
```

### CompletableFuture: Senior паттерны и ловушки

```java
// 1. Цепочка: thenApply (sync) vs thenApplyAsync (async)
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> fetchUser(id), ioExecutor)   // I/O executor
    .thenApply(user -> user.getName())               // sync: в том же потоке что завершил supplyAsync
    .thenApplyAsync(name -> enrich(name), cpuExecutor) // async: в cpuExecutor
    .thenApply(enriched -> format(enriched));        // sync: в cpuExecutor потоке

// ЛОВУШКА: без explicit executor → thenApplyAsync использует ForkJoinPool.commonPool()
// Не используй для I/O операций!

// 2. Обработка исключений:
CompletableFuture<String> safe = future
    .thenApply(this::process)
    .exceptionally(ex -> {
        log.error("Failed", ex);
        return "default";         // recover value
    })
    .whenComplete((result, ex) -> { // ВСЕГДА выполняется (success или failure)
        metrics.record(ex == null ? "ok" : "error");
        // не может изменить результат!
    })
    .handle((result, ex) -> {       // может изменить результат
        if (ex != null) return fallback();
        return result;
    });

// 3. Параллельное выполнение:
// allOf — ждём все, тип Void
CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);
all.thenRun(() -> {
    String r1 = f1.join(); // join() = get() без checked exception
    String r2 = f2.join(); // безопасно после allOf
});

// anyOf — ждём первый (любой)
CompletableFuture<Object> first = CompletableFuture.anyOf(f1, f2, f3);

// 4. Timeout (Java 9+):
future.orTimeout(5, TimeUnit.SECONDS);             // TimeoutException при превышении
future.completeOnTimeout("default", 5, SECONDS);  // default value при превышении

// 5. Цепочка с flat-map (thenCompose):
CompletableFuture<User> getUser(long id) { ... }
CompletableFuture<Order> getOrder(User user) { ... }

// thenApply даст CompletableFuture<CompletableFuture<Order>> — НЕПРАВИЛЬНО
// thenCompose = flatMap → CompletableFuture<Order>
CompletableFuture<Order> order = getUser(userId)
    .thenCompose(user -> getOrder(user)); // thenCompose = flatMap!

// 6. ЛОВУШКА: join() в потоке ForkJoinPool — deadlock!
// BAD: join() блокирует ForkJoinPool worker thread, который нужен для выполнения
CompletableFuture.supplyAsync(() -> {
    return CompletableFuture.supplyAsync(() -> "inner").join(); // МОЖЕТ ДЕДЛОКНУТЬ!
}, ForkJoinPool.commonPool());

// GOOD: thenCompose вместо вложенного join():
CompletableFuture.supplyAsync(() -> "outer")
    .thenCompose(outer -> CompletableFuture.supplyAsync(() -> outer + "inner"));
```

### ThreadPool + Virtual Threads: когда что использовать (Java 21+)

```java
// I/O-bound: виртуальные потоки (один поток на задачу)
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Response>> futures = requests.stream()
        .map(req -> exec.submit(() -> httpClient.send(req)))
        .toList();
}
// Почему: каждый virtual thread unmounts при I/O → N virtual threads на M carrier threads
// Нет смысла в пуле - создание/уничтожение virtual thread бесплатно (~µs)

// CPU-bound: фиксированный пул platform threads = N ядер
ExecutorService cpuPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);
// Почему: virtual threads не дают выигрыша для CPU-bound
// Больше потоков чем ядер = context switching overhead

// Mixed (I/O + CPU): раздельные пулы
ExecutorService ioPool = Executors.newVirtualThreadPerTaskExecutor();
ExecutorService cpuPool = Executors.newFixedThreadPool(nCores);

CompletableFuture.supplyAsync(() -> fetchData(url), ioPool)  // I/O: virtual
    .thenApplyAsync(data -> process(data), cpuPool);         // CPU: platform
```

---

## 19. Заключение

Пулы потоков — мощный инструмент для управления многопоточными задачами. `ExecutorService`, `ScheduledExecutorService`, `CompletableFuture`, и `ForkJoinPool` обеспечивают гибкость и производительность. Основанные на AQS, они используют атомарные операции и `LockSupport` для потокобезопасности. Поддержка виртуальных потоков (Java 21+) делает их ещё эффективнее. Следование лучшим практикам минимизирует риски переполнения очередей и неправильного завершения.

|Задача|Интерфейс/Класс|Комментарий|
|---|---|---|
|Простая задача|`Runnable`|Быстро и просто|
|Задача с результатом|`Callable<V>`|Для возвращаемых значений и исключений|
|Асинхронное выполнение|`ExecutorService`|Управление пулом потоков|
|Получение результата|`Future<V>`|Ожидание завершения|
|Периодические задачи|`ScheduledExecutorService`|Планирование по времени|
|Асинхронная композиция|`CompletableFuture`|Современный подход|
|Рекурсивные задачи|`ForkJoinPool`|Параллельные вычисления|