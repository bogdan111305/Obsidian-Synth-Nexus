---
title: "Прерывание потока в Java"
tags: [java, concurrency, thread, interrupt, nio, virtual-threads]
updated: 2026-03-11
---

# Прерывание потока в Java

> [!QUOTE] Суть
> Прерывание — **кооперативный** механизм: `thread.interrupt()` только устанавливает флаг. Поток сам обязан проверять `isInterrupted()` или ловить `InterruptedException`. `Thread.stop()` — удалён (небезопасен). Никогда не глотай `InterruptedException` молча — восстанови флаг!

> [!WARNING] Ловушка: проглоченный InterruptedException
> `catch (InterruptedException e) { /* ignore */ }` — сбрасывает флаг прерывания! Поток больше не узнает что его прерывали. Всегда делай: `Thread.currentThread().interrupt()` внутри catch чтобы восстановить флаг.

**Прерывание потоков** — это кооперативный механизм в Java, позволяющий сигнализировать потоку о необходимости остановить или изменить свою работу. Потоки должны самостоятельно проверять флаг прерывания и корректно реагировать.

## 1. Прерывание потоков

**Прерывание** — это способ уведомить поток о необходимости завершить или изменить выполнение. В Java это кооперативный механизм: поток сам решает, как реагировать на сигнал прерывания.

- **Особенности**:
    - Прерывание не завершает поток принудительно.
    - Поток проверяет флаг прерывания или обрабатывает `InterruptedException`.
    - Используется для корректного завершения задач, освобождения ресурсов и управления многопоточностью.

## 2. Методы для прерывания

### 2.1. `Thread.interrupt()`

- Устанавливает **флаг прерывания** для потока.
- **Поведение**:
    - Если поток находится в **блокирующем состоянии** (`sleep()`, `wait()`, `join()`), выбрасывается `InterruptedException`, а флаг сбрасывается.
    - Если поток активен, флаг просто устанавливается.

**Пример**:

```java
Thread thread = new Thread(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        System.out.println("Прерван во время сна");
    }
});
thread.start();
thread.interrupt(); // Выбрасывает InterruptedException
```

### 2.2. Проверка флага прерывания

- `Thread.currentThread().isInterrupted()`: Возвращает состояние флага, не сбрасывая его.
- `Thread.interrupted()`: Статический метод, возвращает состояние флага и сбрасывает его.

**Пример**:

```java
public void run() {
    while (!Thread.currentThread().isInterrupted()) {
        // Работа
    }
    System.out.println("Поток прерван");
}
```

### 2.3. Обработка `InterruptedException`

- Блокирующие методы (`sleep()`, `wait()`, `join()`) выбрасывают `InterruptedException`.
- **Правильная реакция**:
    - Восстановить флаг прерывания.
    - Завершить работу или выполнить очистку.

**Пример**:

```java
public void run() {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // Восстанавливаем флаг
            break; // Выходим из цикла
        }
    }
    System.out.println("Поток прерван и завершён");
}
```

## 3. Безопасное завершение потоков

Безопасное завершение потоков — это процесс, позволяющий потоку корректно завершить работу, освободить ресурсы и сохранить консистентность данных.

### 3.1. Почему `stop()`, `suspend()`, `resume()` устарели?

- **Метод `stop()`**:
    - Принудительно завершает поток, игнорируя его состояние.
    - Может оставить объекты в неконсистентном состоянии (например, частично записанные данные).
    - **Deprecated** с Java 2 из-за небезопасности.
- **Методы `suspend()` и `resume()`**:
    - `suspend()` останавливает поток, не освобождая мониторы (`synchronized`).
    - Может вызвать **deadlock**, если другие потоки ждут освобождения ресурсов.
    - **Deprecated** из-за риска блокировки.
- **Решение**: Используйте кооперативные механизмы (`interrupt()`, флаги).

### 3.2. Использование `interrupt()` для завершения

- Поток проверяет флаг прерывания и завершает работу.
- Позволяет выполнить очистку ресурсов.

**Пример**:

```java
public class SafeStopExample implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                System.out.println("Работаю...");
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); // Восстанавливаем флаг
                break;
            }
        }
        System.out.println("Поток корректно завершён");
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new SafeStopExample());
        thread.start();
        Thread.sleep(2000);
        thread.interrupt();
    }
}
```

### 3.3. Использование флага-состояния

- `volatile` флаг для управления состоянием потока.

**Пример**:

