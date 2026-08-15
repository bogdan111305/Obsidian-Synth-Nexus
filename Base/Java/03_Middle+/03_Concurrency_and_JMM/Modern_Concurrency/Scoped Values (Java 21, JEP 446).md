# Scoped Values (Java 21, JEP 446)

> **ScopedValue** — иммутабельная замена `ThreadLocal` для виртуальных потоков. `ThreadLocal` накапливает утечки при миллионах virtual threads. `ScopedValue.where(KEY, value).run(task)` — данные доступны только в scope, без наследования мусора.
> На интервью: почему ThreadLocal плох с virtual threads, как работает rebinding, как ScopedValue наследуется без копирования, когда ThreadLocal всё ещё нужен.

## Связанные темы
[[Процессы и Потоки, Thread, Runnable, состояния потоков]], [[Structured Concurrency]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]], [[ThreadLocal]], [[Virtual Threads — модель и архитектура]]

---

## Проблемы ThreadLocal с виртуальными потоками

Полное устройство `ThreadLocal`/`ThreadLocalMap`, утечки в пулах потоков и стоимость `InheritableThreadLocal` разобраны в [[ThreadLocal]]. Здесь — три причины, по которым эти проблемы стали критичными именно с приходом виртуальных потоков:

1. **`InheritableThreadLocal` копирует всю `ThreadLocalMap` при старте каждого потока** (O(n), не O(1)). При тысячах платформенных потоков это было незаметно; при миллионе виртуальных — миллион копий карты, ощутимый overhead памяти.
2. **Мутабельность — состояние непредсказуемо в динамике.** Любой код в стеке вызовов может незаметно сделать `set()`:
   ```java
   threadLocal.set("A");
   callFramework();               // может вызвать threadLocal.set("B") под капотом
   String val = threadLocal.get(); // "B" — неожиданно!
   ```
   Это не зависит от VT напрямую, но при миллионах короткоживущих VT-задач цена случайной мутации в общем framework-коде растёт.
3. **Забытый `remove()` = утечка на весь срок жизни потока в пуле** — с VT-per-task эта проблема формально исчезает (поток одноразовый), но при повторном использовании `Thread.ofVirtual()`-подобных паттернов или при переносе legacy-кода на VT грабли остаются те же.

---

## ScopedValue API

```java
// Объявление (статическое, как ThreadLocal):
static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
static final ScopedValue<String> TRACE_ID = ScopedValue.newInstance();

// Установка и запуск:
ScopedValue.where(CURRENT_USER, user)
           .where(TRACE_ID, "abc-123")
           .run(() -> processRequest());   // void
           // или .call(() -> fetchData()); // Callable<T> → T

// Чтение (только внутри scope):
User user = CURRENT_USER.get();                         // NoSuchElementException если не bound
User user = CURRENT_USER.orElse(User.anonymous());     // safe default
User user = CURRENT_USER.orElseThrow(IllegalStateException::new);
boolean bound = CURRENT_USER.isBound();                // проверить наличие

// Нет метода set() — иммутабельность гарантирована компилятором
```

---

## Иммутабельность и rebinding

Значение нельзя изменить — можно только перекрыть во вложенном scope (rebinding):

```java
static final ScopedValue<String> ROLE = ScopedValue.newInstance();

ScopedValue.where(ROLE, "USER").run(() -> {
    System.out.println(ROLE.get()); // "USER"

    ScopedValue.where(ROLE, "ADMIN").run(() -> {
        System.out.println(ROLE.get()); // "ADMIN" — перекрыто
    });

    System.out.println(ROLE.get()); // "USER" — восстановлено автоматически!
});
// ROLE.isBound() == false — автоочистка
```

Rebinding создаёт вложенный scope. При выходе — предыдущее значение восстанавливается. Stack-подобная семантика.

---

## Наследование в StructuredTaskScope

`ScopedValue` **автоматически** наследуется в дочерних задачах `StructuredTaskScope.fork()`:

```java
static final ScopedValue<TraceId> TRACE_ID = ScopedValue.newInstance();
static final ScopedValue<String> TENANT_ID = ScopedValue.newInstance();

void processRequest(String tenantId, TraceId traceId) {
    ScopedValue.where(TRACE_ID, traceId)
               .where(TENANT_ID, tenantId)
               .run(() -> {
                   try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                       var dbTask   = scope.fork(this::fetchFromDatabase);  // видит TRACE_ID
                       var apiTask  = scope.fork(this::callExternalApi);    // видит TENANT_ID

                       scope.join().throwIfFailed();
                       combine(dbTask.get(), apiTask.get());
                   } catch (Exception e) { throw new RuntimeException(e); }
               });
}

String fetchFromDatabase() {
    // Автоматически наследует все ScopedValue из parent scope
    log("tenant={}, trace={}", TENANT_ID.get(), TRACE_ID.get());
    return "db-result";
}
```

