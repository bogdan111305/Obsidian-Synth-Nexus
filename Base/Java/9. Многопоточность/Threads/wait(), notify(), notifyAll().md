Методы `wait()`, `notify()`, и `notifyAll()` из класса `Object` предназначены для **кооперативного взаимодействия потоков** через **монитор** объекта. Они позволяют одному потоку ожидать выполнения условия, а другому — уведомлять о его наступлении. Эта статья подробно рассматривает их работу, JVM-реализацию, включая **wait set** как очередь ожидания, примеры применения, подводные камни, производительность и современные альтернативы (Java 21+).

## 1. Основные понятия

- **Назначение**: Реализуют кооперативное взаимодействие потоков через монитор объекта, связанный с каждым объектом в Java.
- **Требование**: Методы `wait()`, `notify()`, и `notifyAll()` должны вызываться **только внутри `synchronized` блока или метода**, иначе выбрасывается `IllegalMonitorStateException`.
- **Механизм**:
    - `wait()`: Освобождает монитор и переводит поток в состояние **WAITING**, помещая его в **wait set** (очередь ожидания) объекта.
    - `notify()`: Пробуждает **один случайный** поток из wait set объекта.
    - `notifyAll()`: Пробуждает **все** потоки, находящиеся в wait set объекта.
- **Wait set**: Специальная структура данных в JVM, представляющая очередь потоков, ожидающих уведомления на конкретном объекте.

## 2. Подробное описание методов

### 2.1. `void wait() throws InterruptedException`

- Вызывается на объекте, чей монитор удерживает поток.
- **Действия**:
    - Освобождает монитор объекта.
    - Помещает поток в **wait set** объекта (очередь ожидания).
    - Переводит поток в состояние `WAITING`.
- Поток остаётся в ожидании, пока:
    - Другой поток не вызовет `notify()` или `notifyAll()` на том же объекте.
    - Поток не будет прерван (`InterruptedException`).
    - Не произойдёт спурийное пробуждение (spurious wakeup).
- После пробуждения поток переходит в состояние `BLOCKED`, ожидая повторного захвата монитора.

**Пример**:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait(); // Освобождает монитор, добавляет поток в wait set
    }
}
```

### 2.2. `void wait(long timeout) throws InterruptedException`

- Аналог `wait()`, но с таймаутом (в миллисекундах).
- Переводит поток в состояние `TIMED_WAITING`.
- Поток пробуждается, если:
    - Истекает таймаут.
    - Вызывается `notify()`/`notifyAll()`.
    - Поток прерывается (`InterruptedException`).

**Пример**:

```java
synchronized (lock) {
    lock.wait(1000); // Ожидание до 1 секунды
}
```

### 2.3. `void notify()`

- Пробуждает **один случайный** поток из wait set объекта.
- Пробуждённый поток переходит в состояние `BLOCKED`, ожидая монитор.
- JVM не гарантирует, какой именно поток будет выбран.

**Особенности**:

- Эффективен, если только один поток должен среагировать.
- Риск пропущенного сигнала, если условие не соответствует пробуждённому потоку.

### 2.4. `void notifyAll()`

- Пробуждает **все** потоки из wait set объекта.
- Все пробуждённые потоки переходят в `BLOCKED`, борясь за монитор.

**Особенности**:

- Безопаснее в сценариях с несколькими условиями или потоками.
- Может вызывать конкуренцию за монитор, снижая производительность.

## 3. Как это работает в JVM?

1. Поток входит в `synchronized(obj)` (вызывает инструкцию `monitorenter`).
2. Вызывает `obj.wait()`:
    - JVM добавляет поток в **wait set** объекта (очередь ожидания).
    - Освобождает монитор объекта (`monitorexit`).
    - Переводит поток в состояние `WAITING` (или `TIMED_WAITING` для `wait(timeout)`).
3. Другой поток в `synchronized(obj)` вызывает `obj.notify()` или `obj.notifyAll()`:
    - JVM переносит один или все потоки из wait set в очередь на монитор (`BLOCKED`).
4. Когда монитор освобождается, один из пробуждённых потоков захватывает его и продолжает выполнение после `wait()`.

**Байт-код (упрощённый)**:

```java
synchronized (lock) {
    lock.wait();
}
```

```java
monitorenter
invokevirtual java/lang/Object.wait()V
monitorexit
```

**Wait set**:

- **Очередь ожидания** (wait set) — это внутренняя структура JVM, связанная с каждым объектом.
- Хранит потоки, ожидающие уведомления (`wait()`).
- Управляется нативными примитивами ОС (например, `pthread_cond_wait` на Linux).

**Java Memory Model (JMM)**:

- `wait()` и `notify()`/`notifyAll()` создают **happens-before** отношения.
- Изменения до вызова `notify()` видны после выхода из `wait()` благодаря барьерам памяти.

## 4. Почему использовать цикл с `wait()`?

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
}
```

