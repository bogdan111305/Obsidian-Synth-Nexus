---
title: "Scoped Values — Java 21 (JEP 446)"
tags: [java, concurrency, java21, loom, scopedvalue, threadlocal, structured-concurrency]
updated: 2026-03-04
---

# Scoped Values (Java 21, JEP 446)

> [!QUOTE] Суть
> **ScopedValue** — иммутабельная замена `ThreadLocal` для виртуальных потоков. `ThreadLocal` накапливает утечки при миллионах virtual threads. `ScopedValue.where(KEY, value).run(task)` — данные доступны только в scope, без наследования мусора.

**Scoped Values** — новый механизм для передачи иммутабельных данных потоку (и его дочерним потокам) в рамках ограниченного scope. Появился в Java 20 как preview (JEP 429), стабилизирован в Java 21 (JEP 446). Решает ключевые проблемы `ThreadLocal` в эпоху виртуальных потоков.

## 1. Проблемы ThreadLocal с виртуальными потоками

`ThreadLocal` создавался для платформенных потоков (их мало — десятки/сотни). С **виртуальными потоками** (Java 21, Project Loom) появляются новые проблемы:

### 1.1. Накладные расходы при наследовании

```java
// InheritableThreadLocal наследует значения дочерним потокам
static final InheritableThreadLocal<User> CURRENT_USER = new InheritableThreadLocal<>();

// При создании виртуального потока — копирование ThreadLocalMap
Thread.ofVirtual().start(() -> {
    // JVM скопировала ВЕСЬ ThreadLocalMap родителя!
    // Если виртуальных потоков 1 000 000 — это 1M копий Map
    User user = CURRENT_USER.get();
});
```

При 1 000 000 виртуальных потоков с `InheritableThreadLocal` — JVM создаёт 1 000 000 копий `ThreadLocalMap`. Overhead памяти огромный.

### 1.2. Мутабельность — источник ошибок

```java
static final ThreadLocal<List<String>> LOG = ThreadLocal.withInitial(ArrayList::new);

void process() {
    LOG.get().add("step1");   // мутирует состояние потока
    // другой код тоже может изменить LOG
    // нет гарантий, что значение не изменится в ходе выполнения
}
```

`ThreadLocal` — **мутабельный**: любой код в потоке может вызвать `set()` и изменить значение. Это источник трудноуловимых bagов.

### 1.3. Отсутствие автоматической очистки

```java
// ThreadLocal в пуле потоков — классическая утечка:
static final ThreadLocal<Connection> DB_CONN = new ThreadLocal<>();

void handleRequest() {
    DB_CONN.set(openConnection());
    // ... работаем
    // ЗАБЫЛИ вызвать DB_CONN.remove() !
    // Connection утёк, поток вернулся в пул с "грязным" ThreadLocal
}
```

## 2. ScopedValue API

`ScopedValue` — **иммутабельная** переменная, видимая только в рамках **явно ограниченного scope** (блока кода).

### 2.1. Базовое использование

```java
import java.lang.ScopedValue;

public class RequestHandler {
    // Объявляем ScopedValue (аналог ThreadLocal объекта)
    static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
    static final ScopedValue<RequestId> REQUEST_ID = ScopedValue.newInstance();

    void handleRequest(User user, RequestId requestId) {
        // Устанавливаем значение и запускаем код в scope
        ScopedValue.where(CURRENT_USER, user)
                   .where(REQUEST_ID, requestId)
                   .run(() -> {
                       processRequest();
                   });
        // После run() — значения автоматически недоступны
    }

    void processRequest() {
        User user = CURRENT_USER.get();         // получить значение
        RequestId id = REQUEST_ID.get();        // безопасно в любом месте scope
        System.out.println("Processing for: " + user + ", request: " + id);
        // Нельзя вызвать CURRENT_USER.set() — нет такого метода!
    }
}
```

### 2.2. `call()` вместо `run()` — возврат значения

```java
// run() → void; call() → возвращает значение (может бросить исключение)
String result = ScopedValue.where(CURRENT_USER, user)
                            .call(() -> fetchUserData()); // Callable<String>
```

