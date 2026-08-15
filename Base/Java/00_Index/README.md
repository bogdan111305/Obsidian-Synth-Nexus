# Java Knowledge Base Index

> Структурированная база знаний по Java для подготовки к Senior-интервью. 80+ заметок, охватывающих язык, JVM, GC, конкурентность и современные проекты (Loom, Valhalla, Panama).
>
> С [реорганизации по уровням](../../../Улучшение%20вики%20знаний/Схема%20реорганизации%20по%20уровням.md) структура папок — `01_Junior/ 02_Middle/ 03_Middle+/ 04_Senior/`, внутри каждой сохранены прежние тематические подпапки (`01_Language_Core`, `03_Concurrency_and_JMM` и т.д.). Ниже — та же карта заметок, но сгруппированная по уровню, с актуальными путями.

---

## 01_Junior

### 01_Language_Core
`Base/Java/01_Junior/01_Language_Core/`

| Категория | Файлы |
|-----------|-------|
| **OOP** | [[Интерфейсы]], [[Абстрактные классы]], [[Наследование]], [[Полиморфизм]], [[Инкапсуляция]], [[Enum]], [[Современные возможности — Records и Sealed]] (Records, Sealed Classes, Pattern Matching, Switch Expressions — всё в одной заметке) |
| **Типы** | [[Java Generic]], [[Приведение типов, widening conversion, narrowing conversion]], [[Java Reflection API]] |
| **Синтаксис** | [[Java Syntax Reference]], [[Java Static]], [[Java Exceptions]], [[Поля, конструкторы, this, инициализаторы]], [[Инициализация объектов с учётом наследования]] |
| **Современное** | [[Современные возможности — var и Text Blocks]] |

### 02_Collections_and_Streams/Interfaces Collection
`Base/Java/01_Junior/02_Collections_and_Streams/Interfaces Collection/`

- [[Интерфейс List]] — ArrayList, LinkedList, CopyOnWriteArrayList
- [[Интерфейс Map]] — HashMap, TreeMap, ConcurrentHashMap internals
- [[Интерфейс Set]] — HashSet, TreeSet, LinkedHashSet
- [[Интерфейсы Iterator и Iterable]] — fail-fast vs fail-safe

> Обзорной заметки про общую иерархию `Collection`/`Map` пока нет — см. пункт A.2 в [аудите вики](../../../Улучшение%20вики%20знаний/Аудит%20вики%20по%20Senior%20Java%20Roadmap.md).

### 05_IO_and_Networking
`Base/Java/01_Junior/05_IO_and_Networking/`

- [[Java String]] — компакт-строки, intern(), String pool
- [[Java Input-Output]] — InputStream, OutputStream, Reader/Writer, NIO2
- [[Java Date and Time]] — java.time, ZonedDateTime, Duration

---

## 02_Middle

### 02_Collections_and_Streams
`Base/Java/02_Middle/02_Collections_and_Streams/`

- [[Производительность коллекций]] — Big-O, benchmark
- [[Современные коллекции Java 9+]] — List.of, Map.copyOf
- [[Java Stream API & Functional Programming]] — Gatherers, collectors, parallel streams

### 03_Concurrency_and_JMM/Platform_Threads
`Base/Java/02_Middle/03_Concurrency_and_JMM/Platform_Threads/`

- [[Процессы и Потоки, Thread, Runnable, состояния потоков]]
- [[Атомарность операций и Volatile]]
- [[Java Monitor]]
- [[Прерывание потока в Java]]

### 03_Concurrency_and_JMM/Concurrent_Utils
`Base/Java/02_Middle/03_Concurrency_and_JMM/Concurrent_Utils/`

- [[ThreadPool, Future, Callable, Executors, CompletableFuture]]

---

## 03_Middle+

### 03_Concurrency_and_JMM
`Base/Java/03_Middle+/03_Concurrency_and_JMM/`