- **Спурийные пробуждения** (spurious wakeups): Поток может проснуться без вызова `notify()` из-за особенностей ОС или JVM.
- **Проверка условия**: Цикл гарантирует, что условие действительно выполнено перед продолжением.
- **Безопасность**: Защищает от ложного выполнения при случайном пробуждении.
- **Пример проблемы без цикла**:
    
    ```java
    synchronized (lock) {
        lock.wait(); // Если проснулся без условия, код продолжит ошибочно
        // Работа
    }
    ```
    

## 5. Разница между `notify()` и `notifyAll()`

|Метод|Действие|Когда использовать|
|---|---|---|
|`notify()`|Пробуждает один случайный поток|Один поток ждёт, простое условие|
|`notifyAll()`|Пробуждает все ожидающие потоки|Несколько условий или потоков|

- **`notify()`**: Эффективнее, но может пропустить нужный поток.
- **`notifyAll()`**: Безопаснее, но увеличивает конкуренцию за монитор.

**Рекомендация**: Используйте `notifyAll()` в сложных сценариях, чтобы избежать пропущенных сигналов.

## 6. Пример использования: Producer-Consumer

**Пример**:

```java
public class ProducerConsumer {
    private final Object lock = new Object();
    private int buffer;
    private boolean isProduced;

    public void produce() throws InterruptedException {
        synchronized (lock) {
            while (isProduced) {
                lock.wait(); // Ждём освобождения буфера
            }
            buffer = (int) (Math.random() * 100);
            System.out.println("Produced: " + buffer);
            isProduced = true;
            lock.notifyAll(); // Уведомляем всех потребителей
        }
    }

    public void consume() throws InterruptedException {
        synchronized (lock) {
            while (!isProduced) {
                lock.wait(); // Ждём появления данных
            }
            System.out.println("Consumed: " + buffer);
            isProduced = false;
            lock.notifyAll(); // Уведомляем всех производителей
        }
    }

    public static void main(String[] args) {
        ProducerConsumer pc = new ProducerConsumer();
        Thread producer = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    pc.produce();
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        Thread consumer = new Thread(() -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    pc.consume();
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        producer.start();
        consumer.start();
    }
}
```

**Особенности**:

- `notifyAll()` пробуждает всех ожидающих (производителей и потребителей).
- Цикл с `wait()` защищает от спурийных пробуждений.
- `synchronized` обеспечивает взаимное исключение для буфера.

## 7. Практическое применение

- **Java EE / Jakarta EE**:
    - Кооперация потоков в сервлетах для обработки асинхронных запросов.
    - Ожидание событий в EJB.
- **Spring**:
    - Синхронизация в `@Service` для обработки событий или очередей.
- **GUI**:
    - Ожидание пользовательских событий в Swing/JavaFX.
- **Паттерны**:
    - Producer-Consumer, барьеры, очереди сообщений.
    - Реализация пулов объектов или кэшей.

**Пример (Spring)**:

```java
@Service
public class EventProcessor {
    private final Object lock = new Object();
    private boolean eventReady;

    public void waitForEvent() throws InterruptedException {
        synchronized (lock) {
            while (!eventReady) {
                lock.wait();
            }
            eventReady = false;
            System.out.println("Event processed");
        }
    }

    public void signalEvent() {
        synchronized (lock) {
            eventReady = true;
            lock.notifyAll();
            System.out.println("Event signaled");
        }
    }
}
```

## 8. JVM-реализация

- **Wait set**:
    - **Очередь ожидания** (wait set) — структура данных в JVM для каждого объекта.
    - Хранит потоки, вызвавшие `wait()` на объекте.
    - Управляется нативными примитивами (например, `pthread_cond_wait` на Linux).
- **Байт-код**:
    
    ```java
    invokevirtual java/lang/Object.wait()V
    // Вызывает native-метод, добавляет поток в wait set
    ```
    
    ```java
    invokevirtual java/lang/Object.notify()V
    // Вызывает native-метод, пробуждает поток(и) из wait set
    ```
    
- **JMM**:
    - `wait()`: Создаёт барьер памяти (`store`), обеспечивая запись изменений.
    - `notify()`/`notifyAll()`: Создаёт барьер (`load`), обеспечивая видимость изменений.
- **Нативные вызовы**:
    - JVM использует OS-примитивы для управления wait set и состояниями потоков.

## 9. Современные альтернативы (Java 21+)

### 9.1. `java.util.concurrent.Condition`

`Condition` (из `ReentrantLock`) — более гибкая альтернатива `wait()`/`notify()`.

**Пример**:

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.Condition;

public class ConditionExample {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    private boolean ready;

    public void await() throws InterruptedException {
        lock.lock();
        try {
            while (!ready) {
                condition.await(); // Аналог wait()
            }
            ready = false;
        } finally {
            lock.unlock();
        }
    }

