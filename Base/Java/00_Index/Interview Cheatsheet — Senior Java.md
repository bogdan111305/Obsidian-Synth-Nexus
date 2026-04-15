# Interview Cheatsheet — Senior Java

> Шпаргалка для Senior Java интервью. Краткие ответы на топ-вопросы с указателями на детальные заметки.

---

## JVM & Memory

**Структура памяти JVM** → [[Java Memory Structure]]
- Heap (Eden→S0/S1→Old), Metaspace, Stack (per-thread), Code Cache, Direct Memory
- `StackOverflowError` = stack переполнен | `OutOfMemoryError` = heap/metaspace/direct

**ClassLoaders** → [[ClassLoaders]]
- Bootstrap → Extension → App (parent delegation). `loadClass` сначала parent.
- TCCL (Thread Context ClassLoader) — обход делегации для frameworks
- Metaspace leak = ClassLoader жив → все его классы в памяти

**JIT** → [[JIT Compiler & Optimizations]]
- C1 (быстрая компиляция) → C2 (агрессивные оптимизации). Tiered: уровни 0-4
- Escape Analysis → scalar replacement, lock elimination
- Deoptimization: нарушение предположения → trap → интерпретатор → recompile
- Мегаморфный call site: >2 типов → JIT не инлайнит

**Safepoints** → [[Safepoints и Stop-The-World]]
- Poling page SIGSEGV → все потоки приостановлены
- `Total pause = time-to-safepoint + actual GC work`
- Safepoint-hostile loop = `-XX:+UseCountedLoopSafepoints`

---

## GC

**GC Roots** → [[GC Roots и достижимость объектов]]
- Thread stacks, static fields, JNI global refs, monitors, JVM internals

**Write Barriers** → [[Write Barriers и Card Table]]
- Card Table: heap → 512-байтные карточки, dirty при записи ссылки
- G1 SATB: запоминает старое значение поля при concurrent marking
- ZGC Load barrier + colored pointers (4 бита в pointer)

**G1GC** → [[G1GC — архитектура и tuning]]
- Regions (1-32 МБ): Eden/Survivor/Old/Humongous
- IHOP (45%) → Concurrent Marking → Mixed GC
- Evacuation Failure → Full GC (Serial!) = катастрофа
- Ключи: `-XX:MaxGCPauseMillis=200`, `-XX:InitiatingHeapOccupancyPercent=45`

**ZGC** → [[ZGC и Generational ZGC]]
- Colored pointers (42 бит адрес + 4 бит GC state)
- Load barrier при каждом чтении ссылки
- 3 STW паузы < 1 мс каждая, остальное concurrent
- Generational ZGC (Java 21): `-XX:+UseZGC -XX:+ZGenerational`

**Выбор GC:**
- Latency <10 мс → ZGC | Стандарт → G1 | Batch → Parallel | Benchmark → Epsilon

**Tuning** → [[JVM флаги для GC]], [[GC и латентность — практика]]
- `Xms=Xmx`, `MaxMetaspaceSize` всегда устанавливать
- Container: `-XX:MaxRAMPercentage=75` вместо `-Xmx`

---

## Concurrency

**JMM** → [[Модель памяти Java (JMM) и барьеры памяти]]
- Happens-Before: volatile write → read, unlock → lock, Thread.start/join
- volatile: visibility + ordering. Не атомарность compound actions
- VarHandle: Plain/Opaque/Acquire-Release/Volatile ordering modes

**Synchronized** → [[Java Monitor]]
- Object header Mark Word: thin lock → heavyweight Monitor
- Lock inflation. Biased locking удалён Java 15+
- Не на null, не на value objects (Valhalla)

**volatile vs AtomicXxx vs synchronized:**
- volatile: single write/read visibility
- Atomic: CAS на single variable (lock-free)
- synchronized: compound actions, multiple variables

**Lock** → [[Lock]]
- AQS CLH queue, Node.waitStatus (SIGNAL/CANCELLED/CONDITION/PROPAGATE)
- Fair vs Unfair: Unfair быстрее на 20-30% (меньше context switch)
- StampedLock: non-reentrant! Нет Condition. Optimistic read pattern.

**Virtual Threads** → [[Virtual Threads — модель и архитектура]], [[Carrier Threads и Pinning]]
- Continuation (stack на heap) + ForkJoinPool scheduler
- Pinning causes: synchronized block, native frame
- Диагностика: `-Djdk.tracePinnedThreads=full`
- Не подходят для CPU-bound, только IO-bound