```java
public class FlagStopExample implements Runnable {
    private volatile boolean running = true;

    @Override
    public void run() {
        while (running) {
            System.out.println("Работаю...");
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
        System.out.println("Поток завершён");
    }

    public void stop() {
        running = false;
    }

    public static void main(String[] args) throws InterruptedException {
        FlagStopExample task = new FlagStopExample();
        Thread thread = new Thread(task);
        thread.start();
        Thread.sleep(2000);
        task.stop();
    }
}
```

### 3.4. Использование `ExecutorService`

- `ExecutorService` управляет пулом потоков и их завершением.

**Пример**:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorStopExample {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        executor.submit(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                System.out.println("Работаю...");
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
            System.out.println("Поток завершён");
        });
        Thread.sleep(2000);
        executor.shutdownNow(); // Прерывает все задачи
    }
}
```

**Особенности**:

- `shutdown()`: Завершает задачи после их выполнения.
- `shutdownNow()`: Прерывает все задачи, возвращая незавершённые.

## 4. JVM-реализация

- **Флаг прерывания**:
    - Хранится в структуре потока ОС.
    - `Thread.interrupt()` вызывает нативный метод (`Thread.interrupt0()`).
- **Байт-код**:
    
    ```java
    invokestatic java/lang/Thread.interrupt()V
    // Вызывает native interrupt0()
    ```
    
- **InterruptedException**:
    
    - Блокирующие методы (`sleep()`, `wait()`) проверяют флаг прерывания.
    - Если флаг установлен, JVM выбрасывает `InterruptedException` и сбрасывает флаг.
    
    ```java
    invokestatic java/lang/Thread.sleep(J)V
    // Может выбросить InterruptedException
    ```
    

## 5. Практическое применение

- **Java EE / Jakarta EE**:
    - Прерывание задач в сервлетах или EJB для обработки таймаутов.
- **Spring**:
    - `@Async` методы с поддержкой прерывания через `TaskExecutor`.
- **GUI**:
    - Прерывание фоновых задач в Swing/JavaFX для отзывчивости интерфейса.
- **Серверы**:
    - Завершение потоков при остановке сервера (например, Tomcat, Jetty).

**Пример (Spring)**:

```java
@Service
public class AsyncService {
    @Async
    public CompletableFuture<String> process() throws InterruptedException {
        while (!Thread.currentThread().isInterrupted()) {
            Thread.sleep(500);
            System.out.println("Работаю...");
        }
        return CompletableFuture.completedFuture("Завершён");
    }
}
```

## 6. Современные возможности (Java 21+)

### 6.1. Виртуальные потоки (Project Loom)

Виртуальные потоки поддерживают прерывания аналогично платформенным потокам.

**Пример**:

```java
public class VirtualThreadInterrupt {
    public static void main(String[] args) throws InterruptedException {
        Thread virtualThread = Thread.ofVirtual().start(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    System.out.println("Виртуальный поток работает...");
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.out.println("Виртуальный поток прерван");
            }
        });
        Thread.sleep(2000);
        virtualThread.interrupt();
    }
}
```

**Особенности**:

- Виртуальные потоки легче прерывать, так как они управляются JVM.
- Подходят для I/O-интенсивных задач.

### 6.2. StructuredTaskScope

`StructuredTaskScope` (Java 21, JEP 453) упрощает управление прерываниями в группе задач.

**Пример**:

```java
import java.util.concurrent.StructuredTaskScope;

