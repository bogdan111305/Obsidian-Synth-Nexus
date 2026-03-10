---
title: "Модель памяти Java (JMM) и барьеры памяти"
tags: [java, jmm, memory-model, happens-before, volatile, barriers, concurrency]
updated: 2026-03-04
---

# Модель памяти Java (JMM) и барьеры памяти

> [!QUOTE] Суть
> **JMM** (Java Memory Model) определяет когда изменения одного потока видны другим. Гарантии строятся на **happens-before**. Без HB отношений — видимость не гарантирована (reordering, CPU cache). `volatile`, `synchronized`, `final`, `Thread.start/join` создают HB.

> [!INFO] Правила Happens-Before (критично знать на Senior)
> 1. **Программный порядок** — операции в одном потоке упорядочены
> 2. **Monitor unlock → lock** — `unlock` HB перед следующим `lock` того же монитора
> 3. **Volatile write → read** — запись в volatile HB перед любым последующим чтением
> 4. **Thread.start()** — всё до start() HB перед первой операцией нового потока
> 5. **Thread.join()** — все операции потока HB перед возвратом из join()
> 6. **Final fields** — инициализация final-поля в конструкторе HB любому читателю объекта

## Введение в Java Memory Model (JMM)

Без JMM два потока могут видеть разные значения одной переменной из-за:
- **CPU кэшей**: каждое ядро кэширует данные локально
- **Reordering**: компилятор/процессор переставляет инструкции для оптимизации
JMM задаёт правила когда изменения становятся видимы другим потокам.

Java Memory Model (JMM), введённая в Java 5 (JSR-133), балансирует между производительностью (позволяя оптимизации) и корректностью (гарантируя видимость). Вместо полной синхронизации всех операций JMM использует концепцию **happens-before**, которая определяет, когда изменения одного потока гарантированно видны другому. На низком уровне эти гарантии реализуются через **memory barriers** — специальные инструкции, вставляемые JIT-компилятором JVM.

## Отношение Happens-Before: Базовые Правила

Happens-before — это частичное упорядочивание операций между потоками. Если операция A happens-before операции B, то все изменения, сделанные в A (или до A), видимы в B. Это отношение транзитивно: если A happens-before B, а B happens-before C, то A happens-before C.

Базовые правила happens-before (на основе спецификации JMM):
1. **Программный порядок внутри потока**: Операции в одном потоке упорядочены в том порядке, в котором они написаны в коде. Например, если в коде A идёт перед B, то A happens-before B в этом потоке.
2. **Мониторы (synchronized)**: Разблокировка (`unlock`) монитора happens-before последующей блокировкой (`lock`) того же монитора в другом потоке. Это обеспечивает, что изменения внутри synchronized-блока видны после входа в него другим потоком.
3. **Volatile-переменные**: Запись в volatile-поле happens-before последующим чтениям того же поля. Это делает volatile мощным инструментом для флагов и простых синхронизаций.
4. **Потоки**: Вызов `Thread.start()` happens-before первой операции в новом потоке. Завершение потока happens-before успешным `Thread.join()`. Это гарантирует, что инициализация видна, а результаты — доступны после join.
5. **Final-поля**: Инициализация final-полей в конструкторе happens-before завершением конструктора. После публикации объекта (например, через присваивание ссылки) все final-поля корректно видны другим потокам без дополнительной синхронизации.

**Новое дополнение**: Happens-before также применяется к атомарным операциям из `java.util.concurrent.atomic`. Например, успешный `compareAndSet` в `AtomicInteger` создаёт happens-before между предыдущими операциями и последующими. Кроме того, в Java 8+ добавлены правила для `VarHandle` и `MemoryOrder`, которые позволяют более тонкий контроль над порядком.

Пример нарушения без happens-before:
```java
class NoSyncExample {
    int value = 0;

    void writer() {
        value = 42;  // Может не быть видно в reader из-за отсутствия синхронизации
    }

    void reader() {
        System.out.println(value);  // Может вывести 0
    }
}
```
Здесь нет happens-before между writer и reader, так что видимость не гарантирована.

## Memory Barriers: Что Это и Зачем

Memory barriers (памятные барьеры) — это низкоуровневые инструкции процессора, которые JIT-компилятор JVM вставляет для предотвращения нежелательного переупорядочивания и обеспечения видимости. Они решают проблемы кэширования и оптимизаций, заставляя процессор синхронизировать данные между локальными кэшами, общей кэш-линией и основной памятью.

