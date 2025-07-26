# Phaser в Java: гибкая многофазная синхронизация

`Phaser` — это класс из пакета `java.util.concurrent`, предназначенный для сложной синхронизации потоков в многопоточных приложениях. Он позволяет группам потоков (участников) синхронизироваться в нескольких фазах, поддерживая динамическое изменение числа участников и фаз. `Phaser` объединяет и расширяет возможности `CountDownLatch` и `CyclicBarrier`, предлагая большую гибкость и повторное использование. Эта статья подробно рассматривает `Phaser`, его реализацию в JVM, примеры, преимущества, ограничения, производительность и современные возможности (Java 21+).

## 1. Для чего используется `Phaser`?

`Phaser` решает задачи синхронизации в сценариях, где потоки работают в несколько этапов (фаз) и должны координироваться перед переходом к следующему этапу. Основные применения:

- Многоэтапные параллельные вычисления (например, итерационные алгоритмы).
- Динамическая синхронизация с изменяющимся числом участников.
- Тестирование многопоточных сценариев с фазами.
- Выполнение действий при переходе между фазами.

## 2. Основные понятия

- **Фаза (Phase)**: Этап синхронизации, где участники вызывают методы `arrive()` или `arriveAndAwaitAdvance()` для завершения текущей фазы.
- **Участник (Party)**: Поток или задача, зарегистрированная в `Phaser`.
- **Регистрация/Дерегистрация**: Число участников может увеличиваться (`register()`) или уменьшаться (`arriveAndDeregister()`).
- **Advance**: Переход к следующей фазе, когда все участники прибыли.
- **Завершение**: `Phaser` завершается, если число участников становится 0 или вызывается `forceTermination()`.

## 3. Ключевые методы

|Метод|Описание|
|---|---|
|`register()`|Регистрирует нового участника|
|`arriveAndAwaitAdvance()`|Участник прибыл и ждёт остальных; переходит к следующей фазе|
|`arriveAndDeregister()`|Участник прибыл и выходит из `Phaser`|
|`arrive()`|Участник прибыл, но не ждёт остальных (асинхронный)|
|`getPhase()`|Возвращает номер текущей фазы|
|`getRegisteredParties()`|Возвращает число зарегистрированных участников|
|`getArrivedParties()`|Возвращает число участников, прибывших на текущую фазу|
|`isTerminated()`|Проверяет, завершён ли `Phaser`|
|`forceTermination()`|Принудительно завершает `Phaser`|

- **Исключения**:
    - `IllegalStateException`: Вызывается, если `Phaser` завершён.
    - `InterruptedException`: При прерывании потока во время ожидания.

## 4. Как работает `Phaser`?

1. Создаётся `Phaser` с начальным числом участников (или 0).
2. Участники выполняют работу и вызывают `arriveAndAwaitAdvance()` для синхронизации.
3. Когда все участники прибыли, `Phaser` переходит к следующей фазе, сбрасывая счётчик прибывших.
4. Динамическое добавление/удаление участников через `register()`/`arriveAndDeregister()`.
5. `onAdvance()` позволяет выполнять действия при переходе фаз.
6. `Phaser` завершается, если участники исчерпаны или вызвано `forceTermination()`.

## 5. Внутренняя реализация

### 5.1. AbstractQueuedSynchronizer (AQS)

- **Основа**: `Phaser` использует AQS для потокобезопасного управления.
- **Состояние**:
    - Поле `state` (volatile long) хранит номер фазы, число участников и прибывших.
    - Битовая структура: фазы, участники и прибывшие кодируются в одном `long`.
- **Очередь**: CLH-очередь (Craig, Landin, Hagersten) управляет ожидающими потоками.
- **CAS**: Атомарные операции (`compareAndSetState`) обновляют `state`.

### 5.2. Механизм

- `arriveAndAwaitAdvance()`: Увеличивает счётчик прибывших, если не все прибыли — поток блокируется через `LockSupport.park()`.
- **Advance**: Когда все участники прибыли, фаза увеличивается, счётчик сбрасывается, потоки пробуждаются через `LockSupport.unpark()`.
- `onAdvance()`: Вызывается перед переходом фаз, позволяет завершить `Phaser`.

### 5.3. JVM и JMM

- **JMM**: `arrive*` и `awaitAdvance` создают **happens-before** отношения.
    
- **Байт-код (упрощённый)**:
    
    ```java
    invokevirtual java/util/concurrent/Phaser.arriveAndAwaitAdvance()I
    // CAS для state, park() или unpark()
    ```
    

## 6. Пример использования

### 6.1. Многофазная синхронизация

