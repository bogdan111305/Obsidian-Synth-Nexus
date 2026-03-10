# CountDownLatch

> [!QUOTE] Суть
> **CountDownLatch** — одноразовый счётчик. Потоки вызывают `await()` и ждут пока счётчик не достигнет нуля через `countDown()`. **Нельзя сбросить** (одноразовый). Используй для: ожидания инициализации, параллельного старта, ожидания завершения N задач.

`CountDownLatch` — это синхронизатор из пакета `java.util.concurrent`, который позволяет **одному или нескольким потокам ожидать**, пока **другие потоки завершат выполнение** определённого числа операций. Он идеально подходит для координации задач, таких как ожидание загрузки ресурсов или синхронизация старта.

## 1. Что такое `CountDownLatch`?

- **Определение**: `CountDownLatch` — синхронизатор, который блокирует потоки, вызвавшие `await()`, пока счётчик не достигнет нуля.
- **Назначение**:
    - Координирует завершение нескольких задач перед продолжением.
    - Поддерживает сценарии "ожидания событий" или "синхронного старта".
- **Основа**: Использует `AbstractQueuedSynchronizer` (AQS) для управления счётчиком и очередью ожидания.

**Пример**:

```java
CountDownLatch latch = new CountDownLatch(3); // Ожидаем 3 события
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doWork();
        latch.countDown(); // Уменьшаем счётчик
    }).start();
}
latch.await(); // Главный поток ждёт, пока счётчик не станет 0
System.out.println("Все задачи завершены");
```

## 2. Как работает `CountDownLatch`?

### 2.1. Внутреннее устройство

- **Основа**: Реализован через `AbstractQueuedSynchronizer` (AQS).
- **Счётчик**: Поле `state` в AQS хранит текущее значение счётчика.
- **Механизм**:
    - `await()`: Блокирует поток, если `state > 0`.
    - `countDown()`: Атомарно уменьшает `state`.
    - Когда `state == 0`, все ожидающие потоки разблокируются.

### 2.2. Механика AQS

- **Очередь CLH**: Потоки, вызвавшие `await()`, добавляются в CLH-очередь (Craig, Landin, and Hagersten queue).
- **Блокировка**: Потоки "усыпляются" через `LockSupport.park()`, минимизируя использование CPU.
- **Пробуждение**:
    - При `state == 0`, метод `releaseShared()` вызывает `unparkSuccessor()` для всех потоков в очереди.
    - Потоки пробуждаются через `LockSupport.unpark()` и продолжают выполнение.

**Внутренние вызовы**:

- `await()` → `AQS.acquireSharedInterruptibly(1)`:
    - Проверяет `tryAcquireShared()`: возвращает `-1`, если `state > 0` (поток паркуется).
    - Возвращает `1`, если `state == 0` (поток продолжает).
- `countDown()` → `AQS.releaseShared(1)`:
    - Вызывает `tryReleaseShared()`: уменьшает `state`.
    - При `state == 0` вызывает `doReleaseShared()` для пробуждения всех.

## 3. Основные методы

|Метод|Описание|
|---|---|
|`await()`|Блокирует поток, пока счётчик не станет 0|
|`await(timeout, unit)`|Блокирует с таймаутом, возвращает `false` при истечении|
|`countDown()`|Уменьшает счётчик на 1|
|`getCount()`|Возвращает текущее значение счётчика|

## 4. Примеры использования

### 4.1. Ожидание завершения задач

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        performTask();
        latch.countDown();
    }).start();
}
latch.await();
System.out.println("Все задачи завершены");
startNextPhase();
```

### 4.2. Синхронизация старта ("По сигналу марш!")

```java
int N = 5;
CountDownLatch startSignal = new CountDownLatch(1);
CountDownLatch doneSignal = new CountDownLatch(N);

