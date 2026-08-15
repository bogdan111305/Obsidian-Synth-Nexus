# Исключения в Java

> `Throwable` → `Error` (не обрабатывать) / `Exception` (checked/unchecked). Checked-исключения обязательны к обработке — компилятор проверяет. `try-with-resources` — обязательный паттерн для ресурсов: закрывает ресурсы и сохраняет оба исключения через `addSuppressed`.

## Связанные темы
[[Java Input-Output]], [[Java Serialization and Deserialization]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]], [[Процессы и Потоки, Thread, Runnable, состояния потоков]]

---

## Иерархия исключений

```
Throwable
├── Error                        ← не обрабатывать: JVM-уровень
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception
    ├── Checked (не RuntimeException)  ← компилятор требует обработки
    │   ├── IOException
    │   └── SQLException
    └── RuntimeException               ← unchecked, по усмотрению
        ├── NullPointerException
        ├── IllegalArgumentException
        └── ArrayIndexOutOfBoundsException
```

**Checked vs Unchecked — когда что использовать:**
- **Checked** — на границах системы: I/O, JDBC, внешние API. Вызывающий _обязан_ решить что делать.
- **Unchecked (RuntimeException)** — нарушение контракта: неверный аргумент, нарушение инварианта. Исправляется изменением кода, а не обработкой.
- **Error** — никогда не перехватывать, только `finally` для cleanup.

---

## Базовый синтаксис

```java
public void readFile(String path) throws IOException {
    if (path == null) throw new IllegalArgumentException("path is null"); // unchecked
    // ...
}

try {
    readFile(null);
} catch (NullPointerException | IllegalArgumentException e) {
    // multi-catch (Java 7+): e — effectively final
    log.warn("Bad input: {}", e.getMessage());
} catch (IOException e) {
    throw new ServiceException("IO failed for path=" + path, e); // chain!
} finally {
    // выполняется ВСЕГДА: даже после return в try/catch
    cleanup();
}
```

**Порядок catch:** от специфичного к общему. Поставить `Exception` выше `IOException` → compile error.

---

## try-with-resources (Java 7+)

**Правило:** любой `AutoCloseable` (файлы, соединения, стримы) — только через try-with-resources.

```java
try (InputStream in  = new FileInputStream("a.txt");
     OutputStream out = new FileOutputStream("b.txt")) {
    transfer(in, out);
} // out.close() вызывается первым (LIFO), потом in.close()
```

### Что генерирует javac

Компилятор разворачивает в сложный `try-finally` с синтетической переменной `$primaryExc`:

```java
// Псевдо-bytecode для одного ресурса:
InputStream in = new FileInputStream("a.txt");
Throwable $primaryExc = null;
try {
    transfer(in, out);
} catch (Throwable t) {
    $primaryExc = t;              // запоминаем первичное исключение
    throw t;
} finally {
    if (in != null) {
        if ($primaryExc != null) {
            try { in.close(); }
            catch (Throwable t) {
                $primaryExc.addSuppressed(t); // не теряем исключение close()
            }
        } else {
            in.close(); // если нет первичного — бросаем из close()
        }
    }
}
```

**Ключевой момент:** если основной блок бросил исключение _и_ `close()` бросил исключение — бросается **первичное**, `close()`-исключение _подавляется_ (не теряется, доступно через `getSuppressed()`).

### Suppressed Exceptions

```java
class BrokenResource implements AutoCloseable {
    public void close() throws Exception {
        throw new Exception("close() failed");
    }
}

try (var r = new BrokenResource()) {
    throw new RuntimeException("primary");
}
// Вылетает: RuntimeException("primary")
// Подавлено:  Exception("close() failed") → e.getSuppressed()[0]
```

```java
} catch (RuntimeException e) {
    System.out.println("Primary: " + e.getMessage());
    for (Throwable s : e.getSuppressed()) {
        System.out.println("Suppressed: " + s.getMessage());
    }
}
```

> [!WARNING] Ручной `finally` скрывает исключение `close()`
> ```java
> Resource r = new Resource();
> try { doWork(); }
> finally { r.close(); } // если close() бросит — исключение из doWork() ТЕРЯЕТСЯ
> ```
> `try-with-resources` сохраняет оба через `addSuppressed`. Всегда используй его.

---

## Exception Chaining

**Правило:** при перебросе всегда передавай `cause` — иначе теряешь исходный стек.

```java
// ПЛОХО — оригинальный стек потерян:
} catch (SQLException e) {
    throw new ServiceException("DB failed");
}

// ХОРОШО — сохраняем причину, добавляем контекст:
} catch (SQLException e) {
    throw new ServiceException("DB failed for userId=" + userId, e);
}
```

**Механика `initCause`:**
```java
// Можно вызвать только один раз — иначе IllegalStateException:
ServiceException ex = new ServiceException("DB error");
ex.initCause(cause);
throw ex;

// Внутри Throwable:
// private Throwable cause = this; // this = "cause not set"
// getCause() возвращает null, если cause == this
```

**`InterruptedException` — особый случай:**
```java
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // восстановить флаг прерывания!
    throw new ServiceException("Interrupted", e);
}
```

---

## Производительность: fillInStackTrace

