# synchronized

**`synchronized`** — ключевое слово в Java, обеспечивающее **мониторный (взаимный) эксклюзивный доступ** к участку кода или объекту, предотвращая гонки данных (race conditions). Оно гарантирует, что только один поток может выполнять синхронизированный код для данного монитора, обеспечивая взаимное исключение и видимость изменений. Эта статья подробно рассматривает `synchronized`, его реализацию в JVM, deadlock, потокобезопасность, подводные камни, производительность и современные альтернативы (Java 21+).

## 1. Что такое `synchronized`?

- **Назначение**:
    - **Взаимное исключение (mutex)**: Только один поток может владеть монитором объекта.
    - **Видимость**: Изменения внутри `synchronized` блока видны другим потокам после освобождения монитора.
- **Гарантии**:
    - Предотвращает гонки данных, обеспечивая целостность общих ресурсов.
    - Устанавливает **happens-before** отношение (Java Memory Model, JMM), гарантируя видимость изменений.

## 2. Где используется `synchronized`?

`synchronized` применяется в двух формах:

### 2.1. Синхронизированный метод

- Синхронизирует весь метод, используя монитор объекта (`this` для нестатических методов, класс для статических).

**Пример**:

```java
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

**Особенности**:

- Монитор — объект `Counter` (или класс для `static` методов).
- Потоки, вызывающие `increment()`, ждут, если монитор занят.

### 2.2. Синхронизированный блок

- Синхронизирует только часть кода, используя указанный объект как монитор.

**Пример**:

```java
public class MyClass {
    private final Object lock = new Object();
    private int value;

    public void updateValue() {
        synchronized (lock) {
            value++; // Критическая секция
        }
    }
}
```

**Преимущества**:

- Точный контроль над синхронизируемым кодом.
- Снижение времени удержания монитора.

## 3. Мониторы и их работа

- **Монитор** — встроенный механизм блокировки, связанный с каждым объектом в Java.
- **Процесс**:
    - Поток вызывает `monitorenter` для захвата монитора.
    - Если монитор занят, поток переходит в состояние `BLOCKED`.
    - После завершения вызывается `monitorexit` для освобождения монитора.
- **Рекурсивность**:
    - Поток может повторно захватить монитор (рекурсивный lock).
    - JVM отслеживает количество входов (`monitorenter`) и выходов (`monitorexit`).

**Пример байт-кода**:

```java
public synchronized void increment() {
    count++;
}
```

**Байт-код**:

```java
monitorenter
iload_1
iconst_1
iadd
istore_1
monitorexit
```

## 4. JVM-реализация и оптимизации

- **Инструкции JVM**:
    - `monitorenter`: Захват монитора.
    - `monitorexit`: Освобождение монитора.
- **Типы мониторов**:
    - **Fat monitor (heavyweight lock)**: Использует системные примитивы ОС (например, `pthread_mutex` на Linux). Затратен при конкуренции.
    - **Lightweight lock**: Применяет CAS (Compare-And-Swap) для захвата монитора без обращения к ОС, если конкуренция низкая.
    - **Biased lock** (устарел): Оптимизация для одного потока. Помечен deprecated в Java 15 (JEP 374), отключён по умолчанию в Java 17, **удалён в Java 21**.
- **JMM (Java Memory Model)**:
    - `synchronized` устанавливает барьеры памяти (`store` при входе, `load` при выходе).
    - Гарантирует видимость изменений для всех потоков, захватывающих тот же монитор.

## 5. Взаимное исключение и видимость

- **Взаимное исключение**: Только один поток выполняет `synchronized` блок/метод.
- **Видимость**: Изменения, сделанные в `synchronized` блоке, видны другим потокам после освобождения монитора.

**Пример**:

```java
public class SharedResource {
    private int data;

    public void setData(int value) {
        synchronized (this) {
            data = value; // Видно всем потокам после выхода
        }
    }

    public int getData() {
        synchronized (this) {
            return data;
        }
    }
}
```

## 6. Deadlock (взаимная блокировка)

**Deadlock** — ситуация, когда два или более потоков бесконечно ждут друг друга, удерживая ресурсы.

### 6.1. Условия возникновения deadlock

Для deadlock необходимы все четыре условия (Coffman conditions):

|Условие|Описание|
|---|---|
|**Взаимное исключение**|Ресурс доступен только одному потоку.|
|**Удержание и ожидание**|Поток держит один ресурс и ждёт другой.|
|**Невозможность принудительного освобождения**|Ресурсы нельзя отобрать у потока.|
|**Круговое ожидание**|Потоки образуют замкнутый круг, ожидая ресурсы друг друга.|

### 6.2. Пример deadlock

```java
public class DeadlockExample {
    private final Object resourceA = new Object();
    private final Object resourceB = new Object();

