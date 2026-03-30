# Virtual Threads vs Platform Threads

> Platform Thread = OS thread (1:1, ~1MB стека, ~тысячи). Virtual Thread = JVM thread (M:N, ~KB стека, ~миллионы). Выбор: VT для I/O-bound, PT для CPU-bound и кода с pinning.

## Связанные темы
[[Virtual Threads — модель и архитектура]], [[Carrier Threads и Pinning]], [[Процессы и Потоки, Thread, Runnable, состояния потоков]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]]

---

## Сравнение

| Характеристика | Platform Thread | Virtual Thread |
|---|---|---|
| Маппинг на ОС | 1:1 (один OS thread) | M:N (многие на немногие carrier) |
| Размер стека | ~512KB–1MB (фиксированный) | ~KB, растёт динамически в heap |
| Создание | ~1ms, дорого | ~1µs, дёшево |
| Максимум | ~тысячи (лимит ОС) | ~миллионы |
| Блокировка I/O | Блокирует OS thread | Демонтируется, carrier свободен |
| CPU-bound | Эффективно | Так же (carrier = PT) |
| `synchronized` | Нормально | Вызывает pinning! |
| ThreadLocal | Нормально | Работает, но дорого при 1M VT |
| `Thread.sleep()` | Блокирует OS thread | Демонтирует VT, carrier свободен |
| Приоритет | Поддерживается | Игнорируется |
| Daemon | Настраивается | Всегда daemon |

---

## Когда что использовать

**Virtual Threads — подходят:**
- I/O-bound сервисы: HTTP запросы, JDBC, файлы, Redis
- Высококонкурентные серверы (тысячи одновременных запросов)
- Замена пула потоков для blocking-код с `Executors.newVirtualThreadPerTaskExecutor()`
- Простой код без `synchronized`/`Object.wait()`

**Platform Threads — подходят:**
- CPU-intensive вычисления (матрицы, криптография, парсинг)
- Код с `synchronized` на внешних ресурсах (JDBC-драйверы, legacy)
- Когда нужен точный контроль над числом потоков
- Код с `ThreadLocal` при большой стоимости per-thread данных

---

## Бенчмарк: throughput при I/O-bound нагрузке

```java
// Имитация 10_000 конкурентных запросов с задержкой 100ms

// Platform Threads (pool 200):
// При 10_000 запросов → очередь → ~50 batch → 50 * 100ms = 5сек

// Virtual Threads (per-task):
// 10_000 VT стартуют сразу → все блокируются на I/O →
// carriers свободны → ~100ms суммарно

// Реальные числа (Spring Boot, wrk benchmark):
// PT (200 threads): ~1000 RPS
// VT: ~10_000 RPS при той же latency
```

---

## Миграция с Platform Threads

**Шаг 1 — заменить executor:**

```java
// Было:
ExecutorService executor = Executors.newFixedThreadPool(200);

// Стало:
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

**Шаг 2 — Spring Boot 3.2+ (автоматически):**

```yaml
# application.yaml
spring:
  threads:
    virtual:
      enabled: true
# Это включает VT для Tomcat, @Async, TaskExecutor
```

**Шаг 3 — проверить на pinning:**

```bash
-Djdk.tracePinnedThreads=full
```

Смотреть в логах: `VirtualThread... pinned at ...`

**Шаг 4 — заменить `synchronized` на `ReentrantLock`:**

```java
// Было (вызывает pinning):
synchronized (resource) { doIO(); }

// Стало (VT может демонтироваться внутри lock):
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try { doIO(); }
finally { lock.unlock(); }
```

---

## Вопросы на интервью

- Чем Virtual Thread отличается от Platform Thread по модели памяти и планированию?
- Когда VT не дают выигрыша в производительности?
- Как мигрировать Spring Boot приложение на Virtual Threads?
- Почему `synchronized` проблематичен с VT?
- Почему нельзя задать приоритет VT?
- Зачем VT всегда daemon?

## Подводные камни

- **VT не ускоряют CPU-bound** — параллелизм ограничен числом carrier (= CPU). 1M VT + CPU-heavy код = те же N параллельных операций.
- **`newVirtualThreadPerTaskExecutor` — не пул** — создаёт новый VT для каждой задачи. Нет очереди задач как в `ThreadPoolExecutor`. Для rate limiting нужен `Semaphore`.
- **ThreadLocal + 1M VT = утечка памяти** — если каждый VT создаёт большие ThreadLocal-значения (connection, buffer), это 1M объектов одновременно. Переходи на `ScopedValue`.
- **Нельзя interrupt через `shutdownNow()`** — VT прерываются через `interrupt()`, но если код игнорирует `InterruptedException`, VT не остановится.