Барьеры решают три задачи:
1. **Управление порядком**: Запрещают перестановку операций чтения (load) и записи (store).
2. **Видимость**: Сбрасывают (flush) изменения в основную память или инвалидируют (invalidate) кэши.
3. **Синхронизация**: Обеспечивают атомарность в многопроцессорных системах.

**Новое дополнение**: В современных JVM (например, HotSpot) барьеры оптимизированы под архитектуру процессора. На x86 они часто "бесплатны" для некоторых операций из-за сильной модели памяти процессора, но на ARM или RISC-V требуют явных инструкций.

### Основные Типы Memory Barriers

Существует четыре основных типа барьеров, соответствующих комбинациям load и store:
1. **LoadLoad**: Гарантирует, что все чтения до барьера завершены перед чтениями после. Предотвращает перестановку двух чтений.
2. **LoadStore**: Чтения до барьера завершены перед записями после. Обеспечивает, что данные прочитаны перед изменением.
3. **StoreLoad**: Записи до барьера завершены перед чтениями после. Самый дорогой барьер, так как требует полной синхронизации кэшей (full fence).
4. **StoreStore**: Записи до барьера завершены перед записями после. Предотвращает перестановку двух записей.

**Новое дополнение**: В Java 9+ введены `VarHandle` с режимами вроде `RELEASE` (StoreStore) и `ACQUIRE` (LoadLoad), позволяющие вручную управлять барьерами для производительности.

### Где Вставляются Memory Barriers

JVM вставляет барьеры автоматически в ключевых местах:
- **При входе в synchronized**: LoadLoad + LoadStore (обновить кэши перед блокировкой).
- **При выходе из synchronized**: StoreStore + StoreLoad (сбросить изменения после разблокировки).
- **После записи в volatile**: StoreStore + StoreLoad (завершить предыдущие записи и сделать видимыми).
- **Перед чтением volatile**: LoadLoad + LoadStore (завершить чтение перед последующими операциями).

**Новое дополнение**: В атомарных классах, как `AtomicReference`, метод `lazySet` использует только StoreStore для отложенной видимости, что быстрее, но слабее volatile.

## Связь JMM, Happens-Before и Memory Barriers в Java-Конструкциях

Memory barriers реализуют happens-before на практике. Рассмотрим ключевые конструкции.

### Volatile-Переменные

Volatile обеспечивает видимость и порядок без атомарности для сложных операций.
- Запись: StoreStore (завершить предыдущие store) + StoreLoad (сделать видимым для load).
- Чтение: LoadLoad + LoadStore.

Пример (расширенный):
```java
class VolatileBarrierExample {
    volatile boolean flag = false;
    int data = 0;

    void producer() {
        data = 100;  // Store, может быть переставлено без volatile
        flag = true; // Volatile store: StoreStore + StoreLoad
    }

    void consumer() {
        if (flag) {  // Volatile load: LoadLoad + LoadStore
            System.out.println(data);  // Гарантировано 100
        }
    }
}
```
Happens-before: Запись flag happens-before чтением flag, делая data видимым.

### Synchronized-Блоки

Synchronized создаёт монитор с барьерами:
- Lock: LoadLoad + LoadStore.
- Unlock: StoreStore + StoreLoad.

Пример с несколькими потоками:
```java
class Counter {
    private int value = 0;

    synchronized void increment() {
        value++;  // Барьеры обеспечивают атомарность и видимость
    }

    synchronized int getValue() {
        return value;
    }
}
```
Happens-before между unlock и lock гарантирует последовательность.

### Атомарные Операции и Final-Поля

`AtomicInteger` использует барьеры для CAS (Compare-And-Swap).
Final-поля: StoreStore после инициализации, чтобы объект не публиковался частично.

**Новое дополнение**: В immutable объектах final-поля позволяют безопасную публикацию без volatile, но для mutable частей нужна дополнительная синхронизация.

### Примеры Проблем и Решений

Без барьеров: Double-Checked Locking (DCL) без volatile может сломаться из-за частичной публикации объекта.
Решение: Добавить volatile к полю экземпляра.

Другой пример — Peterson's algorithm для mutual exclusion, где volatile обеспечивает правильный порядок.

## Барьеры Памяти и Аппаратное Обеспечение

На уровне CPU:
- x86: `mfence` для StoreLoad, но многие барьеры implicit.
- ARM: `dmb` (data memory barrier) для различных типов.

JVM адаптирует барьеры под платформу, обеспечивая портативность JMM.