    public void signal() {
        lock.lock();
        try {
            ready = true;
            condition.signalAll(); // Аналог notifyAll()
        } finally {
            lock.unlock();
        }
    }
}
```

**Преимущества**:

- Поддержка нескольких условий (`newCondition()`).
- Интеграция с `tryLock()` и таймаутами.
- Явное управление блокировками.

# wait(), notify(), notifyAll()

### 9.2. Виртуальные потоки (Java 21, Project Loom)

Виртуальные потоки поддерживают `wait()`/`notify()` аналогично платформенным потокам, однако при блокировке на `synchronized` мониторе виртуальный поток **пинируется** (pinning) — JVM не может демонтировать его с несущего потока. Это ограничивает масштабируемость.

**Пример**:

```java
public class VirtualWaitNotify {
    private final Object lock = new Object();
    private boolean ready;

    public void producer() throws InterruptedException {
        Thread.ofVirtual().start(() -> {
            synchronized (lock) {
                ready = true;
                lock.notifyAll();
            }
        });
    }

    public void consumer() throws InterruptedException {
        Thread.ofVirtual().start(() -> {
            synchronized (lock) {
                try {
                    while (!ready) {
                        lock.wait(); // Пинирование виртуального потока
                    }
                    System.out.println("Получен сигнал");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
    }
}
```

**Рекомендация для виртуальных потоков**: Предпочитайте `ReentrantLock` + `Condition` вместо `synchronized` + `wait/notify` — это избегает пинирования и улучшает масштабируемость.

## 10. Подводные камни

1. **Вызов без `synchronized`**:

    ```java
    lock.wait(); // IllegalMonitorStateException
    ```

    **Решение**: Всегда вызывайте внутри `synchronized`:

    ```java
    synchronized (lock) { lock.wait(); }
    ```

2. **Использование `if` вместо `while`**:

    ```java
    synchronized (lock) {
        if (!condition) lock.wait(); // Риск spurious wakeup
    }
    ```

    **Решение**: Используйте `while`:

    ```java
    while (!condition) lock.wait();
    ```

3. **`notify()` вместо `notifyAll()`**:

    - Может пробудить не тот поток.
    - **Решение**: Используйте `notifyAll()` при нескольких условиях или потоках.

4. **Игнорирование `InterruptedException`**:

    ```java
    try { lock.wait(); } catch (InterruptedException e) { /* ignored */ }
    ```

    **Решение**: Восстанавливайте флаг:

    ```java
    Thread.currentThread().interrupt();
    ```

## 11. Производительность

- **`wait()`/`notify()`**: Эффективны при низкой конкуренции.
- **`notifyAll()`**: Увеличивает конкуренцию за монитор при большом числе потоков.
- **Сравнение**:
    - `wait()`/`notify()`: Встроены в `Object`, просты, но ограничены одним условием на монитор.
    - `Condition`: Несколько условий, поддержка `awaitNanos`, `awaitUntil`.
    - `BlockingQueue`: Инкапсулирует паттерн Producer-Consumer.
- **Рекомендации**:
    - Используйте `wait()`/`notify()` для простой кооперации.
    - Переходите на `Condition` или `BlockingQueue` для сложных сценариев.

## 12. Лучшие практики

1. **Всегда используйте `while` для проверки условия**:

    ```java
    synchronized (lock) {
        while (!condition) lock.wait();
    }
    ```

2. **Предпочитайте `notifyAll()` в сложных сценариях**:

    ```java
    lock.notifyAll();
    ```

3. **Используйте приватный объект-монитор**:

    ```java
    private final Object lock = new Object();
    ```

4. **Переходите на `Condition` для нескольких условий**:

    ```java
    Condition canRead = lock.newCondition();
    Condition canWrite = lock.newCondition();
    ```

5. **Для виртуальных потоков — `ReentrantLock` + `Condition`**:

    ```java
    lock.lock();
    try {
        while (!condition) condition.await();
    } finally {
        lock.unlock();
    }
    ```

6. **Тестируйте кооперацию**:

    ```java
    @Test
    void testWaitNotify() throws InterruptedException {
        ProducerConsumer pc = new ProducerConsumer();
        Thread producer = new Thread(() -> {
            try { pc.produce(); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        Thread consumer = new Thread(() -> {
            try { pc.consume(); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        producer.start();
        consumer.start();
        producer.join(2000);
        consumer.join(2000);
    }
    ```

## 13. Заключение

Методы `wait()`, `notify()`, `notifyAll()` — низкоуровневые инструменты кооперации потоков через монитор объекта. Они требуют `synchronized`, используют **wait set** для очередей ожидания и устанавливают **happens-before** гарантии через JMM. Паттерн «цикл + `wait()`» защищает от spurious wakeups. Современные альтернативы (`Condition`, `BlockingQueue`, `CompletableFuture`) и виртуальные потоки (Java 21) предоставляют более гибкое управление. Следование лучшим практикам — использование `while`, `notifyAll()`, приватных мониторов — обеспечивает корректную и производительную синхронизацию.