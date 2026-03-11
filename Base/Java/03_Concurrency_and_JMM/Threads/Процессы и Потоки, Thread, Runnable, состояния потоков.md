---
title: "Java — Процессы и Потоки, Thread, Runnable, Virtual Threads"
tags: [java, concurrency, threads, runnable, thread-states, virtual-threads, java21, project-loom]
updated: 2026-03-11
---

# Процессы и Потоки в Java

> [!QUOTE] Суть
> **Поток** — единица выполнения внутри JVM-процесса. Состояния: NEW → RUNNABLE → BLOCKED/WAITING/TIMED_WAITING → TERMINATED. Platform threads — нативные OS потоки (дорогие, ~1-2MB stack). **Virtual threads** (Java 21) — легковесные (~KB), миллионы одновременно, маппятся на OS threads автоматически.

**Процессы** и **потоки** — фундаментальные концепции для выполнения программ в операционной системе и Java. Процессы представляют собой независимые программы, а потоки — единицы выполнения внутри процесса, позволяющие выполнять задачи параллельно.

## 1. Процессы

**Процесс** — это запущенная программа, которая выполняется в операционной системе.

- **Характеристики**:
    
    - Обладает собственным **адресным пространством памяти**, включающим:
        - **Код программы**: Исполняемые инструкции.
        - **Данные**: Переменные, куча (heap), стек (stack).
        - **Ресурсы**: Файлы, дескрипторы, сетевые соединения.
    - Процессы полностью **изолированы** друг от друга, что предотвращает доступ к памяти другого процесса.
    - Создание процесса требует значительных ресурсов (память, время).
- **Пример**:
    
    - Запуск JVM (`java MyApp`) создаёт процесс, содержащий байт-код, кучу, стек и ресурсы.

## 2. Потоки

**Поток (Thread)** — это единица выполнения внутри процесса, позволяющая выполнять задачи параллельно или псевдопараллельно.

- **Характеристики**:
    
    - Потоки одного процесса **разделяют общую кучу** (heap) для хранения объектов.
    - Каждый поток имеет собственный **стек вызовов** (stack) для локальных переменных и вызовов методов.
    - Потоки позволяют выполнять несколько задач одновременно, эффективно используя многоядерные процессоры.
- **Параллельность vs. псевдопараллельность**:
    
    - **Параллельность**: Потоки выполняются одновременно на разных ядрах CPU.
    - **Псевдопараллельность**: На одном ядре ОС быстро переключает потоки (таймшаринг), создавая иллюзию одновременного выполнения.

|Условие|Поведение|
|---|---|
|Потоков ≤ ядер CPU|Реальная параллельность|
|Потоков > ядер CPU|Таймшаринг (псевдопараллельность)|

## 3. Создание потоков в Java

Java предоставляет несколько способов создания потоков:

### 3.1. Наследование от `Thread`

Класс `Thread` напрямую представляет поток. Переопределяется метод `run()` для выполнения задачи.

**Пример**:

```java
class MyThread extends Thread {
    public void run() {
        System.out.println("Поток " + Thread.currentThread().getName() + " запущен");
    }
}

public class Main {
    public static void main(String[] args) {
        MyThread thread = new MyThread();
        thread.start(); // Запуск нового потока
    }
}
```

**Особенности**:

- Вызов `start()` создаёт новый поток и вызывает `run()` в нём.
- Прямой вызов `run()` выполняет метод в текущем потоке, без создания нового.

### 3.2. Реализация интерфейса `Runnable`

`Runnable` — функциональный интерфейс с методом `run()`, разделяющий логику задачи и управление потоком.

**Пример**:

```java
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("Поток " + Thread.currentThread().getName() + " запущен");
    }
}

public class Main {
    public static void main(String[] args) {
        Thread thread = new Thread(new MyRunnable());
        thread.start();
    }
}
```

**Преимущества**:

- Позволяет использовать один `Runnable` для нескольких потоков.
- Более гибко, так как класс может наследовать другой класс.

### 3.3. Лямбда-выражения (Java 8+)

Лямбда-выражения упрощают создание `Runnable`.

**Пример**:

```java
public class Main {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            System.out.println("Поток " + Thread.currentThread().getName() + " запущен");
        });
        thread.start();
    }
}
```

