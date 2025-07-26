`CyclicBarrier` — это синхронизатор из пакета `java.util.concurrent`, который позволяет группе потоков (участников) ожидать друг друга в определённой точке (барьере), прежде чем продолжить выполнение. Его ключевая особенность — **цикличность**: после срабатывания барьера он автоматически сбрасывается и готов к повторному использованию. Эта статья подробно рассматривает `CyclicBarrier`, его реализацию в JVM, примеры, преимущества, ограничения, производительность и современные возможности (Java 21+).

## 1. Что такое `CyclicBarrier`?

- **Определение**: `CyclicBarrier` — синхронизатор, который блокирует потоки, вызвавшие `await()`, пока все участники не достигнут барьера, после чего они разблокируются, и барьер сбрасывается.
- **Назначение**:
    - Синхронизирует потоки на этапах выполнения.
    - Поддерживает циклические сценарии, где потоки многократно координируются.
- **Применение**:
    - Параллельные вычисления с этапами (например, итерационные алгоритмы).
    - Тестирование многопоточных сценариев.
    - Координация задач в пулах потоков.

## 2. Зачем нужен `CyclicBarrier`?

`CyclicBarrier` решает задачу синхронизации потоков в точке барьера, где все участники должны завершиться перед переходом к следующему этапу. Примеры:

- Параллельная обработка данных, где каждый поток выполняет часть вычислений, а затем все синхронизируются.
- Симуляция многопоточных систем с итерациями (например, моделирование, игровые циклы).
- Тестирование, где потоки должны начинать одновременно.

## 3. Основные методы

|Метод|Описание|
|---|---|
|`CyclicBarrier(int parties)`|Создаёт барьер для указанного числа участников|
|`CyclicBarrier(int parties, Runnable barrierAction)`|Создаёт барьер с действием, выполняемым при срабатывании|
|`await()`|Ожидает, пока все участники не достигнут барьера; возвращает индекс|
|`await(long timeout, TimeUnit unit)`|Ожидает с таймаутом; выбрасывает `TimeoutException` при истечении|
|`getParties()`|Возвращает общее число участников|
|`getNumberWaiting()`|Возвращает число потоков, ожидающих на барьере|
|`isBroken()`|Проверяет, сломан ли барьер|
|`reset()`|Сбрасывает барьер в начальное состояние, ломая текущий цикл|

- **Исключения**:
    - `InterruptedException`: Поток прерван.
    - `BrokenBarrierException`: Барьер сломан (например, из-за прерывания или таймаута).
    - `TimeoutException`: Истёк таймаут в `await(timeout, unit)`.

## 4. Принцип работы и внутренняя реализация

### 4.1. Взаимодействие потоков

1. Каждый поток вызывает `await()`, увеличивая счётчик ожидающих.
2. Когда число ожидающих достигает `parties`:
    - Выполняется `barrierAction` (если задан) одним из потоков.
    - Сбрасывается внутренний счётчик.
    - Все потоки разблокируются.
3. Барьер готов к следующему циклу.

### 4.2. Внутреннее устройство

- **Основа**: Реализован с использованием `ReentrantLock` и `Condition`.
- **Ключевые поля**:
    - `parties`: Число участников (фиксировано).
    - `count`: Текущее число ожидающих потоков.
    - `generation`: Объект для отслеживания циклов (поколений).
    - `broken`: Флаг, указывающий, что барьер сломан.
    - `lock`: `ReentrantLock` для синхронизации.
    - `trip`: `Condition` для ожидания и пробуждения потоков.
- **Механизм**:
    - `await()`: Захватывает `lock`, проверяет `count`. Если `count < parties`, поток ожидает через `trip.await()`.
    - Когда последний поток вызывает `await()`:
        - Выполняется `barrierAction`.
        - Вызывается `trip.signalAll()` для пробуждения всех.
        - Сбрасывается `count`, создаётся новое `generation`.
- **Сломанный барьер**:
    - Если поток прерывается или истекает таймаут, барьер ломается.
    - Все ожидающие потоки получают `BrokenBarrierException`.

### 4.3. JVM-реализация

- **Синхронизация**:
    - Использует `ReentrantLock` для защиты `count` и `generation`.
    - `Condition` (`trip`) управляет ожиданием через `LockSupport.park()` и `unpark()`.
- **JMM**:
    - `await()` и `signalAll()` создают **happens-before** отношения.
    - Изменения до срабатывания барьера видны после разблокировки.
- **Байт-код (упрощённый)**:
    
    ```java
    invokevirtual java/util/concurrent/CyclicBarrier.await()I
    // Захват lock, вызов trip.await() или signalAll()
    ```
    

## 5. Пример использования

