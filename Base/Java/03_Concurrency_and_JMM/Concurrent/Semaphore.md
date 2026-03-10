# Semaphore

> [!QUOTE] Суть
> **Semaphore** — счётчик разрешений (permits). `acquire()` уменьшает (блокирует если 0), `release()` увеличивает. Используется для ограничения параллельного доступа к ресурсам (пул соединений, rate limiting). `Semaphore(1)` работает как mutex (но без owner!).

`Semaphore` — это класс из пакета `java.util.concurrent`, предназначенный для контроля доступа к ограниченному числу ресурсов с помощью счётчика разрешений (permits). Он позволяет ограничивать количество одновременно работающих потоков, обеспечивая потокобезопасное управление ресурсами. `Semaphore` подходит для реализации пулов соединений, rate limiting и ограничения параллелизма.

## 1. Для чего используется `Semaphore`?

`Semaphore` применяется для:

- Ограничения числа потоков, одновременно использующих ресурс (например, пулы соединений, файлы).
- Управления параллелизмом в многопоточных приложениях.
- Реализации сложных синхронизационных сценариев, где требуется гибкое управление доступом.
- Координации задач в пулах потоков или тестировании.

**Примеры**:

- Ограничение числа одновременных запросов к базе данных.
- Контроль доступа к пулу серверов.
- Реализация семафоров в операционных системах.

## 2. Основные концепции

- **Разрешения (Permits)**: Целочисленный счётчик, определяющий, сколько потоков могут одновременно получить доступ.
- **Захват (`acquire`)**: Поток запрашивает разрешение. Если разрешений нет, поток блокируется.
- **Освобождение (`release`)**: Поток возвращает разрешение, позволяя другим потокам продолжить.
- **Справедливость (Fairness)**: При `fair = true` разрешения выдаются в порядке FIFO.
- **Множественные разрешения**: Возможность запрашивать/освобождать несколько разрешений одновременно.

## 3. Конструкторы и методы

|Метод/Конструктор|Описание|
|---|---|
|`Semaphore(int permits)`|Создаёт семафор с указанным числом разрешений|
|`Semaphore(int permits, boolean fair)`|Создаёт семафор с указанием справедливости|
|`acquire()`|Захватывает одно разрешение, блокируя поток до успеха|
|`acquire(int permits)`|Захватывает указанное число разрешений|
|`tryAcquire()`|Пытается захватить одно разрешение без блокировки|
|`tryAcquire(long timeout, TimeUnit unit)`|Пытается захватить с таймаутом|
|`tryAcquire(int permits)`|Пытается захватить несколько разрешений|
|`release()`|Освобождает одно разрешение|
|`release(int permits)`|Освобождает указанное число разрешений|
|`availablePermits()`|Возвращает текущее число доступных разрешений|
|`getQueueLength()`|Возвращает число потоков в очереди ожидания|
|`hasQueuedThreads()`|Проверяет, есть ли ожидающие потоки|
|`isFair()`|Проверяет, включена ли справедливость|

- **Исключения**:
    - `InterruptedException`: При прерывании потока во время ожидания.
    - `IllegalArgumentException`: При неверном числе разрешений в `acquire`/`release`.

## 4. Принцип работы и внутренняя реализация

### 4.1. Механизм работы

1. Поток вызывает `acquire()`:
    - Если `permits > 0`, счётчик уменьшается, поток продолжает.
    - Если `permits == 0`, поток блокируется в очереди.
2. Поток вызывает `release()`:
    - Счётчик разрешений увеличивается.
    - Один или несколько ожидающих потоков разблокируются.
3. При `fair = true`, разрешения выдаются в порядке FIFO.

### 4.2. Внутреннее устройство

- **Основа**: Реализован через `AbstractQueuedSynchronizer` (AQS).
- **Состояние**:
    - Поле `state` (volatile `int`) хранит число доступных разрешений.
    - `acquire()` уменьшает `state` через CAS (`compareAndSetState`).
    - `release()` увеличивает `state` и пробуждает потоки.
- **Очередь**: CLH-очередь (Craig, Landin, Hagersten) управляет ожидающими потоками.
- **LockSupport**: Использует `park()` для блокировки и `unpark()` для пробуждения.

### 4.3. JVM и JMM

- **JMM**: `acquire()` и `release()` создают **happens-before** отношения, обеспечивая видимость изменений.
- **Байт-код (упрощённый)**:
    
    ```java
    invokevirtual java/util/concurrent/Semaphore.acquire()V
    // CAS для state, park() при ожидании
    ```
    

## 5. Пример использования

### 5.1. Ограничение параллелизма