**Преимущества**:

- Компактный и читаемый код.
- Идеально для простых задач.

### 3.4. Через `ExecutorService` (пулы потоков)

`ExecutorService` управляет пулом потоков, переиспользуя их для выполнения задач.

**Пример**:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Main {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        executor.submit(() -> {
            System.out.println("Поток " + Thread.currentThread().getName() + " запущен");
        });
        executor.shutdown();
    }
}
```

**Преимущества**:

- Переиспользование потоков снижает накладные расходы.
- Управление очередями задач и масштабируемостью.

### 3.5. Виртуальные потоки (Java 21+, Project Loom)

Виртуальные потоки — лёгкие потоки, управляемые JVM, а не ОС.

**Пример**:

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Thread virtualThread = Thread.ofVirtual().start(() -> {
            System.out.println("Виртуальный поток " + Thread.currentThread().getName() + " запущен");
        });
        virtualThread.join();
    }
}
```

**Особенности**:

- Тысячи виртуальных потоков могут работать на одном ядре.
- Подходят для задач с высокой конкуренцией (например, I/O-операции).

## 4. Жизненный цикл и состояния потоков

Java-потоки проходят через несколько состояний, определённых в `Thread.State`:

|Состояние|Описание|
|---|---|
|**NEW**|Поток создан, но `start()` не вызван.|
|**RUNNABLE**|Поток готов к выполнению или выполняется, но может ждать CPU.|
|**BLOCKED**|Поток ожидает монитор (`synchronized`) для входа в критическую секцию.|
|**WAITING**|Поток ожидает уведомления без таймаута (`wait()`, `join()`).|
|**TIMED_WAITING**|Поток ожидает с таймаутом (`sleep()`, `wait(timeout)`).|
|**TERMINATED**|Поток завершил выполнение (`run()` закончился).|

### 4.1. Подробное описание состояний

- **NEW**:
    
    - Поток создан, но не запущен.
    - Можно настроить имя, приоритет (`setPriority()`).
    
    ```java
    Thread t = new Thread(() -> {});
    System.out.println(t.getState()); // NEW
    ```
    
- **RUNNABLE**:
    
    - Поток готов или выполняется.
    - ОС решает, когда выделить CPU.
- **BLOCKED**:
    
    - Поток ждёт монитор для `synchronized` блока.
    
    ```java
    synchronized (lock) { // BLOCKED, если монитор занят
        // Критическая секция
    }
    ```
    
- **WAITING**:
    
    - Поток ожидает сигнала (`wait()`, `join()`, `LockSupport.park()`).
    - Возвращается в `RUNNABLE` после `notify()` или `unpark()`.
- **TIMED_WAITING**:
    
    - Поток ожидает с таймаутом (`sleep()`, `wait(timeout)`, `join(timeout)`).
    
    ```java
    Thread.sleep(1000); // TIMED_WAITING
    ```
    
- **TERMINATED**:
    
    - Поток завершил выполнение.
    - Нельзя перезапустить.

**Пример с состояниями**:

```java
public class ThreadStateExample {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            try {
                Thread.sleep(1000); // TIMED_WAITING
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println("State after creation: " + t.getState()); // NEW
        t.start();
        System.out.println("State after start: " + t.getState()); // RUNNABLE
        Thread.sleep(100);
        System.out.println("State while sleeping: " + t.getState()); // TIMED_WAITING
        t.join();
        System.out.println("State after completion: " + t.getState()); // TERMINATED
    }
}
```

## 5. Методы управления потоками

- `start()`: Запускает поток, переводя его в `RUNNABLE`.
- `run()`: Содержит код задачи, вызывается автоматически после `start()`.
- `sleep(milliseconds)`: Переводит в `TIMED_WAITING`.
- `join()`: Ждёт завершения потока (`WAITING` или `TIMED_WAITING`).
- `interrupt()`: Устанавливает флаг прерывания, полезен для выхода из `sleep()` или `wait()`.
- `isAlive()`: Проверяет, находится ли поток в состоянии `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING` или `TIMED_WAITING`.
- `setPriority(int)`: Устанавливает приоритет (1–10), но эффект зависит от ОС.

## 6. Планирование потоков и переключение контекста

### 6.1. Планировщик (Scheduler)

