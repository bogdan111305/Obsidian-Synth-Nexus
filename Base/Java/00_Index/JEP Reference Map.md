# JEP Reference Map

> Карта ключевых JEP (Java Enhancement Proposals) по темам и версиям Java. Ссылки на заметки в базе знаний.

---

## Project Loom — Virtual Threads

| JEP | Название | Java | Статус | Заметка |
|-----|---------|------|--------|---------|
| JEP 425 | Virtual Threads (Preview) | 19 | Preview | [[Virtual Threads — модель и архитектура]] |
| JEP 436 | Virtual Threads (2nd Preview) | 20 | Preview | |
| JEP 444 | Virtual Threads | 21 | Final | [[Virtual Threads — модель и архитектура]] |
| JEP 453 | Structured Concurrency (Preview) | 21 | Preview | [[Structured Concurrency]] |
| JEP 480 | Structured Concurrency (3rd Preview) | 23 | Preview | |
| JEP 487 | Scoped Values (4th Preview) | 24 | Preview | [[Scoped Values (Java 21, JEP 446)]] |
| JEP 446 | Scoped Values (Preview) | 21 | Preview | |

## Project Valhalla — Value Types

| JEP | Название | Java | Статус | Заметка |
|-----|---------|------|--------|---------|
| JEP 401 | Value Classes and Objects (Preview) | 23 | Preview | [[Value Classes и Primitive Classes]] |
| JEP 402 | Primitive Types in Patterns (Preview) | 23 | Preview | [[Value Classes и Primitive Classes]] |

## Project Panama — Native Interop

| JEP | Название | Java | Статус | Заметка |
|-----|---------|------|--------|---------|
| JEP 389 | Foreign Linker API (Incubator) | 16 | Incubator | [[Foreign Function and Memory API]] |
| JEP 454 | Foreign Function and Memory API | 22 | Final | [[Foreign Function and Memory API]] |
| JEP 338 | Vector API (Incubator) | 16 | Incubator | [[Vector API — SIMD в Java]] |
| JEP 489 | Vector API (9th Incubator) | 23 | Incubator | [[Vector API — SIMD в Java]] |

## Project Leyden — Startup

| JEP | Название | Java | Статус | Заметка |
|-----|---------|------|--------|---------|
| JEP 310 | Application Class-Data Sharing | 10 | Final | [[JVM Startup и AppCDS]] |
| JEP 483 | Ahead-of-Time Class Loading & Linking | 24 | Preview | [[Project Leyden и AOT]] |

## GC

| JEP | Название | Java | Статус | Заметка |
|-----|---------|------|--------|---------|
| JEP 318 | Epsilon: A No-Op GC (Experimental) | 11 | Experimental | [[Epsilon GC]] |
| JEP 333 | ZGC: A Scalable Low-Latency GC | 11 | Experimental | [[ZGC и Generational ZGC]] |
| JEP 377 | ZGC: Production Ready | 15 | Final | [[ZGC и Generational ZGC]] |
| JEP 439 | Generational ZGC | 21 | Final | [[ZGC и Generational ZGC]] |
| JEP 189 | Shenandoah: A Low-Pause-Time GC | 12 | Experimental | [[Shenandoah GC]] |
| JEP 379 | Shenandoah: Production Ready | 15 | Final | [[Shenandoah GC]] |
| JEP 248 | Make G1 the Default GC | 9 | Final | [[G1GC — архитектура и tuning]] |

## Language Features

| JEP | Название | Java | Статус | Заметка |
|-----|---------|------|--------|---------|
| JEP 305 | Pattern Matching instanceof (Preview) | 14 | Preview | [[Pattern Matching]] |
| JEP 394 | Pattern Matching instanceof | 16 | Final | |
| JEP 359 | Records (Preview) | 14 | Preview | [[Records]] |
| JEP 395 | Records | 16 | Final | |
| JEP 360 | Sealed Classes (Preview) | 15 | Preview | [[Sealed Classes]] |
| JEP 409 | Sealed Classes | 17 | Final | |
| JEP 441 | Pattern Matching for switch | 21 | Final | [[Современные возможности — Switch и Pattern Matching]] |
| JEP 440 | Record Patterns | 21 | Final | |
| JEP 461 | Stream Gatherers (Preview) | 22 | Preview | [[Java Stream API & Functional Programming]] |
| JEP 485 | Stream Gatherers | 24 | Final | |
| JEP 456 | Unnamed Patterns and Variables | 22 | Final | |

## JVM / Bytecode

| JEP | Название | Java | Статус | Заметка |
|-----|---------|------|--------|---------|
| JEP 292 | Implement invokedynamic | 7 | Final | [[invokedynamic — механика]] |
| JEP 193 | Variable Handles (VarHandle) | 9 | Final | [[MethodHandle & LambdaMetafactory]] |
| JEP 280 | Indify String Concatenation | 9 | Final | [[MethodHandle & LambdaMetafactory]] |
| JEP 158 | Unified JVM Logging | 9 | Final | [[Анализ GC логов — JFR, GCEasy]] |
| JEP 328 | Flight Recorder | 11 | Final | [[JVM Profiling & Observability]] |
| JEP 421 | Deprecate Finalization | 18 | Final | [[Finalization и Cleaner API]] |

---

## Версии Java — ключевые изменения

| Версия | Год | Ключевые фичи для Senior |
|--------|-----|--------------------------|
| Java 8 | 2014 | Lambda, Stream, Optional, CompletableFuture |
| Java 9 | 2017 | JPMS (modules), VarHandle, Unified Logging |
| Java 11 | 2018 | ZGC (exp), Epsilon, HTTP Client |
| Java 14 | 2020 | Records (preview), Pattern matching (preview) |
| Java 16 | 2021 | Records (final), FFM (incubator) |
| Java 17 | 2021 | Sealed Classes (final), LTS |
| Java 21 | 2023 | Virtual Threads (final), Generational ZGC, Sequenced Collections, LTS |
| Java 22 | 2024 | FFM API (final), Stream Gatherers (preview) |
| Java 23 | 2024 | Valhalla Value Classes (preview), Gatherers 2nd preview |
| Java 24 | 2025 | Gatherers (final), Leyden AOT (preview), Valhalla (2nd preview) |
| Java 25 | 2025 | LTS (ожидается) |