    public void method1() {
        synchronized (resourceA) {
            System.out.println("Thread 1: Locked resource A");
            try { Thread.sleep(100); } catch (InterruptedException e) {}
            synchronized (resourceB) {
                System.out.println("Thread 1: Locked resource B");
            }
        }
    }

    public void method2() {
        synchronized (resourceB) {
            System.out.println("Thread 2: Locked resource B");
            try { Thread.sleep(100); } catch (InterruptedException e) {}
            synchronized (resourceA) {
                System.out.println("Thread 2: Locked resource A");
            }
        }
    }

    public static void main(String[] args) {
        DeadlockExample example = new DeadlockExample();
        Thread t1 = new Thread(() -> example.method1());
        Thread t2 = new Thread(() -> example.method2());
        t1.start();
        t2.start();
    }
}
```

**Что происходит**:

- Поток `t1` захватывает `resourceA` и ждёт `resourceB`.
- Поток `t2` захватывает `resourceB` и ждёт `resourceA`.
- Результат: Взаимная блокировка.

### 6.3. Обнаружение deadlock

- **Симптомы**: Программа "зависает", потоки не завершаются.
- **Инструменты**:
    - `jstack <pid>`: Выводит stack trace, показывая блокировки.
    - IDE (IntelliJ IDEA): Дебаггер для анализа потоков.
    - JConsole, VisualVM: Мониторинг блокировок.
- **Пример вывода `jstack`**:
    
    ```log
    Found one Java-level deadlock:
    "Thread-1":
      waiting to lock monitor 0x000000000263a388 (object resourceB)
      which is held by "Thread-2"
    "Thread-2":
      waiting to lock monitor 0x000000000263a3d8 (object resourceA)
      which is held by "Thread-1"
    ```
    

### 6.4. Избежание deadlock

1. **Единый порядок захвата**:
    
    ```java
    synchronized (resourceA) {
        synchronized (resourceB) {
            // Работа
        }
    }
    ```
    
2. **Использование `ReentrantLock` с `tryLock`**:
    
    ```java
    lock1.tryLock(100, TimeUnit.MILLISECONDS);
    ```
    
3. **Минимизация блокировок**: Уменьшайте код внутри `synchronized`.
4. **Использование высокоуровневых примитивов**: `ConcurrentHashMap`, `Semaphore`, `CountDownLatch`.
5. **Анализ кода**: Проверяйте порядок захвата мониторов.

## 7. Потокобезопасность

Объект **потокобезопасен**, если его методы могут вызываться из нескольких потоков без нарушения целостности данных.

- **Методы достижения**:
    - **Синхронизация**: `synchronized`, `ReentrantLock`.
    - **Иммутабельность**: Объекты, неизменяемые после создания (`String`, `record`).
    - **Атомарные классы**: `AtomicInteger`, `AtomicReference`.
    - **Потокобезопасные коллекции**: `ConcurrentHashMap`, `CopyOnWriteArrayList`.

**Пример потокобезопасного объекта**:

```java
public class ThreadSafeCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}
```

## 8. Практическое применение

- **Java EE / Jakarta EE**:
    - Синхронизация в сервлетах для защиты общих ресурсов.
    - Управление транзакциями в EJB.
- **Spring**:
    - `@Synchronized` или `synchronized` в `@Service` для защиты данных.
- **Коллекции**:
    - `Collections.synchronizedMap()` для потокобезопасных мап.
- **GUI**:
    - Синхронизация в Swing/JavaFX для обновления UI.

**Пример (Spring)**:

```java
@Service
public class CounterService {
    private int count;

    @Synchronized
    public void increment() {
        count++;
    }
}
```

## 9. Современные альтернативы (Java 21+)

### 9.1. ReentrantLock

`ReentrantLock` (Java 5+) — более гибкая альтернатива `synchronized`.

**Пример**:

```java
import java.util.concurrent.locks.ReentrantLock;