- Планировщик ОС распределяет время CPU между потоками.
- JVM создаёт потоки ОС через нативные API (например, `pthreads` на Linux).
- Модели планирования:
    - **Приоритетное**: Потоки с высоким приоритетом получают больше времени.
    - **Круговое (Round-robin)**: Равные интервалы времени для каждого потока.
    - **Вытесняющее (Preemptive)**: Потоки с низким приоритетом прерываются ради более важных.

### 6.2. Переключение контекста

- **Переключение контекста**: Сохранение состояния одного потока и восстановление другого.
- **Состояние включает**:
    - Регистры процессора.
    - Указатель стека.
    - Локальные данные потока.
- Накладные расходы: Частые переключения снижают производительность.

### 6.3. JVM и ОС

- JVM обёртывает потоки ОС, используя нативные API.
- При вызове `thread.start()` JVM создаёт поток ОС и регистрирует его в планировщике.
- Приоритеты (`setPriority()`) передаются ОС, но их интерпретация платформозависима.

**Байт-код (упрощённый)**:

```java
invokestatic java/lang/Thread.start()V
// Вызывает native-метод для создания потока ОС
```

## 7. Отличия процессов и потоков

|Характеристика|Процесс|Поток|
|---|---|---|
|**Память**|Собственное адресное пространство|Общая куча, отдельный стек|
|**Создание**|Медленное, ресурсоёмкое|Быстрое, в рамках процесса|
|**Изоляция**|Полная изоляция|Общий доступ к памяти процесса|
|**Переключение контекста**|Дорогостоящее|Менее затратное|
|**Использование**|Независимые программы|Параллельные задачи в программе|

## 8. Зачем нужны потоки?

- **Производительность**: Параллельное выполнение задач на многоядерных CPU.
- **Асинхронность**: Обработка событий, I/O без блокировки основного потока.
- **Эффективность**: Использование ресурсов процессора для интенсивных задач (например, вычисления, серверные запросы).

## 9. Современные возможности (Java 21+)

### 9.1. Виртуальные потоки (Project Loom)

Виртуальные потоки — лёгкие потоки, управляемые JVM, а не ОС.

**Пример**:

```java
import java.util.concurrent.Executors;

public class Main {
    public static void main(String[] args) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            executor.submit(() -> {
                System.out.println("Виртуальный поток " + Thread.currentThread().getName());
            });
        }
    }
}
```

**Преимущества**:

- Масштабируемость: Поддержка тысяч потоков с минимальными затратами.
- Простота: Упрощает асинхронный код, заменяя сложные пулы потоков.

### 9.2. StructuredTaskScope

`StructuredTaskScope` (Java 21, JEP 453) обеспечивает структурированную конкуренцию.

**Пример**:

```java
import java.util.concurrent.StructuredTaskScope;

public class Main {
    public static void main(String[] args) throws Exception {
        try (var scope = new StructuredTaskScope<String>()) {
            var task1 = scope.fork(() -> "Task 1 result");
            var task2 = scope.fork(() -> "Task 2 result");
            scope.join();
            System.out.println(task1.get() + ", " + task2.get());
        }
    }
}
```

**Преимущества**:

- Управление группой задач с обработкой ошибок и отменой.
- Упрощает параллельное выполнение.

## 10. JVM-реализация

- **Потоки ОС**:
    - JVM использует нативные потоки (например, `pthreads` на Linux).
    - Метод `Thread.start()` вызывает нативный код для создания потока.
- **Память**:
    - **Куча**: Общая для всех потоков, хранит объекты.
    - **Стек**: Отдельный для каждого потока, хранит локальные переменные и вызовы методов.
- **Байт-код**:
    
    ```java
    invokestatic java/lang/Thread.start()V
    // Вызывает native-метод start0()
    ```
    
- **Виртуальные потоки**:
    - JVM использует **continuation** для управления, минимизируя накладные расходы.

## 11. Практическое применение

- **Java EE / Jakarta EE**:
    - Потоки для обработки HTTP-запросов в сервлетах.
    - Пул потоков в EJB для асинхронных задач.
- **Spring**:
    - `@Async` методы используют пулы потоков (`TaskExecutor`).
- **GUI**:
    - Swing/JavaFX используют потоки для обработки событий (`Event Dispatch Thread`).