for (int i = 0; i < N; i++) {
    new Thread(() -> {
        try {
            startSignal.await(); // Ждём сигнала
            doWork();
            doneSignal.countDown();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }).start();
}

prepareEverything();
startSignal.countDown(); // Даём сигнал
doneSignal.await(); // Ждём завершения
System.out.println("Все потоки завершили работу");
```

### 4.3. Загрузка ресурсов

```java
CountDownLatch latch = new CountDownLatch(3);

new Thread(() -> {
    loadFromFile();
    latch.countDown();
}).start();
new Thread(() -> {
    loadFromDatabase();
    latch.countDown();
}).start();
new Thread(() -> {
    loadFromNetwork();
    latch.countDown();
}).start();

latch.await();
System.out.println("Все ресурсы загружены");
initApplication();
```

### 4.4. Тестирование многопоточности

```java
int threadCount = 100;
CountDownLatch startLatch = new CountDownLatch(1);
CountDownLatch finishLatch = new CountDownLatch(threadCount);

for (int i = 0; i < threadCount; i++) {
    new Thread(() -> {
        try {
            startLatch.await();
            simulateUserAction();
            finishLatch.countDown();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }).start();
}

startLatch.countDown();
finishLatch.await();
System.out.println("Все пользователи завершили сценарий");
```

### 4.5. Ожидание с таймаутом

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 2; i++) {
    new Thread(() -> {
        try {
            Thread.sleep(1000);
            latch.countDown();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }).start();
}

try {
    if (latch.await(2, TimeUnit.SECONDS)) {
        System.out.println("Все потоки завершили работу вовремя");
    } else {
        System.out.println("Таймаут истёк, счётчик: " + latch.getCount());
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

**Особенности**:

- Возвращает `false` при истечении таймаута.
- Выбрасывает `InterruptedException` при прерывании.

## 5. Особенности `CountDownLatch`

### 5.1. Одноразовое использование

- После достижения `state == 0`, `CountDownLatch` нельзя сбросить.
- `await()` возвращает `true` немедленно, если счётчик уже 0.
- Для повторного использования создавайте новый объект:
    
    ```java
    CountDownLatch latch = new CountDownLatch(2);
    latch.countDown();
    latch.countDown();
    latch.await(); // true сразу
    latch = new CountDownLatch(3); // Новый объект
    ```
    

### 5.2. Поддержка нескольких ожидающих потоков

- Несколько потоков могут вызывать `await()` и ждать, пока счётчик не станет 0.
- Все ожидающие потоки разбуждаются одновременно.

**Пример**:

```java
CountDownLatch latch = new CountDownLatch(5);

for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        doHeavyWork();
        latch.countDown();
    }).start();
}