### 2.3. Проверка наличия значения

```java
if (CURRENT_USER.isBound()) {
    User user = CURRENT_USER.get();
    // ...
} else {
    // ScopedValue не установлен в текущем scope
}

// orElse — безопасное получение с дефолтом:
User user = CURRENT_USER.orElse(User.anonymous());

// orElseThrow:
User user = CURRENT_USER.orElseThrow(() -> new IllegalStateException("No user in scope"));
```

## 3. Иммутабельность и rebinding

### 3.1. Значение нельзя изменить — можно только перекрыть (rebind)

```java
static final ScopedValue<String> ROLE = ScopedValue.newInstance();

void outerScope() {
    ScopedValue.where(ROLE, "USER").run(() -> {
        System.out.println(ROLE.get()); // "USER"

        // Перекрываем значение во вложенном scope (rebind):
        ScopedValue.where(ROLE, "ADMIN").run(() -> {
            System.out.println(ROLE.get()); // "ADMIN"
        });

        System.out.println(ROLE.get()); // "USER" — восстановлено!
    });
    // ROLE.isBound() == false
}
```

Rebinding создаёт **вложенный scope** с новым значением. Старое значение восстанавливается автоматически при выходе из вложенного scope.

### 3.2. Иммутабельность vs ThreadLocal

```java
// ThreadLocal — можно изменить из любого места:
threadLocal.set("A");
callSomething();  // может вызвать threadLocal.set("B") — мутация!
String val = threadLocal.get(); // "B" — неожиданно!

// ScopedValue — только чтение внутри scope:
ScopedValue.where(scoped, "A").run(() -> {
    callSomething();  // не может изменить scoped — нет set()
    String val = scoped.get(); // гарантированно "A"
});
```

## 4. Наследование в дочерних задачах

`ScopedValue` **автоматически наследуется** дочерними структурами при использовании со **Structured Concurrency**:

```java
import java.util.concurrent.StructuredTaskScope;

static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

void handleRequest(User user) throws InterruptedException, ExecutionException {
    ScopedValue.where(CURRENT_USER, user).run(() -> {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            // Дочерние задачи автоматически видят CURRENT_USER:
            var task1 = scope.fork(() -> fetchOrders(CURRENT_USER.get()));
            var task2 = scope.fork(() -> fetchProfile(CURRENT_USER.get()));

            scope.join().throwIfFailed();

            processResults(task1.get(), task2.get());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    });
}
```

**Разница с `InheritableThreadLocal`:**
- `InheritableThreadLocal` копирует **всю ThreadLocalMap** при создании потока (O(n) overhead)
- `ScopedValue` использует **immutable bindings** — структура разделяется без копирования (O(1))

## 5. Structured Concurrency + Scoped Values паттерн

```java
static final ScopedValue<TraceId> TRACE_ID = ScopedValue.newInstance();
static final ScopedValue<String> TENANT_ID = ScopedValue.newInstance();

void processMultiTenantRequest(String tenantId, TraceId traceId) {
    ScopedValue.where(TRACE_ID, traceId)
               .where(TENANT_ID, tenantId)
               .run(() -> {
                   try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                       var dbFuture = scope.fork(this::fetchFromDatabase);
                       var cacheFuture = scope.fork(this::fetchFromCache);
                       var apiFuture = scope.fork(this::callExternalApi);

                       scope.join().throwIfFailed();

                       // Все 3 задачи видели одинаковые TRACE_ID и TENANT_ID
                       combine(dbFuture.get(), cacheFuture.get(), apiFuture.get());
                   } catch (Exception e) {
                       throw new RuntimeException(e);
                   }
               });
}

String fetchFromDatabase() {
    // Автоматически наследует TRACE_ID и TENANT_ID из parent scope
    log("DB query for tenant: " + TENANT_ID.get() + ", trace: " + TRACE_ID.get());
    return "db-result";
}
```

## 6. Сравнение ScopedValue vs ThreadLocal

