# Structured Concurrency (Java 21+)

> **Structured Concurrency** (Java 21, JEP 453) — параллельные задачи живут внутри try-with-resources scope. Когда scope закрывается — все дочерние задачи автоматически отменяются. Предсказуемое время жизни, без утечек потоков.
> На интервью: чем отличается от CompletableFuture, как гарантированно нет утечек, ShutdownOnFailure vs ShutdownOnSuccess, как работает с ScopedValues.

## Связанные темы
[[Scoped Values (Java 21, JEP 446)]], [[Процессы и Потоки, Thread, Runnable, состояния потоков]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]]

---

## Проблема: неструктурированная конкурентность

```java
// CompletableFuture — трудно управлять ресурсами и ошибками:
public UserProfile getProfile(long userId) {
    var userFuture   = CompletableFuture.supplyAsync(() -> userService.findById(userId));
    var ordersFuture = CompletableFuture.supplyAsync(() -> orderService.findByUser(userId));

    try {
        User user = userFuture.get(5, TimeUnit.SECONDS);
        List<Order> orders = ordersFuture.get(5, TimeUnit.SECONDS);
        return new UserProfile(user, orders);
    } catch (Exception e) {
        // ordersFuture всё ещё работает даже если userFuture упал!
        // Утечка потоков — нет автоматической отмены
        throw new RuntimeException(e);
    }
}
```

**Ключевой инвариант StructuredTaskScope:** `close()` **гарантирует**, что все дочерние задачи завершены до выхода из scope. Никаких утечек потоков.

---

## ShutdownOnFailure — "все или ничего"

```java
public UserProfile getProfile(long userId) throws InterruptedException, ExecutionException {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

        // fork() запускает виртуальный поток немедленно:
        Subtask<User>         userTask    = scope.fork(() -> userService.findById(userId));
        Subtask<List<Order>>  ordersTask  = scope.fork(() -> orderService.findByUser(userId));
        Subtask<List<Address>> addrTask   = scope.fork(() -> addressService.findByUser(userId));

        scope.join()           // ждём ВСЕ задачи ИЛИ первую ошибку
             .throwIfFailed(); // бросаем если любая задача упала

        // Сюда приходим только если ВСЕ задачи успешны:
        return new UserProfile(userTask.get(), ordersTask.get(), addrTask.get());

    } // close(): отменяет оставшиеся задачи и ждёт их завершения
}
```

**Что происходит при ошибке:**
1. `userService.findById()` бросает исключение
2. `ShutdownOnFailure` немедленно interrupt-ит все оставшиеся задачи
3. `join()` возвращает
4. `throwIfFailed()` бросает первое исключение
5. `close()` ждёт завершения всех прерванных задач

---

## ShutdownOnSuccess — "первый победитель"

```java
public String findFastestSource(String query) throws InterruptedException, ExecutionException {
    try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {

        scope.fork(() -> primaryDatabase.search(query));
        scope.fork(() -> cacheService.search(query));
        scope.fork(() -> searchIndex.search(query));

        scope.join(); // ждём ПЕРВЫЙ успешный результат

        return scope.result(); // первая успешная задача
        // Остальные — автоматически отменены
    }
}
```

**С таймаутом:**
```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userTask   = scope.fork(() -> userService.findById(userId));
    var ordersTask = scope.fork(() -> orderService.findByUser(userId));

    scope.joinUntil(Instant.now().plusSeconds(3)); // TimeoutException если не успели
    scope.throwIfFailed();

    return new UserProfile(userTask.get(), ordersTask.get());
}
```

---

## Subtask: проверка состояния

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var task = scope.fork(() -> riskyOperation());
    scope.join();

    switch (task.state()) {
        case SUCCESS     -> { var result = task.get(); /* OK */ }
        case FAILED      -> { var ex = task.exception(); /* обработать */ }
        case UNAVAILABLE -> { /* задача была отменена shutdown-ом */ }
    }
}
```

`task.get()` безопасно вызывать только при `SUCCESS` — иначе `IllegalStateException`.

---

## Кастомный StructuredTaskScope: QuorumScope

```java
// "Кворум" — успех если минимум N из M задач успешны:
public class QuorumScope<T> extends StructuredTaskScope<T> {
    private final int quorum;
    private final List<T> results = new CopyOnWriteArrayList<>();
    private final List<Throwable> exceptions = new CopyOnWriteArrayList<>();

    QuorumScope(int quorum) { this.quorum = quorum; }