for (int i = 0; i < 3; i++) {
    int id = i;
    new Thread(() -> {
        try {
            latch.await();
            System.out.println("Поток " + id + ": начинаю после ожидания");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }).start();
}
```

## 6. JVM-реализация

- **AQS**:
    - Поле `state` хранит счётчик.
    - `await()` использует `acquireSharedInterruptibly()` для проверки `state`.
    - `countDown()` использует `releaseShared()` для уменьшения `state`.
- **CLH-очередь**:
    - Потоки, ожидающие через `await()`, добавляются в очередь ожидания AQS.
    - Реализована как двусвязный список для эффективности.
- **LockSupport**:
    - `park()`: Усыпляет поток, минимизируя использование CPU.
    - `unpark()`: Пробуждает потоки при `state == 0`.
- **JMM**:
    - `countDown()` и `await()` создают **happens-before** отношения.
    - Изменения до `countDown()` видны после выхода из `await()`.

## 7. Практическое применение

- **Java EE / Jakarta EE**:
    - Ожидание инициализации компонентов (серверы, пулы соединений).
- **Spring**:
    - Координация загрузки конфигураций в `@Service`.
- **Пулы потоков**:
    - `ExecutorService` для ожидания завершения задач.
- **Тестирование**:
    - Синхронизация тестовых сценариев.

**Пример (Spring)**:

```java
@Service
public class ResourceInitializer {
    private final CountDownLatch latch = new CountDownLatch(3);

    public void loadResource(String name) {
        // Загрузка ресурса
        latch.countDown();
    }

    public void waitForInitialization() throws InterruptedException {
        latch.await();
        System.out.println("Все ресурсы загружены");
    }
}
```

## 8. Сравнение с другими синхронизаторами

|Инструмент|Повторное использование|Назначение|
|---|---|---|
|`CountDownLatch`|❌|Ожидание завершения событий|
|`CyclicBarrier`|✅|Все потоки ждут друг друга (барьер)|
|`Phaser`|✅|Многофазная синхронизация, масштабируемая|
|`Semaphore`|✅|Ограничение доступа к ресурсам|

## 9. Современные возможности (Java 21+)

### 9.1. Виртуальные потоки

`CountDownLatch` эффективен с виртуальными потоками (Java 21, Project Loom).

**Пример**:

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    Thread.ofVirtual().start(() -> {
        doWork();
        latch.countDown();
    });
}
latch.await();
System.out.println("Все виртуальные потоки завершили работу");
```

**Особенности**:

- Лёгкие потоки снижают затраты на ожидание.
- Подходит для I/O-интенсивных задач.

### 9.2. Альтернативы

- **`CyclicBarrier`**: Для повторного использования.
- **`Phaser`**: Для многофазной синхронизации.
- **`CompletableFuture`**: Для асинхронной координации.

**Пример (`CompletableFuture`)**:

```java
CompletableFuture.allOf(
    CompletableFuture.runAsync(() -> loadFromFile()),
    CompletableFuture.runAsync(() -> loadFromDatabase()),
    CompletableFuture.runAsync(() -> loadFromNetwork())
).join();
System.out.println("Все ресурсы загружены");
```

## 10. Подводные камни

1. **Одноразовое использование**:
    - Нельзя сбросить счётчик.
    - **Решение**: Создавайте новый `CountDownLatch`.
2. **Игнорирование `InterruptedException`**:
    
    ```java
    try {
        latch.await();
    } catch (InterruptedException e) {
        // Игнорирование
    }
    ```
    
    **Решение**: Восстанавливайте флаг:
    
    ```java
    Thread.currentThread().interrupt();
    ```
    
3. **Неправильный счётчик**:
    - Установка неверного начального значения.
    - **Решение**: Тщательно проверяйте количество задач.
4. **Таймаут без проверки**:
    - Игнорирование возвращаемого `false`.
    - **Решение**: Проверяйте результат `await(timeout, unit)`.

## 11. Производительность

- **Преимущества**:
    - Эффективное ожидание через `LockSupport.park()`.
    - Минимальная нагрузка на CPU.
- **Недостатки**:
    - Одноразовое использование ограничивает гибкость.
    - Высокая конкуренция за AQS может замедлить пробуждение.
- **Сравнение**:
    - `CyclicBarrier`: Подходит для повторных барьеров.
    - `Phaser`: Масштабируемость для сложных сценариев.
    - `CompletableFuture`: Асинхронность, меньше накладных расходов.
- **Рекомендации**:
    - Используйте для простых сценариев ожидания.
    - Переходите на `Phaser` для многофазной синхронизации.

## 12. Плюсы и минусы

|Плюсы|Минусы|
|---|---|
|✅ Простота использования|❌ Одноразовое использование|
|✅ Поддержка нескольких ожидающих|❌ Нет сброса счётчика|
|✅ Эффективное ожидание|❌ Ограниченная гибкость|
|✅ Поддержка виртуальных потоков|❌ Требует обработки исключений|

## 13. Лучшие практики

1. **Правильно задавайте начальный счётчик**:
    
    ```java
    CountDownLatch latch = new CountDownLatch(taskCount);
    ```
    
2. **Обрабатывайте `InterruptedException`**:
    
    ```java
    try {
        latch.await();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    ```
    
3. **Используйте таймаут для отказоустойчивости**:
    
    ```java
    if (!latch.await(5, TimeUnit.SECONDS)) {
        handleTimeout();
    }
    ```
    
4. **Переходите на альтернативы для повторного использования**:
    
    ```java
    CyclicBarrier barrier = new CyclicBarrier(parties);
    ```
    
5. **Тестируйте синхронизацию**:
    
    ```java
    @Test
    void testCountDownLatch() throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(2);
        Thread t1 = new Thread(() -> latch.countDown());
        Thread t2 = new Thread(() -> latch.countDown());
        t1.start(); t2.start();
        latch.await(1, TimeUnit.SECONDS);
        assertEquals(0, latch.getCount());
    }
    ```
    

## 14. Заключение

`CountDownLatch` — мощный и простой синхронизатор для ожидания завершения задач. Основанный на AQS, он использует CLH-очередь и `LockSupport` для эффективного управления потоками. Несмотря на ограничение одноразового использования, он идеален для сценариев инициализации, тестирования и синхронного старта. Современные альтернативы (`CyclicBarrier`, `Phaser`, `CompletableFuture`) и поддержка виртуальных потоков (Java 21+) расширяют возможности. Следование лучшим практикам обеспечивает надёжную и эффективную синхронизацию.