|Характеристика|ThreadLocal|ScopedValue|
|---|---|---|
|Мутабельность|Да (`set()` в любом месте)|Нет (только `where().run()`)|
|Scope|Весь поток до `remove()`|Явно ограниченный блок (`run()`/`call()`)|
|Автоматическая очистка|Нет (нужен `remove()`)|Да (при выходе из scope)|
|Наследование|`InheritableThreadLocal` (копирование)|Автоматически (без копирования)|
|Виртуальные потоки|Проблемы с памятью (1M копий Map)|Эффективно (sharing bindings)|
|Семантика|Каждый поток — своё значение|Иммутабельный контекст для scope|
|Утечки в thread pool|Частая проблема|Невозможны (scope ограничен)|
|Java версия|Java 1.2|Java 21 (stable)|

## 7. Миграция с ThreadLocal на ScopedValue

**Типичный паттерн ThreadLocal:**

```java
// Старый код:
public class SecurityContext {
    private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();

    public static void setUser(User user) { CURRENT_USER.set(user); }
    public static User getUser() { return CURRENT_USER.get(); }
    public static void clear() { CURRENT_USER.remove(); }
}

void handleRequest(User user) {
    SecurityContext.setUser(user);
    try {
        processRequest();
    } finally {
        SecurityContext.clear(); // не забыть!
    }
}
```

**Миграция на ScopedValue:**

```java
// Новый код:
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
    SecurityContext.runAs(user, () -> {
        processRequest();
        return null;
    });
    // Автоматически очищается — нет need for finally!
}
```

## 8. Ограничения и подводные камни

1. **Требует явного scope** — нельзя передать значение "глобально" на всё время жизни потока (как ThreadLocal). Если нужна мутабельность — ThreadLocal всё ещё подходит.

2. **Java 21+** — недоступен в старых версиях. Для совместимости нужен ThreadLocal.

3. **`ScopedValue` и `ExecutorService`** (без Structured Concurrency):
```java
// Значение НЕ наследуется автоматически в обычный ExecutorService:
ScopedValue.where(CURRENT_USER, user).run(() -> {
    executor.submit(() -> {
        CURRENT_USER.get(); // IllegalStateException — не в scope!
    });
});
// Используйте StructuredTaskScope вместо ExecutorService
```

4. **Не используйте для мутабельного состояния** — если задача требует изменения данных, нужен другой механизм (AtomicReference, ConcurrentHashMap).

## Interview Q&A (Senior Level)

**Q1: Почему `ThreadLocal` стал проблемой с появлением виртуальных потоков?**

> Два основных ограничения: (1) **`InheritableThreadLocal` overhead**: при создании виртуального потока JVM копирует `ThreadLocalMap` родителя. При 1 000 000 виртуальных потоков — 1M копий Map → огромный overhead памяти. (2) **Мутабельность**: любой код в потоке может вызвать `ThreadLocal.set()`, что затрудняет рассуждение о состоянии, особенно в асинхронном коде с виртуальными потоками где поток может быть "заимствован" пулом.

**Q2: В чём разница между rebinding и set() в контексте ScopedValue vs ThreadLocal?**

> `ThreadLocal.set()` изменяет значение **мутабельно** — старое значение теряется, любой код в потоке может изменить его снова. `ScopedValue` **не имеет `set()`** — только `where(value).run()`. Это создаёт вложенный иммутабельный scope: внутри блока значение гарантированно неизменно, при выходе из блока автоматически восстанавливается предыдущее значение (stack-like семантика). Это делает rebinding безопасным для многоуровневых scope.

**Q3: Как ScopedValue наследуется в StructuredTaskScope без копирования?**

> `ScopedValue` хранит привязки в виде иммутабельной linked-list структуры (bindings), а не копируется как `ThreadLocalMap`. При создании дочерней задачи в `StructuredTaskScope.fork()` дочерняя задача получает **ссылку** на тот же `Bindings` объект родителя (O(1)), а не копию. Поскольку bindings иммутабельны, нет гонок данных. При rebinding добавляется новый узел в начало списка, не изменяя старые узлы — предыдущее значение остаётся доступным через pop при выходе из scope.

**Q4: Когда ThreadLocal всё ещё предпочтительнее ScopedValue?**