**Structured Concurrency** → [[Structured Concurrency]]
- `StructuredTaskScope.ShutdownOnFailure` — отменяет всех при первой ошибке
- `ShutdownOnSuccess` — отменяет всех при первом успехе (первый ответ)

---

## Language

**Generics** → [[Java Generic]]
- Type Erasure: `List<T>` → `List<Object>` в байткоде
- PECS: Producer extends, Consumer super
- Wildcard Capture, Super Type Token (TypeReference)

**Records** → [[Records]]
- Canonical constructor, compact constructor, custom accessors
- Реализует equals/hashCode/toString по полям
- Immutable value object, можно implements interface

**Sealed Classes** → [[Sealed Classes]]
- `sealed class Shape permits Circle, Rectangle`
- Exhaustive switch без default (компилятор проверяет)
- JIT: deoptimization не нужна — закрытая иерархия

**Pattern Matching** → [[Pattern Matching]], [[Современные возможности — Switch и Pattern Matching]]
- `instanceof String s &&` → type test + binding
- Switch patterns: guards `when`, nested patterns, record deconstruction

---

## Modern Java (Java 21+)

**Stream Gatherers** (Java 22-24) → [[Java Stream API & Functional Programming]]
- `Stream.gather(Gatherer)` — stateful intermediate operations
- `Gatherers.windowFixed(n)`, `windowSliding(n)`, `fold()`

**FFM API** (Java 22) → [[Foreign Function and Memory API]]
- `MemorySegment`, `Arena` — безопасная работа с off-heap памятью
- `Linker.nativeLinker().downcallHandle()` — вызов C функций без JNI
- Arena scope = lifetime management

**Project Valhalla** (Preview) → [[Value Classes и Primitive Classes]]
- `value class Point { int x, y; }` — no identity, flat memory layout
- Нет synchronized, нет null по умолчанию, нет наследования
- Цель: Integer/Long как value types, zero boxing в коллекциях

---

## Топ-10 вопросов с подводными камнями

1. **Почему double-checked locking требует volatile?**
   → Без volatile: JIT может переупорядочить инициализацию и присвоение → другой поток увидит non-null reference на не-инициализированный объект

2. **Чем отличается HashMap от ConcurrentHashMap?**
   → CHM: 16 buckets (segments), CAS на чтение, synchronized только на bucket при write, size() approximate

3. **Что происходит при переполнении Metaspace?**
   → Full GC для unloading классов → если не помогает → `OutOfMemoryError: Metaspace`

4. **Зачем finalize() deprecated?**
   → 2 GC цикла, воскрешение возможно, Finalizer Thread single и может зависнуть, GC не гарантирует вызов

5. **Почему ArrayList лучше LinkedList для большинства задач?**
   → Cache locality: ArrayList[i] = один cache line. LinkedList = random pointer chasing

6. **Как работает String.intern()?**
   → Помещает в String Pool (heap Java 8+). `intern()` дорог при первом вызове. Не злоупотреблять.

7. **Что такое Evacuation Failure в G1?**
   → G1 не нашёл региона для evacuation → откат к Serial Full GC (~секунды). Признак: heap слишком мал или IHOP слишком высокий

8. **Зачем Virtual Threads не подходят для CPU-bound?**
   → 1 carrier thread = 1 CPU. N virtual threads на 1 carrier = sequential execution. Не параллелизм.

9. **Что такое happens-before для volatile?**
   → volatile write X happens-before любого volatile read X, который видит это значение

10. **Почему synchronized на String опасно?**
    → String pool: разные части кода могут synchronize на одном объекте → неожиданный deadlock

---

## Флаги для Production

```bash
-Xms4g -Xmx4g                          # fixed heap
-XX:+UseG1GC                            # explicit GC choice
-XX:MaxGCPauseMillis=200               # pause target
-XX:MaxMetaspaceSize=512m              # prevent Metaspace OOM
-XX:+HeapDumpOnOutOfMemoryError        # dump on OOM
-XX:HeapDumpPath=/var/log/app/         # dump location
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
-Xlog:safepoint=info:file=/var/log/app/safepoint.log:time
-XX:StartFlightRecording=maxage=1h,maxsize=500m,filename=/var/log/app/jfr/
```