- **Коллекции**:
    - Параллельная обработка в `parallelStream()`.

**Пример (Spring)**:

```java
@Service
public class AsyncService {
    @Async
    public CompletableFuture<String> process() {
        return CompletableFuture.completedFuture("Processed in " + Thread.currentThread().getName());
    }
}
```

## 12. Подводные камни

1. **Прямой вызов `run()`**:
    
    ```java
    thread.run(); // Выполняется в текущем потоке
    ```
    
    **Решение**: Всегда используйте `start()`.
    
2. **Утечки ресурсов**:
    
    - Незакрытые пулы потоков (`ExecutorService`) могут удерживать ресурсы.
    - **Решение**: Вызывайте `shutdown()` или используйте `try-with-resources`.
3. **Неправильные приоритеты**:
    
    - Приоритеты (`setPriority()`) не гарантируют поведения, так как зависят от ОС.
    - **Решение**: Полагайтесь на планировщик ОС.
4. **Race conditions**:
    
    - Общий доступ к данным без синхронизации.
    - **Решение**: Используйте `synchronized`, `Lock` или `Atomic` классы.
5. **Долгие операции в виртуальных потоках**:
    
    - Виртуальные потоки не подходят для CPU-интенсивных задач.
    - **Решение**: Используйте платформенные потоки для вычислений.

## 13. Производительность

- **Переключение контекста**:
    - Затраты на сохранение/восстановление состояния (регистры, стек).
    - Виртуальные потоки минимизируют эти расходы.
- **Пулы потоков**:
    - `ExecutorService` снижает затраты на создание потоков.
- **Виртуальные потоки**:
    - Эффективны для I/O-задач, но не для CPU-интенсивных.
- **Рекомендации**:
    - Используйте `ExecutorService` для большинства задач.
    - Применяйте виртуальные потоки для высококонкурентных приложений.

## 14. Лучшие практики

1. **Предпочитайте `Runnable` и лямбда-выражения**:
    
    ```java
    Thread thread = new Thread(() -> {});
    ```
    
2. **Используйте пулы потоков**:
    
    ```java
    ExecutorService executor = Executors.newFixedThreadPool(4);
    ```
    
3. **Применяйте виртуальные потоки для I/O**:
    
    ```java
    Thread.ofVirtual().start(() -> {});
    ```
    
4. **Управляйте ресурсами**:
    
    ```java
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        executor.submit(() -> {});
    }
    ```
    
5. **Обрабатывайте прерывания**:
    
    ```java
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    ```
    
6. **Тестируйте состояния**:
    
    ```java
    @Test
    void testThreadState() throws InterruptedException {
        Thread t = new Thread(() -> {
            try { Thread.sleep(100); } catch (InterruptedException e) {}
        });
        assertEquals(Thread.State.NEW, t.getState());
        t.start();
        Thread.sleep(10);
        assertEquals(Thread.State.TIMED_WAITING, t.getState());
    }
    ```
    

## 15. Virtual Threads: внутреннее устройство (Senior)

### Архитектура Project Loom

> [!INFO] Виртуальный поток = **Continuation + Scheduler + Carrier Thread**

```
Platform Thread (OS Thread):
  JVM Thread ──────── OS Thread (kernel) ──── CPU Core
  Stack: ~1-2 MB в нативной памяти ОС
  Создание: ~1 ms, Переключение контекста: ~1-10 µs (kernel syscall)
  Максимум: ~thousands (ограничение ОС/memory)

Virtual Thread (Project Loom):
  VirtualThread ──────── Continuation ──── ForkJoinPool Worker (Carrier)
  Stack: ~KB в Java Heap (serialized continuation)
  Создание: ~1 µs, "Переключение": ~100-300 ns (JVM, no syscall)
  Максимум: millions (ограничение только Heap)

Continuation — это capture и restore стека вызовов:
  При блокирующей операции (I/O, sleep, await...):
  1. VirtualThread.yield() — стек сериализуется в Heap
  2. Carrier thread освобождается для другого виртуального потока
  3. Когда I/O завершён — continuation восстанавливается на (любом) carrier
```