```java
import java.util.concurrent.*;

public class CyclicBarrierExample {
    private static final int NUM_THREADS = 3;

    public static void main(String[] args) {
        Runnable barrierAction = () -> System.out.println("Все потоки достигли барьера, запускаем следующую фазу");
        CyclicBarrier barrier = new CyclicBarrier(NUM_THREADS, barrierAction);

        for (int i = 0; i < NUM_THREADS; i++) {
            Thread worker = new Thread(new Worker(barrier, i));
            worker.start();
        }
    }

    static class Worker implements Runnable {
        private final CyclicBarrier barrier;
        private final int id;

        Worker(CyclicBarrier barrier, int id) {
            this.barrier = barrier;
            this.id = id;
        }

        public void run() {
            try {
                System.out.println("Поток " + id + " выполняет часть работы...");
                Thread.sleep((long) (Math.random() * 3000));

                System.out.println("Поток " + id + " ждёт на барьере");
                int arrivalIndex = barrier.await();

                System.out.println("Поток " + id + " продолжает после барьера, индекс: " + arrivalIndex);

                // Повторное использование барьера
                Thread.sleep((long) (Math.random() * 2000));
                System.out.println("Поток " + id + " ждёт на барьере 2-го этапа");
                barrier.await();

                System.out.println("Поток " + id + " завершил работу");
            } catch (InterruptedException | BrokenBarrierException e) {
                Thread.currentThread().interrupt();
                System.err.println("Поток " + id + " прерван: " + e);
            }
        }
    }
}
```

**Объяснение**:

- Создаётся барьер для 3 потоков с `barrierAction`.
- Каждый поток выполняет работу, вызывает `await()`, и ждёт остальных.
- После срабатывания барьера выполняется `barrierAction`, потоки продолжают.
- Барьер используется повторно для второго этапа.

## 6. Практическое применение

- **Java EE / Jakarta EE**:
    - Синхронизация этапов обработки запросов в сервлетах.
    - Координация инициализации компонентов.
- **Spring**:
    - Параллельная обработка данных в `@Service`.
- **Пулы потоков**:
    - `ExecutorService` для синхронизации задач.
- **Параллельные вычисления**:
    - Итерационные алгоритмы (например, численные методы).
- **Тестирование**:
    - Синхронный запуск тестовых сценариев.

**Пример (Spring)**:

```java
@Service
public class DataProcessor {
    private final CyclicBarrier barrier;

    public DataProcessor(int parties) {
        this.barrier = new CyclicBarrier(parties, () -> System.out.println("Этап обработки завершён"));
    }

    public void processData(int id) {
        try {
            System.out.println("Поток " + id + " обрабатывает данные...");
            Thread.sleep((long) (Math.random() * 1000));
            barrier.await();
            System.out.println("Поток " + id + " переходит к следующему этапу");
        } catch (InterruptedException | BrokenBarrierException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 7. Современные возможности (Java 21+)

### 7.1. Виртуальные потоки

`CyclicBarrier` эффективен с виртуальными потоками (Java 21, Project Loom).

**Пример**:

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("Этап завершён"));
for (int i = 0; i < 3; i++) {
    int id = i;
    Thread.ofVirtual().start(() -> {
        try {
            System.out.println("Виртуальный поток " + id + " выполняет работу...");
            Thread.sleep((long) (Math.random() * 1000));
            barrier.await();
            System.out.println("Виртуальный поток " + id + " продолжает");
        } catch (InterruptedException | BrokenBarrierException e) {
            Thread.currentThread().interrupt();
        }
    });
}
```

**Особенности**:

- Лёгкие потоки снижают затраты на ожидание.
- Подходит для I/O-интенсивных задач.

### 7.2. Альтернативы

- **`Phaser`**: Гибкая многофазная синхронизация с динамическим числом участников.
- **`CompletableFuture`**: Асинхронная координация задач.
- **`ExecutorService`**: Управление задачами в пуле.

**Пример (`Phaser`)**:

```java
Phaser phaser = new Phaser(3);
for (int i = 0; i < 3; i++) {
    int id = i;
    Thread.ofVirtual().start(() -> {
        System.out.println("Поток " + id + " выполняет работу...");
        phaser.arriveAndAwaitAdvance();
        System.out.println("Поток " + id + " продолжает");
    });
}
```

## 8. Сравнение с другими синхронизаторами

|Инструмент|Повторное использование|Назначение|
|---|---|---|
|`CyclicBarrier`|✅|Все потоки ждут друг друга (барьер)|
|`CountDownLatch`|❌|Ожидание завершения событий|
|`Phaser`|✅|Многофазная синхронизация, динамическая|
|`Semaphore`|✅|Ограничение доступа к ресурсам|

**Отличие от `CountDownLatch`**:

- `CyclicBarrier` переиспользуем, фиксированное число участников, `barrierAction`.
- `CountDownLatch` одноразовый, поддерживает уменьшение счётчика разными потоками.

## 9. Подводные камни

1. **Игнорирование исключений**:
    
    ```java
    try {
        barrier.await();
    } catch (InterruptedException | BrokenBarrierException e) {
        // Игнорирование
    }
    ```
    
    **Решение**: Обрабатывайте исключения:
    
    ```java
    Thread.currentThread().interrupt();
    ```
    
2. **Сломанный барьер**:
    - Прерывание или таймаут ломает барьер.
    - **Решение**: Проверяйте `isBroken()` или используйте `reset()`.