> `ThreadLocal` остаётся предпочтительным когда: (1) **нужна мутабельность** — поток должен изменять значение в ходе работы (счётчики, аккумуляторы); (2) **нет явного scope** — значение нужно на протяжении всего жизненного цикла потока без ограничений; (3) **совместимость с Java < 21** — ScopedValue доступен только с Java 21; (4) **legacy код** с устоявшейся ThreadLocal-архитектурой, где рефакторинг на ScopedValue нецелесообразен. В новом коде на Java 21+ с виртуальными потоками — `ScopedValue` предпочтительнее для передачи контекста.

**Q5: Что произойдёт если использовать ScopedValue вне scope (без предварительного `where().run()`)?**

> `ScopedValue.get()` вне scope бросает `NoSuchElementException`. Для безопасного использования: `ScopedValue.isBound()` проверяет наличие значения, `orElse(defaultValue)` возвращает дефолт если не bound, `orElseThrow(supplier)` позволяет задать своё исключение. Это разрыв с `ThreadLocal`, который возвращает `null` (или `initialValue()`) по умолчанию — ScopedValue явно сигнализирует о проблеме отсутствующего контекста вместо скрытых NPE.

## 9. Интеграция со Spring Framework

### 9.1. Spring MVC + Virtual Threads + ScopedValue

Spring Boot 3.2+ поддерживает виртуальные потоки для обработки HTTP запросов. `ScopedValue` — правильный способ передачи контекста в этой модели.

```java
// application.properties:
// spring.threads.virtual.enabled=true
// Каждый HTTP запрос теперь обрабатывается виртуальным потоком

// Инфраструктура: перехватчик для установки ScopedValue
@Component
public class ScopedValueHandlerInterceptor implements HandlerInterceptor {

    static final ScopedValue<SecurityContext> SECURITY_CTX = ScopedValue.newInstance();
    static final ScopedValue<RequestMetadata> REQUEST_META = ScopedValue.newInstance();

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        // ПРОБЛЕМА: preHandle нельзя просто "установить" ScopedValue —
        // нужно обернуть весь вызов. Решение — через Filter:
        return true;
    }
}

// ПРАВИЛЬНЫЙ СПОСОБ: Filter обёртывает весь request в ScopedValue scope
@Component
@Order(1)
public class ScopedContextFilter extends OncePerRequestFilter {

    static final ScopedValue<Authentication> AUTHENTICATION = ScopedValue.newInstance();
    static final ScopedValue<String> CORRELATION_ID = ScopedValue.newInstance();
    static final ScopedValue<Locale> REQUEST_LOCALE = ScopedValue.newInstance();

    @Autowired
    private AuthenticationExtractor authExtractor;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        Authentication auth = authExtractor.extract(request);
        String correlationId = Optional.ofNullable(
            request.getHeader("X-Correlation-ID"))
            .orElse(UUID.randomUUID().toString());

        try {
            ScopedValue.where(AUTHENTICATION, auth)
                       .where(CORRELATION_ID, correlationId)
                       .where(REQUEST_LOCALE, request.getLocale())
                       .run(() -> {
                           try {
                               chain.doFilter(request, response);
                           } catch (Exception e) {
                               throw new RuntimeException(e);
                           }
                       });
        } catch (RuntimeException e) {
            if (e.getCause() instanceof ServletException se) throw se;
            if (e.getCause() instanceof IOException ioe) throw ioe;
            throw e;
        }
    }
}
```

### 9.2. Доступ к ScopedValue в Spring компонентах

```java
@Service
public class UserService {

    public UserDto getCurrentUserProfile() {
        // Получаем auth из ScopedValue (установлен Filter-ом выше)
        Authentication auth = ScopedContextFilter.AUTHENTICATION.get();
        String userId = auth.getPrincipal().toString();

        // Используем correlation ID для логирования
        String corrId = ScopedContextFilter.CORRELATION_ID.get();
        log.info("Fetching profile for user={}, correlationId={}", userId, corrId);

        return userRepository.findById(userId)
            .map(UserDto::from)
            .orElseThrow(() -> new UserNotFoundException(userId));
    }
}

// Пример контроллера (никаких SecurityContextHolder!)
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired UserService userService;

    @GetMapping("/me")
    public ResponseEntity<UserDto> getProfile() {
        return ResponseEntity.ok(userService.getCurrentUserProfile());
        // UserService сам читает из ScopedValue — нет параметров!
    }
}
```