public class StructuredInterrupt {
    public static void main(String[] args) throws Exception {
        try (var scope = new StructuredTaskScope<String>()) {
            var task = scope.fork(() -> {
                try {
                    Thread.sleep(1000);
                    return "Задача выполнена";
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    throw e;
                }
            });
            Thread.sleep(500);
            scope.shutdown(); // Прерывает задачи
            scope.join();
        }
    }
}
```

**Преимущества**:

- Автоматическое прерывание всех задач в области.
- Упрощённая обработка ошибок.

## 7. Подводные камни

1. **Игнорирование `InterruptedException`**:
    
    ```java
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        // Пустой catch игнорирует прерывание
    }
    ```
    
    **Решение**: Восстанавливайте флаг:
    
    ```java
    Thread.currentThread().interrupt();
    ```
    
2. **Неправильное использование `Thread.interrupted()`**:
    
    - Сбрасывает флаг, что может скрыть прерывание.
    - **Решение**: Используйте `isInterrupted()` для проверки без сброса.
3. **Незавершённые ресурсы**:
    
    - Поток, игнорирующий прерывание, может удерживать ресурсы.
    - **Решение**: Всегда освобождайте ресурсы в `catch` или `finally`.
4. **Виртуальные потоки**:
    
    - Долгие операции в виртуальных потоках могут блокировать другие.
    - **Решение**: Используйте платформенные потоки для CPU-интенсивных задач.

## 8. Производительность

- **Прерывания**:
    - Накладные расходы минимальны (проверка флага, нативный вызов).
    - `InterruptedException` добавляет затраты на обработку исключений.
- **ExecutorService**:
    - `shutdownNow()` эффективнее для массового прерывания.
- **Виртуальные потоки**:
    - Прерывания быстрее, так как не зависят от ОС.
- **Рекомендации**:
    - Минимизируйте блокирующие операции.
    - Используйте `StructuredTaskScope` для управления группами задач.

## 9. Плюсы и минусы прерываний

|Плюсы|Минусы|
|---|---|
|✅ Кооперативный механизм|❌ Требует явной обработки в коде|
|✅ Безопасное завершение|❌ Не завершает поток принудительно|
|✅ Поддержка в виртуальных потоках|❌ Риск игнорирования исключений|

## 10. Лучшие практики

1. **Обрабатывайте `InterruptedException`**:
    
    ```java
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        // Освободить ресурсы
    }
    ```
    
2. **Используйте `isInterrupted()` для проверки**:
    
    ```java
    while (!Thread.currentThread().isInterrupted()) { ... }
    ```
    
3. **Применяйте `ExecutorService`**:
    
    ```java
    executor.shutdownNow();
    ```
    
4. **Используйте виртуальные потоки для I/O**:
    
    ```java
    Thread.ofVirtual().start(() -> {});
    ```
    
5. **Тестируйте прерывания**:
    
    ```java
    @Test
    void testInterrupt() throws InterruptedException {
        Thread thread = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                assertTrue(Thread.currentThread().isInterrupted());
            }
        });
        thread.start();
        thread.interrupt();
        thread.join();
    }
    ```
    
6. **Освобождайте ресурсы**:
    
    ```java
    finally {
        // Закрыть файлы, соединения
    }
    ```
    

## 11. Заключение

Прерывание потоков в Java — кооперативный механизм, позволяющий безопасно завершить или изменить работу потока. Методы `interrupt()`, `isInterrupted()` и `interrupted()` вместе с обработкой `InterruptedException` обеспечивают гибкое управление. Устаревшие методы (`stop()`, `suspend()`, `resume()`) небезопасны и не используются. Современные возможности (виртуальные потоки, `StructuredTaskScope`) упрощают управление прерываниями. Понимание JVM-реализации, правильная обработка исключений и следование лучшим практикам позволяют создавать надёжные многопоточные приложения.

---

## Senior Insights

### S1. isInterrupted() vs Thread.interrupted() — критическая разница

Самая частая ошибка — смешать эти два метода:

```java
// isInterrupted() — INSTANCE метод, НЕ сбрасывает флаг:
thread.isInterrupted();                     // проверяет флаг потока thread
Thread.currentThread().isInterrupted();     // проверяет флаг текущего потока
// Флаг остаётся установленным!

// Thread.interrupted() — STATIC метод, СБРАСЫВАЕТ флаг:
boolean wasInterrupted = Thread.interrupted(); // читает И сбрасывает флаг
// После вызова: флаг = false, даже если был true
```

**Классическая ошибка — случайный сброс флага:**
```java
// Код библиотеки (например, фреймворк):
void processTask() {
    if (Thread.interrupted()) { // СБРОСИЛ флаг!
        throw new InterruptedException();
    }
    // теперь другой код думает что прерывания не было
}

// Твой код:
thread.interrupt();
processTask(); // ← может украсть флаг прерывания
while (!Thread.currentThread().isInterrupted()) { // ← никогда не выйдет!
    ...
}
```

> [!WARNING] Правило: `Thread.interrupted()` используй только когда хочешь ЯВНО сбросить флаг и обработать прерывание сам. В остальных случаях — `isInterrupted()`.

### S2. interrupt() и NIO: ClosedByInterruptException

`Thread.sleep()` / `Object.wait()` бросают `InterruptedException`. Но NIO-каналы (`SocketChannel`, `FileChannel`) реализуют `InterruptibleChannel` — при прерывании потока канал **закрывается**:

```java
// Blocking I/O (java.io) — interrupt прерывает через исключение:
ServerSocket serverSocket = new ServerSocket(8080);
Thread ioThread = new Thread(() -> {
    try {
        Socket conn = serverSocket.accept(); // блокируется
    } catch (IOException e) {
        // НЕ прерывается через interrupt()!
        // Нужно явно закрыть serverSocket снаружи
    }
});