```java
// Настройка scheduler (ForkJoinPool) — по умолчанию:
// Parallelism = Runtime.getRuntime().availableProcessors()
// Каждый carrier thread = platform thread в ForkJoinPool

// Кастомный scheduler (редко нужен):
ExecutorService scheduler = Executors.newFixedThreadPool(4);
Thread.ofVirtual()
    .scheduler(scheduler)  // использовать кастомный scheduler
    .start(task);

// Проверить является ли поток виртуальным:
Thread.currentThread().isVirtual(); // true/false

// Виртуальный поток НЕ имеет смысла в newFixedThreadPool:
// BAD: пул из 100 виртуальных потоков — потеря преимущества
Executors.newFixedThreadPool(100); // всё ещё 100 platform threads

// GOOD: один виртуальный поток на задачу:
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Request req : requests) {
        exec.submit(() -> handleRequest(req)); // каждый запрос = свой virtual thread
    }
}
```

### Pinning — главная ловушка виртуальных потоков

> [!WARNING] Pinning блокирует carrier thread — нивелирует всё преимущество Virtual Threads

Виртуальный поток **прикрепляется (pinned)** к carrier thread и не может быть снят при:
1. **`synchronized` блоке** — carrier thread блокируется вместе с virtual thread
2. **Native method / JNI** — JVM не может demount continuation в нативном коде

```java
// ПЛОХО: Virtual thread pinned на synchronized
class PinnedExample {
    synchronized void slowDbQuery() { // PINNING!
        // ... 200ms запрос к БД ...
        // Всё это время carrier thread заблокирован
        // Один carrier = один поток ОС = wasteful
    }
}

// ХОРОШО: ReentrantLock вместо synchronized
class UnpinnedExample {
    private final ReentrantLock lock = new ReentrantLock();

    void slowDbQuery() {
        lock.lock();
        try {
            // ... 200ms запрос к БД ...
            // Virtual thread yields на I/O → carrier освобождается
            // ReentrantLock использует LockSupport.park() → virtual thread unmounts!
        } finally {
            lock.unlock();
        }
    }
}

// Диагностика pinning:
// JVM флаг: -Djdk.tracePinnedThreads=full (или =short)
// Выведет stack trace при каждом pinning событии

// JFR Event: jdk.VirtualThreadPinned
// jcmd <pid> JFR.start duration=30s filename=pinning.jfr
// В JMC: event browser → Virtual Thread Pinned
```

**Java 24 Update**: В Java 24 (JEP 491) `synchronized` был улучшен — виртуальные потоки теперь могут размонтироваться даже внутри `synchronized` блока в большинстве случаев. Pinning остаётся только для JNI.

### Platform Thread vs Virtual Thread: сравнение

| Характеристика | Platform Thread | Virtual Thread |
|----------------|----------------|----------------|
| Управление | ОС (kernel-level) | JVM (user-space) |
| Stack Memory | ~1–2 MB (нативная) | ~KB (Java Heap) |
| Создание | ~1 ms | ~1–10 µs |
| Переключение контекста | ~1–10 µs (syscall) | ~100–300 ns (JVM) |
| Максимальное количество | ~thousands | millions |
| CPU-bound задачи | ✅ Оптимален | ❌ Нет преимущества |
| I/O-bound задачи | ❌ Блокирует OS thread | ✅ Unmounts при блокировке |
| ThreadLocal | ✅ Поддерживается | ⚠️ Медленно (копирование) → лучше ScopedValue |
| `synchronized` | ✅ Нет проблем | ⚠️ Pinning (до Java 24) |
| Debugging/profiling | ✅ Хорошая поддержка | ⚠️ Нужен обновлённый profiler |

### Когда НЕ использовать Virtual Threads

```java
// 1. CPU-intensive задачи — виртуальных потоков будет много,
//    но все они конкурируют за те же N carrier threads:
// BAD:
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    exec.submit(() -> heavyComputation()); // нет I/O, нет yield, нет выгоды
}
// GOOD:
ForkJoinPool.commonPool().submit(() -> heavyComputation());

// 2. Библиотеки с pinning-кодом — JDBC drivers (до обновления),
//    старые connection pools, JNI-based код

// 3. ThreadLocal с большими объектами — при millions virtual threads
//    каждый копирует ThreadLocal → много памяти
//    Замена: ScopedValue (Java 21+)
```