```java
import java.util.concurrent.Phaser;

public class PhaserExample {
    private static final int NUMBER_OF_PARTIES = 3;

    public static void main(String[] args) {
        Phaser phaser = new Phaser(NUMBER_OF_PARTIES) {
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println("Завершена фаза: " + phase + ", участников: " + registeredParties);
                return phase >= 2 || registeredParties == 0;
            }
        };

        for (int i = 0; i < NUMBER_OF_PARTIES; i++) {
            Thread t = new Thread(new Task(phaser), "Worker-" + i);
            t.start();
        }
    }

    static class Task implements Runnable {
        private final Phaser phaser;

        Task(Phaser phaser) {
            this.phaser = phaser;
        }

        @Override
        public void run() {
            for (int phase = 0; phase < 3; phase++) {
                System.out.println(Thread.currentThread().getName() + " выполняет работу на фазе " + phase);
                doWork();
                phaser.arriveAndAwaitAdvance();
            }
            phaser.arriveAndDeregister();
            System.out.println(Thread.currentThread().getName() + " покинул Phaser");
        }

        private void doWork() {
            try {
                Thread.sleep((long) (Math.random() * 2000));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

**Объяснение**:

- `Phaser` создан для 3 участников, завершается после 3 фаз.
- Каждый поток выполняет работу, синхронизируется через `arriveAndAwaitAdvance()`.
- `onAdvance()` выводит информацию о фазе и завершает `Phaser` после 3 фаз.
- Потоки выходят через `arriveAndDeregister()`.

### 6.2. Динамическая регистрация

```java
Phaser phaser = new Phaser(2);
Thread t1 = new Thread(() -> {
    phaser.register(); // Новый участник
    System.out.println("Динамически добавлен участник");
    phaser.arriveAndAwaitAdvance();
});
t1.start();
```

## 7. Практическое применение

- **Java EE / Jakarta EE**:
    - Синхронизация этапов обработки в сервлетах.
    - Координация инициализации компонентов.
- **Spring**:
    - Параллельная обработка в `@Service` с динамическими задачами.
- **Пулы потоков**:
    - `ExecutorService` для многофазных задач.
- **Параллельные вычисления**:
    - Итерационные алгоритмы (например, машинное обучение).

**Пример (Spring)**:

```java
@Service
public class DataProcessor {
    private final Phaser phaser;

    public DataProcessor(int parties) {
        this.phaser = new Phaser(parties) {
            @Override
            protected boolean onAdvance(int phase, int parties) {
                System.out.println("Этап " + phase + " завершён");
                return phase >= 2;
            }
        };
    }