// NIO (java.nio) — interrupt закрывает канал!
SocketChannel channel = SocketChannel.open();
channel.configureBlocking(true);

Thread nioThread = new Thread(() -> {
    try {
        ByteBuffer buf = ByteBuffer.allocate(1024);
        channel.read(buf); // блокируется
    } catch (ClosedByInterruptException e) {
        // ← бросается при interrupt()
        // Канал закрыт! Нельзя переиспользовать
        System.out.println("Channel closed by interrupt");
    } catch (IOException e) { ... }
});

nioThread.start();
nioThread.interrupt(); // → ClosedByInterruptException + channel закрыт
```

**Механизм:** `InterruptibleChannel` регистрирует поток через `Interruptible` объект. При `thread.interrupt()` JVM проверяет: есть ли зарегистрированный `Interruptible`? Если да — вызывает `blocker.interrupt(thread)` → `channel.close()`.

```
Thread.interrupt() →
  1. Установить флаг
  2. if (blocker != null): blocker.interrupt(this)  // закрыть канал
  3. LockSupport.unpark(this)  // разбудить если паркован
```

### S3. Прерывание и виртуальные потоки

Виртуальные потоки (Java 21+) прерываются **так же**, но есть важные отличия:

```java
Thread vt = Thread.ofVirtual().start(() -> {
    try {
        // Виртуальный поток припаркован на carrier thread
        Thread.sleep(Duration.ofSeconds(10));
    } catch (InterruptedException e) {
        // Carrier thread не прерывается — только виртуальный
        Thread.currentThread().interrupt();
        System.out.println("Virtual thread interrupted");
    }
});

vt.interrupt(); // прерывает виртуальный поток, не carrier
```

**Виртуальный поток + blocking I/O:**
```java
// При вызове blocking socket read на виртуальном потоке:
// JVM автоматически переводит в NIO под капотом
// и паркует виртуальный поток (не carrier thread)
// При interrupt(): виртуальный поток разпаркуется + InterruptedException
// Carrier thread НЕ затронут — продолжает выполнять других
```

**Виртуальный поток + пиннинг (synchronized блок):**
```java
Thread vt = Thread.ofVirtual().start(() -> {
    synchronized (lock) {  // ← пиннинг carrier thread
        try {
            Thread.sleep(1000); // виртуальный поток пиннирован
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            // Carrier thread НЕ освободится пока не выйдем из synchronized
        }
    }
});
// Избегай синхронизированных блоков в виртуальных потоках!
// Java 24 (JEP 491): synchronized больше не пиннирует carrier thread
```

### S4. Правильный паттерн: interrupt-aware задача

```java
// Полный корректный паттерн для задачи с cleanup:
public class GracefulTask implements Runnable {

    @Override
    public void run() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                processOneItem();
            }
        } catch (InterruptedException e) {
            // Восстановить флаг — пусть вызывающий код тоже знает
            Thread.currentThread().interrupt();
        } finally {
            cleanup(); // всегда выполнится
        }
        System.out.println("Task gracefully stopped");
    }

    private void processOneItem() throws InterruptedException {
        // I/O операция — может бросить InterruptedException
        SomeBlockingAPI.call();
    }

    private void cleanup() {
        // закрыть ресурсы, flush буферы, etc.
    }
}

// ExecutorService + awaiting termination:
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();
exec.submit(new GracefulTask());
exec.shutdown();
try {
    if (!exec.awaitTermination(5, TimeUnit.SECONDS)) {
        exec.shutdownNow(); // принудительный interrupt всех задач
        exec.awaitTermination(1, TimeUnit.SECONDS);
    }
} catch (InterruptedException e) {
    exec.shutdownNow();
    Thread.currentThread().interrupt();
}
```

> [!INFO] Связано
> - [[Процессы и Потоки, Thread, Runnable, состояния потоков]] — состояния WAITING/BLOCKED/RUNNABLE
> - [[Synchronized]] — interrupt не прерывает ожидание монитора!
> - [[Lock]] — `lockInterruptibly()` — прерываемый захват блокировки