public class LockCounter {
    private final ReentrantLock lock = new ReentrantLock();
    private int count;

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

**Преимущества**:

- Поддержка `tryLock()` для избежания deadlock.
- Справедливость (fairness): Очередь потоков.
- Условные переменные (`Condition`).

### 9.2. StampedLock

`StampedLock` (Java 8+) поддерживает оптимистические блокировки.

**Пример**:

```java
import java.util.concurrent.locks.StampedLock;

public class OptimisticCounter {
    private final StampedLock lock = new StampedLock();
    private int count;

    public int getCount() {
        long stamp = lock.tryOptimisticRead();
        int result = count;
        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                result = count;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return result;
    }
}
```

**Преимущества**:

- Оптимистическое чтение без блокировки.
- Подходит для сценариев с частым чтением.

### 9.3. Виртуальные потоки

Виртуальные потоки (Java 21, Project Loom) поддерживают `synchronized`, но с важным ограничением: блокировка на `synchronized` мониторе вызывает **пинирование (pinning)** — JVM не может снять виртуальный поток с несущего платформенного потока. Это снижает масштабируемость при высокой конкуренции.

**Пример**:

```java
public class VirtualThreadSync {
    private int count;

    public synchronized void increment() {
        count++; // Пинирование при конкуренции
    }

    public static void main(String[] args) {
        VirtualThreadSync counter = new VirtualThreadSync();
        Thread.ofVirtual().start(() -> counter.increment());
    }
}
```

**Рекомендация**: В виртуальных потоках предпочитайте `ReentrantLock` — он не вызывает пинирования:

```java
private final ReentrantLock lock = new ReentrantLock();
public void increment() {
    lock.lock();
    try { count++; } finally { lock.unlock(); }
}
```

## 10. Подводные камни

1. **Синхронизация на публичных объектах**:
    
    ```java
    synchronized (this) { ... } // Опасно, если this доступен извне
    ```
    
    **Решение**: Используйте приватные объекты:
    
    ```java
    private final Object lock = new Object();
    ```
    
2. **Долгие операции в `synchronized`**:
    
    - Замедляют выполнение, увеличивают конкуренцию.
    - **Решение**: Минимизируйте код в `synchronized` блоках.
3. **Игнорирование deadlock**:
    
    - Неправильный порядок захвата мониторов.
    - **Решение**: Устанавливайте единый порядок.
4. **Низкая производительность**:
    
    - Fat monitors затратны при высокой конкуренции.
    - **Решение**: Используйте `ReentrantLock` или атомарные классы.

## 11. Производительность

- **Fat monitors**: Высокие затраты на системные вызовы.
- **Lightweight/biased locks**: Минимизируют накладные расходы при низкой конкуренции.
- **Сравнение**:
    - `synchronized`: Простота, но ограниченная гибкость.
    - `ReentrantLock`: Гибкость, поддержка `tryLock`.
    - `Atomic` классы: Высокая производительность для простых операций.
- **Рекомендации**:
    - Используйте `synchronized` для простых случаев.
    - Применяйте `ReentrantLock` для сложных сценариев.
    - Используйте `Atomic` для операций без блокировок.

## 12. Лучшие практики

1. **Используйте приватные мониторы**:
    
    ```java
    private final Object lock = new Object();
    ```
    
2. **Минимизируйте блокировки**:
    
    ```java
    synchronized (lock) { value++; }
    ```
    
3. **Применяйте `ReentrantLock` для гибкости**:
    
    ```java
    lock.tryLock(100, TimeUnit.MILLISECONDS);
    ```
    
4. **Используйте потокобезопасные коллекции**:
    
    ```java
    Map<String, Integer> map = new ConcurrentHashMap<>();
    ```
    
5. **Тестируйте синхронизацию**:
    
    ```java
    @Test
    void testSynchronizedCounter() throws InterruptedException {
        Counter counter = new Counter();
        Thread t1 = new Thread(() -> { for (int i = 0; i < 1000; i++) counter.increment(); });
        Thread t2 = new Thread(() -> { for (int i = 0; i < 1000; i++) counter.increment(); });
        t1.start(); t2.start();
        t1.join(); t2.join();
        assertEquals(2000, counter.getCount());
    }
    ```
    
6. **Избегайте deadlock**:
    - Устанавливайте единый порядок захвата.
    - Используйте `tryLock()`.

## 13. Заключение

Ключевое слово `synchronized` в Java обеспечивает взаимное исключение и видимость изменений, предотвращая гонки данных. Оно работает через мониторы объектов, поддерживая рекурсивные блокировки и оптимизации JVM (biased/lightweight locks). Однако неправильное использование может привести к deadlock. Современные альтернативы (`ReentrantLock`, `StampedLock`, атомарные классы) и виртуальные потоки (Java 21) предоставляют гибкость и производительность. Следование лучшим практикам и тщательное проектирование позволяют создавать надёжные потокобезопасные приложения.