# Прерывание потока в Java

> Прерывание — **кооперативный** механизм: `thread.interrupt()` только устанавливает флаг. Поток сам проверяет `isInterrupted()` или ловит `InterruptedException`. `Thread.stop()` удалён — небезопасен.
> На интервью: разница isInterrupted/Thread.interrupted, что происходит при interrupt в NIO, как устроен graceful shutdown, virtual threads и прерывания.

## Связанные темы
[[Процессы и Потоки, Thread, Runnable, состояния потоков]], [[Java Monitor]], [[Lock]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]]

---

## Механизм прерывания

```java
thread.interrupt();                          // установить флаг прерывания

Thread.currentThread().isInterrupted();      // проверить флаг (НЕ сбрасывает)
Thread.interrupted();                        // проверить И СБРОСИТЬ флаг (static!)

// Блокирующие методы (sleep, wait, join, lockInterruptibly):
// → при установленном флаге бросают InterruptedException И сбрасывают флаг
```

**JVM-реализация:** флаг хранится в OS-структуре потока. `interrupt()` вызывает нативный `interrupt0()`, который:
1. Устанавливает флаг
2. Если поток заблокирован на `Interruptible` (NIO) — вызывает `blocker.interrupt(thread)`
3. `LockSupport.unpark(thread)` — разбудить если паркован

---

## isInterrupted() vs Thread.interrupted()

Самая частая путаница на интервью:

```java
// isInterrupted() — INSTANCE метод, флаг НЕ сбрасывается:
thread.isInterrupted();                    // проверяет флаг потока thread
Thread.currentThread().isInterrupted();   // проверяет флаг текущего потока

// Thread.interrupted() — STATIC метод, флаг СБРАСЫВАЕТСЯ:
boolean was = Thread.interrupted(); // читает И сбрасывает → флаг становится false
```

**Ловушка: случайная кража флага**
```java
// Фреймворк-код "под капотом":
void processTask() {
    if (Thread.interrupted()) { // СБРОСИЛ флаг прерывания!
        throw new RuntimeException("interrupted");
    }
}

// Твой код:
thread.interrupt();
framework.processTask(); // ← украл флаг
while (!Thread.currentThread().isInterrupted()) { // ← не выйдет никогда!
    ...
}
```

> [!WARNING] Правило: `Thread.interrupted()` только когда хочешь ЯВНО сбросить флаг и обработать прерывание сам. Для проверки без сброса — `isInterrupted()`.

---

## Обработка InterruptedException

```java
// ПРАВИЛЬНО — восстановить флаг:
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // восстановить флаг!
    // освободить ресурсы, выйти из цикла
}

// ПРАВИЛЬНО — пробросить выше (если метод объявляет throws):
void doWork() throws InterruptedException {
    Thread.sleep(1000); // пусть вызывающий обработает
}

// НЕПРАВИЛЬНО — глотать молча:
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // ignore — флаг сброшен, поток никогда не узнает что его прерывали!
}
```

**Полный паттерн: interrupt-aware задача с cleanup**
```java
public class GracefulTask implements Runnable {

    @Override
    public void run() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                processOneItem();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // восстановить для вызывающего
        } finally {
            cleanup(); // всегда выполнится — освободить ресурсы
        }
    }

    private void processOneItem() throws InterruptedException {
        blockingCall(); // бросит InterruptedException при флаге
    }
}
```

---

## shutdownNow() и ExecutorService

```java
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();
exec.submit(new GracefulTask());
exec.shutdown(); // ждём завершения текущих задач

try {
    if (!exec.awaitTermination(5, TimeUnit.SECONDS)) {
        exec.shutdownNow(); // interrupt() всем активным задачам
        exec.awaitTermination(1, TimeUnit.SECONDS);
    }
} catch (InterruptedException e) {
    exec.shutdownNow();
    Thread.currentThread().interrupt();
}
```

`shutdown()` — graceful: задачи завершатся сами.
`shutdownNow()` — interrupt всем, возвращает список незапущенных задач.

---

## interrupt() и NIO: ClosedByInterruptException

`Thread.sleep()` / `wait()` → `InterruptedException`. Но NIO-каналы (`SocketChannel`, `FileChannel`) реализуют `InterruptibleChannel` — при прерывании потока канал **закрывается**:

