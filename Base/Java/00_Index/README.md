# Java Knowledge Base Index

> Структурированная база знаний по Java для подготовки к Senior-интервью. 80+ заметок, охватывающих язык, JVM, GC, конкурентность и современные проекты (Loom, Valhalla, Panama).

---

## 01 — Ядро языка
`Base/Java/01_Language_Core/`

| Категория | Файлы |
|-----------|-------|
| **OOP** | [[Интерфейсы]], [[Абстрактные классы]], [[Наследование]], [[Полиморфизм]], [[Инкапсуляция]], [[Enum]], [[Records]], [[Sealed Classes]], [[Pattern Matching]] |
| **Типы** | [[Java Generic]], [[Приведение типов, widening conversion, narrowing conversion]], [[Java Reflection API]] |
| **Синтаксис** | [[Java Syntax Reference]], [[Java Static]], [[Java Exceptions]], [[Поля, конструкторы, this, инициализаторы]], [[Инициализация объектов с учётом наследования]] |
| **Современное** | [[Современные возможности — Records и Sealed]], [[Современные возможности — Switch и Pattern Matching]], [[Современные возможности — var и Text Blocks]] |

## 02 — Коллекции и Streams
`Base/Java/02_Collections_and_Streams/`

- [[Общая иерархия коллекций]] — Collection, List, Set, Map иерархия
- [[Интерфейс List]] — ArrayList, LinkedList, CopyOnWriteArrayList
- [[Интерфейс Map]] — HashMap, TreeMap, ConcurrentHashMap internals
- [[Интерфейс Set]] — HashSet, TreeSet, LinkedHashSet
- [[Интерфейсы Iterator и Iterable]] — fail-fast vs fail-safe
- [[Производительность коллекций]] — Big-O, benchmark
- [[Современные коллекции Java 9+]] — List.of, Map.copyOf
- [[Java Stream API & Functional Programming]] — Gatherers, collectors, parallel streams

## 03 — Конкурентность и JMM
`Base/Java/03_Concurrency_and_JMM/`

| Слой | Файлы |
|------|-------|
| **JMM** | [[Модель памяти Java (JMM) и барьеры памяти]] |
| **Platform Threads** | [[Процессы и Потоки, Thread, Runnable, состояния потоков]], [[Атомарность операций и Volatile]], [[Java Monitor]], [[Прерывание потока в Java]] |
| **Concurrent Utils** | [[Atomic]], [[CAS и Unsafe]], [[Lock]], [[Synchronizers]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]] |
| **Modern Concurrency** | [[Virtual Threads — модель и архитектура]], [[Virtual Threads vs Platform Threads]], [[Carrier Threads и Pinning]], [[Happens-Before в контексте Virtual Threads]], [[Structured Concurrency]], [[Scoped Values (Java 21, JEP 446)]] |

## 04 — JVM Internals
`Base/Java/04_JVM_Internals/`

- [[ClassLoaders]] — delegation model, TCCL, hot reload, Metaspace leaks
- [[JIT Compiler & Optimizations]] — Tiered, Escape Analysis, Deopt, OSR
- [[Java Bytecode — структура и опкоды]] — .class format, opcodes, constant pool
- [[invokedynamic — механика]] — bootstrap method, CallSite, лямбды
- [[MethodHandle & LambdaMetafactory]] — VarHandle, memory ordering
- [[Java Memory Structure]] — Heap zones, Metaspace, Code Cache, Direct
- [[Reference Types (Weak, Soft, Phantom)]] — ReferenceQueue, Cleaner
- [[JVM Profiling & Observability]] — JFR, JMX, async-profiler
- [[Java Agents & Instrumentation API]] — JVMTI, Byte Buddy
- [[JVM Startup и AppCDS]] — class loading phases, CDS
- [[Project Leyden и AOT]] — AOTCache, Layered caches, vs GraalVM NI

## 05 — IO и Networking
`Base/Java/05_IO_and_Networking/`

- [[Java String]] — компакт-строки, intern(), String pool
- [[Java Input-Output]] — InputStream, OutputStream, Reader/Writer, NIO2
- [[NIO Networking]] — Selector, Channel, ByteBuffer, non-blocking IO
- [[Java Serialization and Deserialization]] — ObjectStream, Externalizable, Jackson
- [[Java Date and Time]] — java.time, ZonedDateTime, Duration

## 06 — GC и Память
`Base/Java/06_GC_and_Memory/`

### GC Internals
- [[GC Roots и достижимость объектов]] — tri-color marking, types of roots
- [[Safepoints и Stop-The-World]] — polling page, time-to-safepoint
- [[Write Barriers и Card Table]] — SATB, RSet, colored pointers
- [[Finalization и Cleaner API]] — finalize() проблемы, PhantomReference

### GC Алгоритмы
- [[G1GC — архитектура и tuning]] — regions, IHOP, Mixed GC, Evacuation Failure
- [[ZGC и Generational ZGC]] — colored pointers, load barrier, <1 мс паузы
- [[Shenandoah GC]] — Brooks pointers, concurrent relocation
- [[Parallel и Serial GC]] — copying collector, mark-compact, throughput
- [[Epsilon GC]] — no-op GC, benchmarking, short-lived jobs

### GC Tuning
- [[JVM флаги для GC]] — полный справочник флагов
- [[Анализ GC логов — JFR, GCEasy]] — unified logging, JFR events, GCEasy
- [[GC и латентность — практика]] — allocation rate, promotion, off-heap

## 07 — Modern Java Platform
`Base/Java/07_Modern_Java_Platform/`

### Project Loom (Java 21)
- [[Virtual Threads — модель и архитектура]]
- [[Carrier Threads и Pinning]]
- [[Structured Concurrency]]

### Project Valhalla
- [[Value Classes и Primitive Classes]] — flat memory layout, zero boxing
- [[Object Identity — что меняется]] — synchronized, ==, IdentityObject
- [[Valhalla и коллекции]] — specialized generics, boxing overhead

### Project Panama (Java 22)
- [[Foreign Function and Memory API]] — MemorySegment, Arena, Linker
- [[Panama vs Unsafe vs JNI]] — сравнение подходов
- [[Vector API — SIMD в Java]] — AVX, SPECIES, dot product

---

## Быстрые ссылки для интервью
- → [[Interview Cheatsheet — Senior Java]] — шпаргалка по всем темам
- → [[JEP Reference Map]] — JEP по темам и версиям Java