Дорогой шаг при создании исключения — нативный метод `fillInStackTrace()`, обходящий весь стек:

| Операция | Время |
|---|---|
| `new Object()` | ~5 нс |
| `new Exception()` (стек 5 фреймов) | ~500 нс |
| `new Exception()` (стек 50 фреймов) | ~5000 нс |
| `throw/catch` уже созданного | ~50 нс |

### OmitStackTraceInFastThrow

JIT оптимизирует часто бросаемые NPE/AIOOB/ClassCast — начинает бросать singleton-исключение **без stack trace**:

```
java.lang.NullPointerException
    (no stack trace)   ← это JVM оптимизация, а не баг
```

Для воспроизведения с полным стеком: `-XX:-OmitStackTraceInFastThrow`

### FastException Pattern

```java
// Для исключений как сигнал (не ошибка), где stacktrace не нужен:
public class ControlException extends RuntimeException {
    public ControlException(String msg) {
        super(msg, null, true, false); // writableStackTrace = false
    }
}
// Производительность: ~5 нс вместо ~500 нс
// Применение: Vavr ControlThrowable, Scala BreakControl
```

---

## Custom Exceptions в микросервисах

```java
// Базовое для доменных ошибок:
public abstract class DomainException extends RuntimeException {
    private final ErrorCode code;
    private final Map<String, Object> context;

    protected DomainException(ErrorCode code, String message, Map<String, Object> ctx) {
        super(message);
        this.code = code;
        this.context = Map.copyOf(ctx);
    }

    protected DomainException(ErrorCode code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
        this.context = Map.of();
    }
}

// Конкретный тип:
public class UserNotFoundException extends DomainException {
    public UserNotFoundException(long userId) {
        super(ErrorCode.USER_NOT_FOUND, "User not found: " + userId,
              Map.of("userId", userId));
    }
}
```

**Перехват в Spring — `@RestControllerAdvice`:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(DomainException.class)
    public ResponseEntity<ErrorResponse> handleDomain(DomainException ex) {
        return ResponseEntity
            .status(ex.getCode().getHttpStatus())
            .body(new ErrorResponse(ex.getCode(), ex.getMessage(), ex.getContext()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex); // полный стек в лог
        return ResponseEntity.internalServerError()
            .body(new ErrorResponse(ErrorCode.INTERNAL_ERROR, "Internal server error", Map.of()));
    }
}
```

---

## Исключения в многопоточности

Исключение из `Thread.run()` не передаётся в вызывающий поток — оно просто теряется, если не установить `UncaughtExceptionHandler`:

```java
// Через Future:
Future<?> f = executor.submit(() -> { throw new RuntimeException("oops"); });
try {
    f.get();              // ExecutionException оборачивает оригинал
} catch (ExecutionException e) {
    Throwable cause = e.getCause(); // RuntimeException("oops")
}

// Через UncaughtExceptionHandler:
thread.setUncaughtExceptionHandler((t, e) -> log.error("Thread {} died", t.getName(), e));
```

---

## Вопросы на интервью

- Чем Checked отличается от Unchecked? Когда какое использовать?
- Что такое `Error`? Можно ли его поймать? Нужно ли?
- Как работает `try-with-resources` на уровне байткода? Что такое `$primaryExc`?
- Что такое Suppressed Exception? Как получить через API?
- Чем `try-with-resources` лучше ручного `finally` для закрытия ресурсов?
- Почему `throw new Exception()` медленнее `new Object()`? Что такое `fillInStackTrace`?
- Что такое `OmitStackTraceInFastThrow`? Как диагностировать NPE без стека в логах?
- Что такое FastException pattern? В каких случаях оправдан?
- Что нужно сделать с `InterruptedException`, помимо логирования?
- Как правильно проектировать иерархию исключений для REST API?
- Как `@RestControllerAdvice` обрабатывает исключения в Spring?
- Почему нельзя ловить `Throwable` в production-коде?

## Подводные камни

- **Пустой catch** — молчаливо поглощает ошибку, отладка невозможна. Всегда логируй или перебрасывай.
- **Потеря cause при перебросе** — `throw new ServiceException("msg")` без второго аргумента обрезает оригинальный стек. Часы отладки гарантированы.
- **`InterruptedException` без `interrupt()`** — `catch (InterruptedException e) { /* ignore */ }` сбрасывает флаг прерывания, нить не сможет завершиться корректно.
- **Ловим `Exception`, пропуская `RuntimeException`** — `catch (Exception e)` ловит и NPE, и IAE, маскируя баги в коде.
- **Ручной `finally` + `close()`** — если `close()` бросит исключение, исключение из основного блока потеряется. Используй `try-with-resources`.
- **Не лови `Error`** — `OutOfMemoryError` сигнализирует о неисправимом состоянии JVM. Поймаешь — ситуация только ухудшится.
- **Производительность в hot path** — создание `new Exception()` стоит ~500 нс–5 мкс в зависимости от глубины стека. Не используй исключения для control flow.
- **`OmitStackTraceInFastThrow` в production** — NPE/AIOOB могут логироваться без стека. Для диагностики добавь `-XX:-OmitStackTraceInFastThrow`, воспроизведи, убери.
