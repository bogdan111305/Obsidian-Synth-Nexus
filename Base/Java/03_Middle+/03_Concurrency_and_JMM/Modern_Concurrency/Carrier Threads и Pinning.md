# Carrier Threads и Pinning

> **Carrier thread** — платформенный поток, исполняющий Virtual Thread. **Pinning** — ситуация, когда VT не может демонтироваться с carrier при блокировке: carrier блокируется вместе с VT, аннулируя выгоды Virtual Threads.

## Связанные темы
[[Virtual Threads — модель и архитектура]], [[Virtual Threads vs Platform Threads]], [[Атомарность операций и Volatile]], [[Java Monitor]]

---

## Что такое Carrier Thread

```
ForkJoinPool (scheduler)
├── carrier-0 (PT) → выполняет VT-42
├── carrier-1 (PT) → выполняет VT-7
├── carrier-2 (PT) → выполняет VT-1001
└── carrier-3 (PT) → свободен

При I/O в VT-42:
├── VT-42 демонтируется → carrier-0 свободен
├── carrier-0 берёт VT-999 из очереди
└── VT-42 возобновится позже на любом carrier
```

По умолчанию: `parallelism = availableProcessors()`, maxPoolSize = 256.

---

## Что такое Pinning

Pinning — VT "приклеен" к carrier и не может демонтироваться. Carrier блокируется вместе с VT:

```
При pinning:
├── VT-42 ждёт I/O (100ms)
├── carrier-0 заблокирован вместе с VT-42
├── carrier-0 НЕ может выполнять другие VT
└── если все carriers заняты pinned VT → дедлок!
```

### Причины Pinning

**1. `synchronized` блок/метод с блокирующей операцией:**

```java
// ПЛОХО — вызывает pinning:
synchronized (lock) {
    Thread.sleep(100);    // VT хочет демонтироваться, но не может!
    connection.query();   // то же самое — VT заблокирован внутри synchronized
}

// ХОРОШО — VT может демонтироваться:
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    Thread.sleep(100);   // VT демонтируется, carrier свободен
    connection.query();
} finally {
    lock.unlock();
}
```

**2. `Object.wait()` / `notify()` внутри `synchronized`:**

```java
// ПЛОХО:
synchronized (monitor) {
    while (!condition) {
        monitor.wait(); // pinning!
    }
}

// ХОРОШО:
Condition condition = lock.newCondition();
lock.lock();
try {
    while (!ready) {
        condition.await(); // VT может демонтироваться
    }
} finally {
    lock.unlock();
}
```

**3. Native методы (JNI):**

```java
// Любой native метод в стеке вызовов → pinning до завершения JNI
// Нельзя исправить — ограничение JVM
```

**4. Финализаторы (устаревший механизм):**

```java
// @Override protected void finalize() — pinning при вызове
// Решение: использовать Cleaner API вместо finalize()
```

---

## Диагностика Pinning

```bash
# JVM флаг — логировать все события pinning:
-Djdk.tracePinnedThreads=full

# Вывод при pinning:
# Thread[#21,ForkJoinPool-1-worker-1,5,CarrierThreads]
#     java.base/java.lang.VirtualThread$PinnedEvent.emitPinnedEvent(VT.java:181)
#     java.base/java.lang.Object.wait0(Native Method)          ← причина
#     com.example.MyService.process(MyService.java:42)

# Краткий вывод (только методы):
-Djdk.tracePinnedThreads=short

# JFR событие:
# jdk.VirtualThreadPinned — записывается когда VT pinned > threshold (20ms по умолчанию)
```

```java
// Программная диагностика через JFR:
RecordingConfiguration config = new RecordingConfiguration.Builder()
    .enable("jdk.VirtualThreadPinned")
    .withThreshold("jdk.VirtualThreadPinned", Duration.ofMillis(10))
    .build();
```

---

## Последствия Pinning

```
Симптомы:
1. Throughput не растёт при увеличении числа VT
2. Все carrier threads заняты → новые VT ждут в очереди
3. При 100% pinning + I/O → поведение как у platform thread pool
4. В крайнем случае: дедлок (все carriers заняты, новые задачи не выполняются)

JVM пытается создать новый carrier (сверх parallelism) при долгом pinning,
но ограничен maxPoolSize (256 по умолчанию).
```

---

## Стратегии исправления Pinning

| Причина | Решение |
|---|---|
| `synchronized` + I/O | Заменить на `ReentrantLock` |
| `Object.wait()` | Заменить на `Condition.await()` |
| JDBC-драйвер с `synchronized` | Мигрировать на R2DBC или ждать обновления драйвера |
| Legacy `synchronized` в библиотеке | Изолировать в отдельный PT pool |
| JNI | Нельзя исправить; запускать в PT |

**JDBC и Pinning — важный нюанс:**

```java
// PostgreSQL JDBC (до pgjdbc 42.7.x) использует synchronized внутри
// → каждый SQL запрос в VT вызывает pinning!

// Решение 1: R2DBC (реактивный драйвер)
// Решение 2: изолировать JDBC в отдельный PT executor:
ExecutorService jdbcExecutor = Executors.newFixedThreadPool(50);
// выполнять JDBC запросы через jdbcExecutor.submit(...)
// а остальной код — через VT
```

---

## Вопросы на интервью

- Что такое pinning? Почему `synchronized` его вызывает?
- Чем отличается `synchronized` от `ReentrantLock` в контексте VT?
- Как диагностировать pinning в production?
- Что происходит с carrier thread при pinning?
- Почему JDBC-код может вызывать pinning даже без явного `synchronized`?
- Что делает JVM при исчерпании carrier threads из-за pinning?

## Подводные камни

- **JDBC + VT = pinning** — большинство JDBC-драйверов используют `synchronized` внутри. Без осознания этого миграция на VT не даст выигрыша для DB-intensive кода.
- **Spring `@Transactional` + VT** — `@Transactional` держит соединение на протяжении метода. Если метод блокирующий + JDBC с pinning → carrier заблокирован всё время транзакции.
- **JVM создаёт компенсирующие carrier** — при долгом pinning JVM создаёт дополнительные carrier (до maxPoolSize=256). Это маскирует проблему, но не решает — 256 заблокированных PT = старый мир.
- **`-Djdk.tracePinnedThreads` имеет overhead** — не включай в production постоянно. Только для диагностики.
- **`Object.wait()` без pinning в Java 24+** — начиная с Java 24 JEP 491 улучшает ситуацию: `synchronized` методы/блоки больше не вызывают pinning (preview в Java 23). Проверяй версию JDK.
