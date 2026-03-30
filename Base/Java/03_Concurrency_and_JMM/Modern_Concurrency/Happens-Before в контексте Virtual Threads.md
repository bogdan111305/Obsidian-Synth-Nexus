# Happens-Before в контексте Virtual Threads

> JMM (Java Memory Model) полностью применяется к Virtual Threads: старт VT happens-before его первое действие, финализация VT happens-before `join()`. При перемонтировании на другой carrier гарантируется видимость всех предыдущих действий VT.

## Связанные темы
[[Модель памяти Java (JMM) и барьеры памяти]], [[Virtual Threads — модель и архитектура]], [[Carrier Threads и Pinning]], [[Атомарность операций и Volatile]]

---

## JMM-гарантии для Virtual Threads

JEP 425 (Virtual Threads) явно специфицирует happens-before отношения:

| Событие | Happens-Before |
|---|---|
| `Thread.startVirtualThread(task)` | Первое действие в `task` |
| Последнее действие VT | `thread.join()` в вызывающем потоке |
| Последнее действие VT | Проверка `thread.isAlive() == false` |
| Действия до `unpark(vt)` | Действия после возобновления VT |

---

## Перемонтирование и видимость памяти

Ключевой вопрос: если VT выполнялся на carrier-1, заблокировался, возобновился на carrier-2 — гарантируется ли видимость данных?

**Ответ: да, гарантируется.**

```java
// Внутренняя механика (упрощённо):
// При демонтировании (park):
//   scheduler.submit(continuation) → volatile write в ForkJoinPool
//
// При монтировании (unpark):
//   carrier читает задачу из очереди → volatile read
//
// volatile read happens-after volatile write →
// всё что VT делал до park, видно после unpark
// даже на другом carrier thread

// Это аналогично:
volatile boolean flag = false;
// Thread A: flag = true (write)
// Thread B: if (flag) ... (read) → видит все изменения до write
```

---

## Практические паттерны

### Shared state между VT — нужна синхронизация

```java
// ПЛОХО: race condition, как и с platform threads
int counter = 0;
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1000; i++) {
        exec.submit(() -> counter++); // не атомарно!
    }
}

// ХОРОШО: атомарный счётчик
AtomicInteger counter = new AtomicInteger();
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1000; i++) {
        exec.submit(counter::incrementAndGet);
    }
}
```

### ThreadLocal в VT — работает, но с оговорками

```java
// ThreadLocal работает: каждый VT имеет свой экземпляр
ThreadLocal<String> requestId = new ThreadLocal<>();

try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    exec.submit(() -> {
        requestId.set("req-123");
        doWork(); // requestId виден здесь
        // При перемонтировании на другой carrier — всё равно виден
    });
}
// requestId.get() после submit — null (разные VT)
```

**Почему ThreadLocal работает при смене carrier:**
- ThreadLocal привязан к `Thread` (VT), не к carrier
- При монтировании carrier копирует ссылку на `Thread.threadLocals` VT
- Смена carrier не влияет на ThreadLocals VT

### ScopedValue — рекомендуемая альтернатива

```java
// ScopedValue: иммутабельный, безопасен при параллельных VT
ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

ScopedValue.where(REQUEST_ID, "req-123").run(() -> {
    doWork();
    // REQUEST_ID.get() == "req-123" в любом child VT, на любом carrier
});
```

---

## Structured Concurrency и Happens-Before

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> f1 = scope.fork(() -> fetchUser());    // VT 1
    Future<String> f2 = scope.fork(() -> fetchOrders());  // VT 2

    scope.join();          // happens-after оба VT завершились
    scope.throwIfFailed();

    // Здесь гарантированно видны результаты обоих VT:
    String user = f1.get();
    String orders = f2.get();
}
// scope.close() → join → happens-before всё что после блока
```

`StructuredTaskScope.join()` создаёт happens-before: завершение дочерних VT → продолжение родительского.

---

## Вопросы на интервью

- Применяется ли JMM к Virtual Threads?
- Что happens-before при старте и завершении Virtual Thread?
- Гарантируется ли видимость данных после перемонтирования VT на другой carrier?
- Нужна ли синхронизация при shared mutable state между VT?
- Чем ScopedValue лучше ThreadLocal в контексте VT?
- Как StructuredTaskScope гарантирует happens-before?

## Подводные камни

- **Нет специальной JMM для VT** — правила те же что для PT. Гонки, видимость, атомарность — всё применяется одинаково.
- **Carrier-local ≠ VT-local** — ThreadLocal привязан к VT (Thread), не к carrier. Смена carrier не нарушает ThreadLocal. Но если разные VT использует один ThreadLocal по имени — это разные значения (как ожидается).
- **`ThreadLocal.remove()` обязателен** — при `newVirtualThreadPerTaskExecutor()` каждый VT новый → ThreadLocal очистится при GC. Но если кто-то сохраняет VT в пуле — риск утечки как у PT.
- **Внутренняя синхронизация через volatile в ForkJoinPool** — это деталь реализации JDK. Спецификация JEP 425 гарантирует happens-before, не механизм реализации.