    public void processData(int id) {
        System.out.println("Поток " + id + " обрабатывает данные...");
        try {
            Thread.sleep((long) (Math.random() * 1000));
            phaser.arriveAndAwaitAdvance();
            System.out.println("Поток " + id + " переходит к следующему этапу");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 8. Сравнение с другими синхронизаторами

|Класс|Количество фаз|Регистрация участников|Повторное использование|Назначение|
|---|---|---|---|---|
|`CountDownLatch`|1|Нет|Нет|Одноразовое ожидание события|
|`CyclicBarrier`|Много|Фиксированное|Да|Синхронизация фиксированной группы|
|`Phaser`|Много|Динамическое|Да|Сложная многофазная синхронизация|
|`Semaphore`|Нет|Динамическое|Да|Ограничение доступа к ресурсам|

**Отличия**:

- `Phaser` поддерживает динамическое число участников и действия через `onAdvance()`.
- `CountDownLatch` одноразовый, не поддерживает фазы.
- `CyclicBarrier` фиксированное число участников.

## 9. Современные возможности (Java 21+)

### 9.1. Виртуальные потоки

`Phaser` эффективен с виртуальными потоками (Java 21, Project Loom).

**Пример**:

```java
Phaser phaser = new Phaser(3) {
    @Override
    protected boolean onAdvance(int phase, int parties) {
        System.out.println("Фаза " + phase + " завершена");
        return phase >= 2;
    }
};
for (int i = 0; i < 3; i++) {
    int id = i;
    Thread.ofVirtual().start(() -> {
        System.out.println("Виртуальный поток " + id + " выполняет работу...");
        phaser.arriveAndAwaitAdvance();
        System.out.println("Виртуальный поток " + id + " продолжает");
    });
}
```

**Особенности**:

- Лёгкие потоки снижают затраты на ожидание.
- Подходит для I/O-интенсивных задач.

### 9.2. Альтернативы

- `CompletableFuture`: Асинхронная координация задач.
- `ForkJoinPool`: Для рекурсивных задач.
- `ExecutorService`: Управление задачами в пуле.

**Пример (**`CompletableFuture`**)**:

```java
CompletableFuture.allOf(
    CompletableFuture.runAsync(() -> doWork()),
    CompletableFuture.runAsync(() -> doWork()),
    CompletableFuture.runAsync(() -> doWork())
).join();
System.out.println("Этап завершён");
```

## 10. Подводные камни

1. **Игнорирование исключений**:
    
    ```java
    try {
        phaser.arriveAndAwaitAdvance();
    } catch (InterruptedException e) {
        // Игнорирование
    }
    ```
    
    **Решение**: Восстанавливайте флаг:
    
    ```java
    Thread.currentThread().interrupt();
    ```
    
2. **Неправильное управление фазами**:
    
    - Ошибка в `onAdvance()` может прервать `Phaser`.
    - **Решение**: Тестируйте логику завершения.
3. **Динамическая регистрация**:
    
    - Неправильное добавление участников приводит к deadlock.
    - **Решение**: Точно отслеживайте `register()` и `arriveAndDeregister()`.
4. **Завершение** `Phaser`:
    
    - Если участники не синхронизированы, `Phaser` может завершиться преждевременно.
    - **Решение**: Проверяйте `isTerminated()`.

## 11. Производительность

- **Преимущества**:
    - Гибкость в управлении фазами и участниками.
    - Эффективное ожидание через `LockSupport`.
- **Недостатки**:
    - Сложность AQS увеличивает накладные расходы.
    - Высокая конкуренция замедляет переход фаз.
- **Сравнение**:
    - `CountDownLatch`: Проще, но одноразовый.
    - `CyclicBarrier`: Проще, но фиксированное число участников.
    - `CompletableFuture`: Асинхронность, меньше накладных расходов.
- **Рекомендации**:
    - Используйте `Phaser` для сложной синхронизации.
    - Рассмотрите `CompletableFuture` для асинхронных задач.

## 12. Плюсы и минусы

|Плюсы|Минусы|
|---|---|
|✅ Динамическая регистрация|❌ Сложность реализации|
|✅ Многофазная синхронизация|❌ Риск неправильного завершения|
|✅ Поддержка `onAdvance`|❌ Сложность отладки|
|✅ Поддержка виртуальных потоков|❌ Требует обработки исключений|

## 13. Лучшие практики

1. **Переопределяйте** `onAdvance` **для логики завершения**:
    
    ```java
    Phaser phaser = new Phaser() {
        @Override
        protected boolean onAdvance(int phase, int parties) {
            return phase >= maxPhases || parties == 0;
        }
    };
    ```
    
2. **Обрабатывайте исключения**:
    
    ```java
    try {
        phaser.arriveAndAwaitAdvance();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    ```
    
3. **Контролируйте регистрацию**:
    
    ```java
    int parties = phaser.getRegisteredParties();
    phaser.register();
    ```
    
4. **Проверяйте завершение**:
    
    ```java
    if (phaser.isTerminated()) {
        handleTermination();
    }
    ```
    
5. **Тестируйте синхронизацию**:
    
    ```java
    @Test
    void testPhaser() throws InterruptedException {
        Phaser phaser = new Phaser(2);
        AtomicInteger counter = new AtomicInteger();
        Thread t1 = new Thread(() -> {
            phaser.arriveAndAwaitAdvance();
            counter.incrementAndGet();
        });
        Thread t2 = new Thread(() -> {
            phaser.arriveAndAwaitAdvance();
            counter.incrementAndGet();
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        assertEquals(2, counter.get());
    }
    ```
    

## 14. Пример: Параллельные вычисления

```java
import java.util.concurrent.Phaser;

public class MatrixMultiplication {
    private final Phaser phaser;
    private final double[][] matrixA;
    private final double[][] matrixB;
    private final double[][] result;
    private final int size;

    public MatrixMultiplication(int size, int threads) {
        this.phaser = new Phaser(threads) {
            @Override
            protected boolean onAdvance(int phase, int parties) {
                System.out.println("Фаза " + phase + " завершена");
                return phase >= 1;
            }
        };
        this.matrixA = new double[size][size];
        this.matrixB = new double[size][size];
        this.result = new double[size][size];
        this.size = size;
    }

    public void multiply() {
        int chunk = size / phaser.getRegisteredParties();
        for (int i = 0; i < phaser.getRegisteredParties(); i++) {
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
            phaser.arriveAndAwaitAdvance();
        } catch (Exception e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Объяснение**:

- Каждый поток обрабатывает часть матрицы.
- `phaser.arriveAndAwaitAdvance()` синхронизирует потоки после этапа.
- `onAdvance()` завершает `Phaser` после 2 фаз.

## 15. Заключение

`Phaser` — мощный и гибкий инструмент для многофазной синхронизации с динамическим числом участников. Основанный на AQS, он обеспечивает потокобезопасность и эффективность через CAS и `LockSupport`. Поддержка `onAdvance`, динамической регистрации и виртуальных потоков (Java 21+) делает его идеальным для сложных сценариев, таких как параллельные вычисления и тестирование. Несмотря на сложность реализации, лучшие практики и тестирование минимизируют риски. `Phaser` превосходит `CountDownLatch` и `CyclicBarrier` в гибкости, но требует тщательной отладки.