```java
// java.io (blocking): interrupt НЕ прерывает блокировку на accept()/read()
// Нужно явно закрыть сокет снаружи

// java.nio: interrupt закрывает канал!
SocketChannel channel = SocketChannel.open();
channel.configureBlocking(true);

Thread nioThread = new Thread(() -> {
    try {
        ByteBuffer buf = ByteBuffer.allocate(1024);
        channel.read(buf);              // блокируется
    } catch (ClosedByInterruptException e) {
        // канал закрыт! переиспользовать нельзя
    } catch (IOException e) { ... }
});

nioThread.start();
nioThread.interrupt(); // → ClosedByInterruptException + channel.close()
```

**Цепочка при interrupt() на NIO:**
```
Thread.interrupt() →
  1. Установить флаг
  2. blocker != null → blocker.interrupt(thread) → channel.close()
  3. LockSupport.unpark(thread) — разбудить если паркован
```

---

## Прерывание и виртуальные потоки

```java
Thread vt = Thread.ofVirtual().start(() -> {
    try {
        Thread.sleep(Duration.ofSeconds(10));
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        System.out.println("Virtual thread interrupted");
        // Carrier thread НЕ прерывается — только виртуальный
    }
});
vt.interrupt(); // прерывает виртуальный поток, не carrier
```

**Virtual thread + blocking I/O:** JVM переводит в NIO под капотом, паркует виртуальный поток (не carrier). При `interrupt()` — виртуальный поток разпаркуется, carrier продолжает других.

**Virtual thread + пиннинг:**
```java
Thread vt = Thread.ofVirtual().start(() -> {
    synchronized (lock) {   // пиннинг carrier thread до выхода из блока
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            // carrier thread НЕ освободится пока не выйдем из synchronized
        }
    }
});
// Java 24 (JEP 491): synchronized больше не пиннирует carrier thread
```

---

## Устаревшие методы

| Метод | Причина устаревания |
|-------|---------------------|
| `stop()` | Принудительно убивает поток, мониторы не освобождаются → inconsistent state |
| `suspend()` | Останавливает поток, удерживая мониторы → deadlock |
| `resume()` | Парный к suspend() — оба небезопасны |

Решение для всех случаев: кооперативный interrupt + флаг.

---

## Вопросы на интервью

- Что делает `thread.interrupt()`? Останавливает ли он поток?
- Чем `isInterrupted()` отличается от `Thread.interrupted()`? Что опасного в последнем?
- Почему нельзя глотать `InterruptedException` молча?
- Что происходит при `interrupt()` на потоке заблокированном в NIO channel read?
- Чем `ClosedByInterruptException` отличается от `InterruptedException`?
- Чем `shutdown()` отличается от `shutdownNow()`?
- Как виртуальные потоки реагируют на прерывание? Прерывается ли carrier thread?
- Прерывает ли `interrupt()` поток, ожидающий монитора (`synchronized`)? (Нет — только WAITING, не BLOCKED)

---

## Подводные камни

- **Проглоченный InterruptedException** — пустой catch сбрасывает флаг, поток не знает о прерывании. Всегда: `Thread.currentThread().interrupt()` или пробрасывай выше.
- **`Thread.interrupted()` вместо `isInterrupted()`** — сбрасывает флаг. Библиотечный код может "украсть" прерывание.
- **`interrupt()` не прерывает BLOCKED** — поток, ждущий монитора (`synchronized`), игнорирует interrupt. Только `WAITING`/`TIMED_WAITING` реагируют через `InterruptedException`. Используй `lockInterruptibly()` для прерываемого ожидания.
- **NIO channel закрывается** — `ClosedByInterruptException` → канал нельзя переиспользовать. Если нужно переиспользовать — не прерывай поток; закрывай канал явно из другого потока.
- **Volatile flag vs interrupt** — `volatile boolean running = false` не прерывает `Thread.sleep()`. Для прерывания блокирующих вызовов нужен `interrupt()`.
- **`synchronized` + virtual thread** — пиннинг: carrier thread не освободится на время удержания монитора. Используй `ReentrantLock` (решено в Java 24, JEP 491).