    @Override
    protected void handleComplete(Subtask<? extends T> task) {
        switch (task.state()) {
            case SUCCESS -> {
                results.add(task.get());
                if (results.size() >= quorum) shutdown(); // достигли кворума
            }
            case FAILED  -> exceptions.add(task.exception());
        }
    }

    List<T> results() throws ExecutionException {
        ensureOwnerAndJoined();
        if (results.size() < quorum)
            throw new ExecutionException("Quorum not reached",
                exceptions.isEmpty() ? null : exceptions.get(0));
        return List.copyOf(results);
    }
}

// Использование (репликация БД — кворум 2 из 3):
try (var scope = new QuorumScope<String>(2)) {
    scope.fork(() -> replica1.read(key));
    scope.fork(() -> replica2.read(key));
    scope.fork(() -> replica3.read(key));

    scope.join();
    List<String> quorumResults = scope.results();
}
```

---

## Интеграция со Scoped Values

```java
static final ScopedValue<TraceContext> TRACE       = ScopedValue.newInstance();
static final ScopedValue<User>         CURRENT_USER = ScopedValue.newInstance();

void handleRequest(TraceContext ctx, User user) throws Exception {
    ScopedValue.where(TRACE, ctx)
               .where(CURRENT_USER, user)
               .run(() -> {
                   try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                       // Все fork-нутые задачи автоматически наследуют TRACE и CURRENT_USER!
                       scope.fork(() -> {
                           log("DB query: user={}, trace={}", CURRENT_USER.get(), TRACE.get());
                           return db.query(TRACE.get().traceId());
                       });
                       scope.fork(() -> cache.get(CURRENT_USER.get().id()));

                       scope.join().throwIfFailed();
                   } catch (Exception e) { throw new RuntimeException(e); }
               });
}
```

---

## StructuredTaskScope vs CompletableFuture

| | CompletableFuture | StructuredTaskScope |
|---|---|---|
| Отмена при ошибке | Ручная | Автоматическая |
| Утечки потоков | Возможны | Невозможны (scope) |
| Жизненный цикл | Размытый | Чёткий (try-with-resources) |
| Exception handling | Сложный (`.exceptionally()`) | Прямой (`throwIfFailed()`) |
| Отладка stektrace | 15+ строк FJP internals | Прямой stack от fork() до ошибки |
| Scoped Values | Нет автонаследования | Автоматически в fork() |
| Thread type | Platform/Virtual | Virtual (default) |
| Java версия | Java 8+ | Java 21+ |
| "Первый выигрывает" | `anyOf()` (без отмены остальных) | `ShutdownOnSuccess` (с отменой) |

**Когда CompletableFuture всё ещё лучше:**
- Длинные pipeline трансформаций: `.thenApply().thenCompose().thenAccept()`
- Взаимодействие с существующим async API (библиотека возвращает CF)
- Java < 21

---

## Вопросы на интервью

- Как StructuredTaskScope гарантирует отсутствие утечек потоков?
- Чем `ShutdownOnFailure` отличается от `ShutdownOnSuccess`?
- Почему structured concurrency называется "структурным"? Аналогия с обычным кодом?
- Как реализовать кворум поверх StructuredTaskScope?
- Как `ScopedValue` наследуется в `fork()` без копирования?
- Что вернёт `task.get()` если задача была отменена (UNAVAILABLE)?
- Почему рекомендуется использовать virtual threads с StructuredTaskScope?
- Как добавить таймаут на весь scope?

---

## Подводные камни

- **`task.get()` до `join()`** — `IllegalStateException`: состояние задачи ещё не определено. Всегда сначала `scope.join()`, потом читай результаты.
- **`scope.fork()` после `join()`** — `IllegalStateException`: нельзя добавлять задачи после ожидания.
- **Checked exceptions внутри fork()** — `fork()` принимает `Callable<T>` (может бросать исключения), но лямбда требует обработки checked. Оборачивай в unchecked или используй вспомогательный метод.
- **Pinning в fork()-нутых задачах** — если задача внутри `synchronized` блока делает I/O, carrier thread пиннирован. Используй `ReentrantLock`.
- **ScopedValue + ExecutorService** — значение НЕ наследуется при `executor.submit()`. Только `StructuredTaskScope.fork()` гарантирует наследование.
- **Вложенные StructuredTaskScope** — допустимы, но scope должны быть строго вложены. Нельзя передать `Subtask` из внутреннего scope во внешний.