```java
import java.util.concurrent.Semaphore;

public class SemaphoreExample {
    private static final int MAX_CONCURRENT_THREADS = 3;
    private static final Semaphore semaphore = new Semaphore(MAX_CONCURRENT_THREADS, true);

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            Thread worker = new Thread(new Worker(i));
            worker.start();
        }
    }

    static class Worker implements Runnable {
        private final int id;

        Worker(int id) {
            this.id = id;
        }

        @Override
        public void run() {
            try {
                System.out.println("Поток " + id + " пытается получить разрешение...");
                semaphore.acquire();
                System.out.println("Поток " + id + " получил разрешение и начал работу");
                Thread.sleep(2000);
                System.out.println("Поток " + id + " закончил работу и освобождает разрешение");
                semaphore.release();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

**Объяснение**:

- Семафор ограничивает число одновременно работающих потоков до 3.
- Остальные потоки ждут, пока не освободится разрешение.
- `fair = true` обеспечивает FIFO-порядок.

### 5.2. Пул соединений

```java
import java.util.concurrent.Semaphore;

public class ConnectionPool {
    private static final int MAX_CONNECTIONS = 5;
    private final Semaphore semaphore = new Semaphore(MAX_CONNECTIONS);

    public void getConnection() throws InterruptedException {
        semaphore.acquire();
        System.out.println("Соединение получено, доступно: " + semaphore.availablePermits());
    }

    public void releaseConnection() {
        semaphore.release();
        System.out.println("Соединение освобождено, доступно: " + semaphore.availablePermits());
    }
}
```

## 6. Практическое применение

- **Java EE / Jakarta EE**:
    - Ограничение числа одновременных запросов к серверу.
    - Управление пулом соединений к базе данных.
- **Spring**:
    - Контроль параллелизма в `@Service` для обработки запросов.
- **Пулы потоков**:
    - `ExecutorService` для ограничения числа задач.
- **Тестирование**:
    - Симуляция ограниченных ресурсов в тестах.

**Пример (Spring)**:

```java
@Service
public class ResourceService {
    private final Semaphore semaphore = new Semaphore(3, true);

    public void accessResource(int id) throws InterruptedException {
        semaphore.acquire();
        try {
            System.out.println("Поток " + id + " получил доступ к ресурсу");
            Thread.sleep(1000);
        } finally {
            semaphore.release();
            System.out.println("Поток " + id + " освободил ресурс");
        }
    }
}
```

## 7. Современные возможности (Java 21+)

### 7.1. Виртуальные потоки

`Semaphore` эффективен с виртуальными потоками (Java 21, Project Loom).

**Пример**:

```java
import java.util.concurrent.Semaphore;