### 9.3. Сравнение с SecurityContextHolder (ThreadLocal)

```java
// SPRING SECURITY ТРАДИЦИОННО: ThreadLocal через SecurityContextHolder
@Service
public class TraditionalService {
    public String getCurrentUser() {
        // SecurityContextHolder использует ThreadLocal под капотом
        SecurityContext ctx = SecurityContextHolder.getContext();
        Authentication auth = ctx.getAuthentication();
        return auth.getName();
        // ПРОБЛЕМА: При virtual threads и StructuredTaskScope
        // SecurityContextHolder НЕ передаётся в дочерние потоки!
    }
}

// MODERN WAY: ScopedValue передаётся автоматически в StructuredTaskScope.fork()
@Service
public class ModernService {
    public CompletedProfile buildProfile(String userId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            // ScopedValue.AUTHENTICATION автоматически доступна в fork-ах!
            var orders = scope.fork(() -> orderService.getOrders(userId));
            var prefs = scope.fork(() -> prefsService.getPreferences(
                ScopedContextFilter.AUTHENTICATION.get().getName() // работает!
            ));

            scope.join().throwIfFailed();
            return new CompletedProfile(orders.get(), prefs.get());
        }
    }
}
```

### 9.4. Spring WebFlux + ScopedValue (осторожно!)

```java
// WebFlux использует Reactor с Project Reactor Context (не ThreadLocal!)
// ScopedValue НЕ работает с Reactor Mono/Flux цепочками напрямую!

// WRONG:
@GetMapping("/profile")
public Mono<UserDto> getProfile() {
    return Mono.fromCallable(() -> {
        // ScopedValue.get() может бросить NoSuchElementException!
        // Reactor может перепланировать на другой поток без scope
        Authentication auth = ScopedContextFilter.AUTHENTICATION.get(); // ОПАСНО!
        return userService.getProfile(auth);
    });
}

// ПРАВИЛЬНО для WebFlux: использовать ReactiveSecurityContextHolder
// или передавать контекст явно через Reactor Context:
@GetMapping("/profile")
public Mono<UserDto> getProfileReactive() {
    return ReactiveSecurityContextHolder.getContext()
        .map(SecurityContext::getAuthentication)
        .flatMap(auth -> Mono.fromCallable(
            () -> userService.getProfile(auth)  // auth передан явно
        ));
}

// ScopedValue + WebFlux: только если используете Virtual Thread executor
// spring.webflux.threadpool.type=virtual (Spring Boot 3.3+)
```

### 9.5. Миграция: Spring Security ThreadLocal → ScopedValue

```java
// Шаг 1: Создать ScopedValue holder
public class SecurityScopedValues {
    public static final ScopedValue<Authentication> AUTH = ScopedValue.newInstance();
    public static final ScopedValue<String> TENANT_ID = ScopedValue.newInstance();

    // Утилитный метод для получения с fallback на SecurityContextHolder
    public static Authentication getAuthentication() {
        if (AUTH.isBound()) {
            return AUTH.get();
        }
        // Fallback для non-virtual thread контекстов (legacy код)
        return SecurityContextHolder.getContext().getAuthentication();
    }
}

// Шаг 2: Filter устанавливает и SecurityContextHolder (legacy) и ScopedValue (new)
// Шаг 3: Постепенно мигрировать компоненты на SecurityScopedValues.getAuthentication()
// Шаг 4: После полной миграции убрать SecurityContextHolder
```

## Связанные темы

- [[Процессы и Потоки, Thread, Runnable, состояния потоков]] — виртуальные потоки (Project Loom)
- [[ThreadPool, Future, Callable, Executors, CompletableFuture]] — StructuredTaskScope
- [[Structured Concurrency]] — StructuredTaskScope + ScopedValue синергия
- [[Атомарность операций и Volatile]] — видимость между потоками