**Новое дополнение**: В Java 21+ улучшена поддержка виртуальных потоков (Project Loom), где барьеры минимизированы для лёгких потоков.

## Senior Insights

### Полный граф happens-before с VarHandle

```java
import java.lang.invoke.*;

// VarHandle предоставляет тонкий контроль над happens-before:
class JMMVarHandleExample {
    int data = 0;
    private static final VarHandle DATA = /* ... */;

    // RELEASE semantics: все предшествующие операции видны ПОСЛЕ этой записи
    // (StoreStore + предыдущие stores)
    void publish(int value) {
        data = 99;                    // обычная запись (может быть переупорядочена выше)
        DATA.setRelease(this, value); // RELEASE: data=99 гарантированно видна до setRelease
    }

    // ACQUIRE semantics: все последующие операции видят записи ДО соответствующего RELEASE
    // (LoadLoad + последующие loads)
    int consume() {
        int v = (int) DATA.getAcquire(this); // ACQUIRE
        return v + data; // data=99 гарантированно видна здесь!
    }
    // ACQUIRE-RELEASE пара создаёт happens-before без полного volatile overhead!
}
```

### JMM Double-Checked Locking — разбор по байт-коду

```java
// BAD: Сломан без volatile (Java 5+)
class BrokenSingleton {
    private static BrokenSingleton instance;

    public static BrokenSingleton getInstance() {
        if (instance == null) {           // Проверка 1 (без синхронизации!)
            synchronized (BrokenSingleton.class) {
                if (instance == null) {
                    instance = new BrokenSingleton(); // ПРОБЛЕМА ЗДЕСЬ!
                    // JIT/CPU может переупорядочить:
                    // 1. Аллокировать память
                    // 2. Записать ссылку в instance ← другой поток видит не-null!
                    // 3. Вызвать конструктор       ← объект ещё не инициализирован!
                }
            }
        }
        return instance; // может вернуть частично инициализированный объект!
    }
}

// SENIOR WAY: volatile + happens-before для final publication
class CorrectSingleton {
    private static volatile CorrectSingleton instance; // volatile!

    public static CorrectSingleton getInstance() {
        if (instance == null) {
            synchronized (CorrectSingleton.class) {
                if (instance == null) {
                    instance = new CorrectSingleton();
                    // volatile write = StoreStore + StoreLoad барьер
                    // Конструктор ВСЕГДА завершается ДО volatile записи
                    // Другой поток читает volatile → LoadLoad барьер → видит полный объект
                }
            }
        }
        return instance;
    }
}

// ЛУЧШИЙ СПОСОБ (Java 5+): Initialization-on-demand holder
class BestSingleton {
    private static class Holder {
        // Класс загружается при первом обращении к Holder.INSTANCE
        // Загрузка класса — synchronized в ClassLoader → thread-safe!
        // Static final поле публикуется через happens-before ClassLoader lock
        static final BestSingleton INSTANCE = new BestSingleton();
    }

    public static BestSingleton getInstance() {
        return Holder.INSTANCE; // нет синхронизации, нет volatile overhead
    }
}
```

### Happens-before для final полей: безопасная публикация

```java
// Специальное правило JMM: запись в final поле в конструкторе
// happens-before любого чтения объекта ДРУГИМ потоком (если объект опубликован корректно)

class ImmutablePoint {
    final int x; // final!
    final int y;

    ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
        // JIT вставляет StoreStore барьер здесь (после записи в final поля)
        // Гарантирует: x и y видны ПОСЛЕ публикации объекта
    }
}

// Безопасная публикация — достаточно любого из:
// 1. Через volatile поле
// 2. Через synchronized блок
// 3. Через final поле другого объекта
// 4. Через static инициализатор класса

// ОПАСНО: "escaped reference" — объект опубликован до завершения конструктора
class UnsafePublication {
    private static UnsafePublication instance;

    UnsafePublication() {
        instance = this; // ЛОВУШКА! Другой поток видит instance до завершения конструктора
        // ... долгая инициализация ...
    }
}
```

## Связанные темы

- [[Java Memory Structure]] — Heap, Stack, Metaspace, GC
- [[Атомарность операций и Volatile]] — применение volatile на практике
- [[Synchronized]] — монитор и критические секции
- [[CAS и Unsafe]] — low-level атомарные операции
- [[Процессы и Потоки, Thread, Runnable, состояния потоков|Виртуальные потоки Java 21]] — влияние Loom на JMM