**Почему без копирования:** `ScopedValue` хранит привязки как иммутабельный linked-list. `fork()` передаёт **ссылку** на тот же `Bindings` объект (O(1)), не копирует (vs `InheritableThreadLocal` — O(n) копирование ThreadLocalMap).

> [!WARNING] ScopedValue НЕ наследуется в обычный `ExecutorService.submit()` — только в `StructuredTaskScope.fork()`.

---

## ScopedValue vs ThreadLocal

| | `ThreadLocal` | `ScopedValue` |
|---|---|---|
| Мутабельность | Да (`set()` в любом месте) | Нет (только `where().run()`) |
| Scope | Весь поток до `remove()` | Явно ограниченный блок |
| Автоочистка | Нет (`remove()` обязателен) | Да (при выходе из scope) |
| Наследование | `InheritableThreadLocal` (O(n) копирование) | Автоматически (O(1) sharing) |
| Virtual Threads | Проблемы с памятью (1M копий Map) | Эффективно |
| Утечки в thread pool | Частая проблема | Невозможны |
| Java версия | Java 1.2 | Java 21 (stable) |

---

## Миграция с ThreadLocal

```java
// Старый код:
public class SecurityContext {
    private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();

    public static void setUser(User user) { CURRENT_USER.set(user); }
    public static User getUser() { return CURRENT_USER.get(); }
    public static void clear() { CURRENT_USER.remove(); } // не забыть!
}

void handleRequest(User user) {
    SecurityContext.setUser(user);
    try { processRequest(); }
    finally { SecurityContext.clear(); }
}

// Новый код с ScopedValue:
public class SecurityContext {
    static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

    public static <T> T runAs(User user, Callable<T> action) throws Exception {
        return ScopedValue.where(CURRENT_USER, user).call(action);
    }

    public static User getUser() {
        return CURRENT_USER.orElseThrow(() ->
            new IllegalStateException("No user in security context"));
    }
}

void handleRequest(User user) throws Exception {
    SecurityContext.runAs(user, () -> { processRequest(); return null; });
    // Автоочистка — никакого finally!
}
```

**Spring MVC:** обернуть весь request в ScopedValue scope через `OncePerRequestFilter`. Interceptor не подходит — нужно обернуть `chain.doFilter()`.

**Spring WebFlux:** ScopedValue не работает с Reactor Mono/Flux цепочками — реактивный pipeline может переключать потоки вне scope. Используй `ReactiveSecurityContextHolder`.

---

## Вопросы на интервью

- Почему `ThreadLocal` стал проблемой с появлением виртуальных потоков? (InheritableThreadLocal overhead + мутабельность)
- Что такое rebinding в ScopedValue? Как восстанавливается предыдущее значение?
- Как `ScopedValue` наследуется в `StructuredTaskScope.fork()` без копирования?
- Чем `ScopedValue.run()` отличается от `ScopedValue.call()`?
- Что вернёт `ScopedValue.get()` вне scope? (NoSuchElementException)
- Когда `ThreadLocal` всё ещё предпочтительнее `ScopedValue`?
- Почему `ScopedValue` не наследуется в `ExecutorService.submit()`?

---

## Подводные камни

- **`get()` вне scope → `NoSuchElementException`** — в отличие от ThreadLocal, который возвращает null. Используй `isBound()` или `orElse()` для defensive кода.
- **Не наследуется в обычный ExecutorService** — только `StructuredTaskScope.fork()`. При использовании `executor.submit()` нужно явно передавать значения или capture в лямбду.
- **ScopedValue не для мутабельного состояния** — если задача требует изменения данных в ходе обработки, нужен `AtomicReference` или `ThreadLocal`.
- **WebFlux + ScopedValue = проблемы** — Reactor pipeline может быть опубликован на другом потоке вне scope. Используй Reactor Context (`contextWrite`).
- **Нет set() — это фича, не баг** — если нужна мутабельность, это сигнал что архитектура требует пересмотра, а не обходного пути.