| Слой | Файлы |
|------|-------|
| **JMM** | [[Модель памяти Java (JMM) и барьеры памяти]] |
| **Concurrent Utils** | [[Atomic]], [[CAS и Unsafe]], [[Lock]], [[Synchronizers]] |
| **Modern Concurrency** | [[Virtual Threads — модель и архитектура]], [[Virtual Threads vs Platform Threads]], [[Carrier Threads и Pinning]], [[Happens-Before в контексте Virtual Threads]], [[Structured Concurrency]], [[Scoped Values (Java 21, JEP 446)]] |

### 04_JVM_Internals
`Base/Java/03_Middle+/04_JVM_Internals/`

- [[ClassLoaders]] — delegation model, TCCL, hot reload, Metaspace leaks
- [[JIT Compiler & Optimizations]] — Tiered, Escape Analysis, Deopt, OSR
- [[Java Bytecode — структура и опкоды]] — .class format, opcodes, constant pool
- [[invokedynamic — механика]] — bootstrap method, CallSite, лямбды
- [[MethodHandle & LambdaMetafactory]] — VarHandle, memory ordering
- [[Java Memory Structure]] — Heap zones, Metaspace, Code Cache, Direct
- [[Reference Types (Weak, Soft, Phantom)]] — ReferenceQueue, Cleaner

### 05_IO_and_Networking
`Base/Java/03_Middle+/05_IO_and_Networking/`

- [[NIO Networking]] — Selector, Channel, ByteBuffer, non-blocking IO
- [[Java Serialization and Deserialization]] — ObjectStream, Externalizable, Jackson

---

## 04_Senior

### 04_JVM_Internals
`Base/Java/04_Senior/04_JVM_Internals/`

- [[JVM Profiling & Observability]] — JFR, JMX, async-profiler
- [[Java Agents & Instrumentation API]] — JVMTI, Byte Buddy
- [[JVM Startup и AppCDS]] — class loading phases, CDS
- [[Project Leyden и AOT]] — AOTCache, Layered caches, vs GraalVM NI

### 06_GC_and_Memory
`Base/Java/04_Senior/06_GC_and_Memory/`

**GC Internals**
- [[GC Roots и достижимость объектов]] — tri-color marking, types of roots
- [[Safepoints и Stop-The-World]] — polling page, time-to-safepoint
- [[Write Barriers и Card Table]] — SATB, RSet, colored pointers
- [[Finalization и Cleaner API]] — finalize() проблемы, PhantomReference

**GC Алгоритмы**
- [[G1GC — архитектура и tuning]] — regions, IHOP, Mixed GC, Evacuation Failure
- [[ZGC и Generational ZGC]] — colored pointers, load barrier, <1 мс паузы
- [[Shenandoah GC]] — Brooks pointers, concurrent relocation
- [[Parallel и Serial GC]] — copying collector, mark-compact, throughput
- [[Epsilon GC]] — no-op GC, benchmarking, short-lived jobs

**GC Tuning**
- [[JVM флаги для GC]] — полный справочник флагов
- [[Анализ GC логов — JFR, GCEasy]] — unified logging, JFR events, GCEasy
- [[GC и латентность — практика]] — allocation rate, promotion, off-heap

### 07_Modern_Java_Platform
`Base/Java/04_Senior/07_Modern_Java_Platform/`

**Project Loom (Java 21)**
- [[Virtual Threads — модель и архитектура]]
- [[Carrier Threads и Pinning]]
- [[Structured Concurrency]]

**Project Valhalla**
- [[Value Classes и Primitive Classes]] — flat memory layout, zero boxing
- [[Object Identity — что меняется]] — synchronized, ==, IdentityObject
- [[Valhalla и коллекции]] — specialized generics, boxing overhead

**Project Panama (Java 22)**
- [[Foreign Function and Memory API]] — MemorySegment, Arena, Linker
- [[Panama vs Unsafe vs JNI]] — сравнение подходов
- [[Vector API — SIMD в Java]] — AVX, SPECIES, dot product

---

## Быстрые ссылки для интервью
- → [[Interview Cheatsheet — Senior Java]] — шпаргалка по всем темам
- → [[JEP Reference Map]] — JEP по темам и версиям Java