public class VirtualSemaphoreExample {
    private static final Semaphore semaphore = new Semaphore(3, true);

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            int id = i;
            Thread.ofVirtual().start(() -> {
                try {
                    semaphore.acquire();
                    System.out.println("Виртуальный поток " + id + " получил разрешение");
                    Thread.sleep(1000);
                    semaphore.release();
                    System.out.println("Виртуальный поток " + id + " освободил разрешение");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
    }
}
```

**Особенности**:

- Лёгкие потоки снижают затраты на ожидание.
- Подходит для I/O-интенсивных задач.

### 7.2. Альтернативы

- **`Phaser`**: Для многофазной синхронизации с динамическими участниками.
- **`CompletableFuture`**: Для асинхронной координации.
- **`ThreadPoolExecutor`**: Для управления задачами с ограничением.

**Пример (`CompletableFuture`)**:

```java
CompletableFuture.allOf(
    CompletableFuture.runAsync(() -> doWork()),
    CompletableFuture.runAsync(() -> doWork())
).join();
System.out.println("Задачи завершены");
```

## 8. Сравнение с другими синхронизаторами

|Класс|Основная задача|Особенности|
|---|---|---|
|`Semaphore`|Контроль доступа к ресурсам|Поддержка нескольких разрешений, fairness|
|`CountDownLatch`|Ожидание завершения событий|Одноразовый, только уменьшение счётчика|
|`CyclicBarrier`|Синхронизация группы в точке|Повторное использование, фиксированные участники|
|`Phaser`|Многофазная синхронизация|Динамические участники, гибкость|

**Отличия**:

- `Semaphore` управляет доступом, а не синхронизацией фаз.
- `CountDownLatch` одноразовый, не поддерживает повторное использование.
- `CyclicBarrier` фиксированное число участников, синхронизация в точке.
- `Phaser` поддерживает динамическое число участников и фазы.

## 9. Подводные камни

1. **Игнорирование `InterruptedException`**:
    
    ```java
    try {
        semaphore.acquire();
    } catch (InterruptedException e) {
        // Игнорирование
    }
    ```
    
    **Решение**: Восстанавливайте флаг:
    
    ```java
    Thread.currentThread().interrupt();
    ```
    
2. **Неправильное освобождение разрешений**:
    - Вызов `release()` без `acquire()` увеличивает счётчик некорректно.
    - **Решение**: Балансируйте `acquire()` и `release()`.
3. **Несправедливый порядок**:
    - При `fair = false` возможна "голодная" блокировка потоков.
    - **Решение**: Используйте `fair = true` для FIFO.
4. **Множественные разрешения**:
    - Ошибки в `acquire(int)`/`release(int)` приводят к рассинхронизации.
    - **Решение**: Проверяйте логику.

## 10. Производительность

- **Преимущества**:
    - Эффективное управление через AQS и `LockSupport`.
    - Гибкость с `tryAcquire` и `fairness`.
- **Недостатки**:
    - Высокая конкуренция замедляет очередь ожидания.
    - `fair = true` увеличивает накладные расходы.
- **Сравнение**:
    - `CountDownLatch`: Проще, но одноразовый.
    - `CyclicBarrier`: Для фиксированных групп, синхронизация.
    - `Phaser`: Гибче, но сложнее.
    - `CompletableFuture`: Асинхронность, меньше накладных расходов.
- **Рекомендации**:
    - Используйте `Semaphore` для ограничения ресурсов.
    - Рассмотрите `CompletableFuture` для асинхронных задач.

## 11. Плюсы и минусы

|Плюсы|Минусы|
|---|---|
|✅ Гибкость в управлении доступом|❌ Риск рассинхронизации|
|✅ Поддержка fairness|❌ Сложность отладки|
|✅ Поддержка виртуальных потоков|❌ Требует обработки исключений|
|✅ Множественные разрешения|❌ Возможна "голодная" блокировка|

## 12. Лучшие практики

1. **Балансируйте `acquire` и `release`**:
    
    ```java
    semaphore.acquire();
    try {
        // Работа
    } finally {
        semaphore.release();
    }
    ```
    
2. **Обрабатывайте исключения**:
    
    ```java
    try {
        semaphore.acquire();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    ```
    
3. **Используйте `tryAcquire` для гибкости**:
    
    ```java
    if (semaphore.tryAcquire(1, TimeUnit.SECONDS)) {
        try {
            // Работа
        } finally {
            semaphore.release();
        }
    }
    ```
    
4. **Выбирайте справедливость при необходимости**:
    
    ```java
    Semaphore semaphore = new Semaphore(3, true);
    ```
    
5. **Тестируйте синхронизацию**:
    
    ```java
    @Test
    void testSemaphore() throws InterruptedException {
        Semaphore semaphore = new Semaphore(2);
        AtomicInteger counter = new AtomicInteger();
        Thread t1 = new Thread(() -> {
            try {
                semaphore.acquire();
                counter.incrementAndGet();
                semaphore.release();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        Thread t2 = new Thread(() -> {
            try {
                semaphore.acquire();
                counter.incrementAndGet();
                semaphore.release();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        assertEquals(2, counter.get());
    }
    ```
    

## 13. Пример: Ограничение HTTP-запросов

```java
import java.util.concurrent.Semaphore;

public class HttpRequestLimiter {
    private final Semaphore semaphore;

    public HttpRequestLimiter(int maxRequests) {
        this.semaphore = new Semaphore(maxRequests, true);
    }

    public void sendRequest(int id) {
        try {
            if (semaphore.tryAcquire(1, TimeUnit.SECONDS)) {
                try {
                    System.out.println("Запрос " + id + " отправлен");
                    Thread.sleep(500);
                } finally {
                    semaphore.release();
                    System.out.println("Запрос " + id + " завершён");
                }
            } else {
                System.out.println("Запрос " + id + " отклонён: лимит достигнут");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Объяснение**:

- Семафор ограничивает число одновременных HTTP-запросов.
- `tryAcquire` предотвращает блокировку при превышении лимита.
- `finally` гарантирует освобождение разрешения.

## 14. Заключение

`Semaphore` — мощный инструмент для управления доступом к ограниченным ресурсам. Основанный на AQS, он обеспечивает потокобезопасность через атомарные операции и CLH-очередь. Поддержка справедливости, множественных разрешений и виртуальных потоков (Java 21+) делает его универсальным для пулов ресурсов, ограничения параллелизма и тестирования. Несмотря на риск рассинхронизации, лучшие практики и тестирование минимизируют проблемы. `Semaphore` превосходит `CountDownLatch` и `CyclicBarrier` в управлении ресурсами, но требует тщательной отладки.