3. **Неправильное число участников**:
    - Ошибка в `parties` приводит к deadlock.
    - **Решение**: Точно задавайте `parties`.
4. **Неправильное использование `barrierAction`**:
    - Долгое выполнение блокирует потоки.
    - **Решение**: Делайте `barrierAction` быстрым.

## 10. Производительность

- **Преимущества**:
    - Эффективное ожидание через `Condition` и `LockSupport`.
    - Минимальная нагрузка на CPU.
- **Недостатки**:
    - Высокая конкуренция за `lock` может замедлить срабатывание.
    - `barrierAction` может стать узким местом.
- **Сравнение**:
    - `CountDownLatch`: Проще, но одноразовый.
    - `Phaser`: Гибче, поддерживает динамическое число участников.
    - `CompletableFuture`: Асинхронность, меньше накладных расходов.
- **Рекомендации**:
    - Используйте для фиксированных этапов.
    - Переходите на `Phaser` для сложных сценариев.

## 11. Плюсы и минусы

|Плюсы|Минусы|
|---|---|
|✅ Цикличность|❌ Сложность обработки исключений|
|✅ Поддержка `barrierAction`|❌ Фиксированное число участников|
|✅ Эффективное ожидание|❌ Риск сломанного барьера|
|✅ Поддержка виртуальных потоков|❌ `barrierAction` может замедлить|

## 12. Лучшие практики

1. **Правильно задавайте число участников**:
    
    ```java
    CyclicBarrier barrier = new CyclicBarrier(taskCount);
    ```
    
2. **Обрабатывайте исключения**:
    
    ```java
    try {
        barrier.await();
    } catch (InterruptedException | BrokenBarrierException e) {
        Thread.currentThread().interrupt();
        handleBrokenBarrier();
    }
    ```
    
3. **Используйте таймаут для отказоустойчивости**:
    
    ```java
    try {
        barrier.await(5, TimeUnit.SECONDS);
    } catch (TimeoutException e) {
        handleTimeout();
    }
    ```
    
4. **Делайте `barrierAction` быстрым**:
    
    ```java
    Runnable barrierAction = () -> System.out.println("Этап завершён");
    ```
    
5. **Тестируйте синхронизацию**:
    
    ```java
    @Test
    void testCyclicBarrier() throws InterruptedException {
        CyclicBarrier barrier = new CyclicBarrier(2);
        AtomicInteger counter = new AtomicInteger(0);
        Runnable action = () -> counter.incrementAndGet();
        Thread t1 = new Thread(() -> {
            try {
                barrier.await();
                action.run();
            } catch (InterruptedException | BrokenBarrierException e) {
                Thread.currentThread().interrupt();
            }
        });
        Thread t2 = new Thread(() -> {
            try {
                barrier.await();
                action.run();
            } catch (InterruptedException | BrokenBarrierException e) {
                Thread.currentThread().interrupt();
            }
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        assertEquals(2, counter.get());
    }
    ```
    

## 13. Пример: Параллельные вычисления

```java
import java.util.concurrent.*;

public class MatrixMultiplication {
    private final CyclicBarrier barrier;
    private final double[][] matrixA;
    private final double[][] matrixB;
    private final double[][] result;
    private final int size;

    public MatrixMultiplication(int size, int threads) {
        this.barrier = new CyclicBarrier(threads, () -> System.out.println("Этап умножения завершён"));
        this.matrixA = new double[size][size];
        this.matrixB = new double[size][size];
        this.result = new double[size][size];
        this.size = size;
    }

    public void multiply() {
        int chunk = size / barrier.getParties();
        for (int i = 0; i < barrier.getParties(); i++) {
            int start = i * chunk;
            int end = (i + 1) * chunk;
            Thread.ofVirtual().start(() -> computeChunk(start, end));
        }
    }

    private void computeChunk(int start, int end) {
        try {
            for (int i = start; i < end && i < size; i++) {
                for (int j = 0; j < size; j++) {
                    for (int k = 0; k < size; k++) {
                        result[i][j] += matrixA[i][k] * matrixB[k][j];
                    }
                }
            }
            barrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Объяснение**:

- Каждый поток обрабатывает часть матрицы.
- `barrier.await()` синхронизирует потоки после каждого этапа.
- Поддерживает повторные итерации для многоэтапных вычислений.

## 14. Заключение

`CyclicBarrier` — мощный синхронизатор для координации потоков на этапах выполнения. Его цикличность позволяет повторно использовать барьер, а `barrierAction` добавляет гибкость. Реализация через `ReentrantLock` и `Condition` обеспечивает эффективность и потокобезопасность. Несмотря на риск сломанного барьера и необходимость обработки исключений, `CyclicBarrier` идеален для параллельных вычислений и тестирования. Современные альтернативы (`Phaser`, `CompletableFuture`) и поддержка виртуальных потоков (Java 21+) расширяют его применимость. Следование лучшим практикам обеспечивает надёжную синхронизацию.