# JVM флаги для GC

> Справочник ключевых `-XX:` флагов для настройки GC. Флаги делятся на: выбор GC, heap sizing, GC-специфичные параметры, диагностика. Большинство дефолтов работают хорошо — меняй только то, что объясняется в GC логах.

## Связанные темы
[[G1GC — архитектура и tuning]], [[ZGC и Generational ZGC]], [[Анализ GC логов — JFR, GCEasy]], [[GC и латентность — практика]], [[Структура памяти JVM]]

---

## Выбор GC

```bash
-XX:+UseG1GC          # G1 (default Java 9+), heap 4-100 GB
-XX:+UseZGC           # ZGC (default Java 21+ при -XX:+UseZGC), <1ms pauses
-XX:+ZGenerational    # Generational ZGC (Java 21+, рекомендуется с UseZGC)
-XX:+UseShenandoahGC  # Shenandoah, low-latency, OpenJDK/Red Hat
-XX:+UseParallelGC    # Parallel (throughput), batch jobs
-XX:+UseSerialGC      # Serial, tiny heap / single CPU
-XX:+UseEpsilonGC     # No-op, benchmarks
```

## Heap Sizing

```bash
# Базовые
-Xms<size>   # initial heap (e.g., -Xms4g)
-Xmx<size>   # max heap     (e.g., -Xmx4g)
# Рекомендация: Xms = Xmx в production (no resize pauses)

# Young Generation
-Xmn<size>              # Young Gen size (override NewRatio)
-XX:NewRatio=2          # Old:Young = 2:1 → Young = heap/3
-XX:NewSize=512m        # min Young size
-XX:MaxNewSize=2g       # max Young size

# Metaspace (Java 8+, replaces PermGen)
-XX:MetaspaceSize=256m      # initial commitment (triggers GC when reached)
-XX:MaxMetaspaceSize=512m   # без лимита по дефолту — ставь всегда!

# Stack
-Xss512k    # per-thread stack size (default ~512k-1m)
            # Virtual threads: стек виртуален, -Xss не влияет

# Direct Memory (NIO, off-heap)
-XX:MaxDirectMemorySize=1g  # default = Xmx
```

## G1-специфичные флаги

```bash
-XX:MaxGCPauseMillis=200          # целевая пауза (мягкая)
-XX:InitiatingHeapOccupancyPercent=45  # % Old Gen до Concurrent Marking
-XX:G1HeapRegionSize=8m           # размер региона (1,2,4,8,16,32 МБ)
-XX:G1NewSizePercent=5            # min % heap для Young
-XX:G1MaxNewSizePercent=60        # max % heap для Young
-XX:G1MixedGCCountTarget=8       # Mixed GC cycles для очистки Old
-XX:G1MixedGCLiveThresholdPercent=85  # не Mixed GC если живых > 85%
-XX:G1HeapWastePercent=5          # допустимый % мусора без Mixed GC
-XX:G1ReservePercent=10           # % heap резерв для evacuation
-XX:ConcGCThreads=4               # concurrent marking threads
-XX:ParallelGCThreads=8           # STW GC threads
```

## ZGC-специфичные флаги

```bash
-XX:+UseZGC
-XX:+ZGenerational            # Java 21+
-XX:ConcGCThreads=4           # concurrent threads
-XX:+ZUncommit                # return memory to OS (default true)
-XX:ZUncommitDelay=300        # seconds before uncommit
-XX:ZAllocationSpikeTolerance=2.0  # spike tolerance factor
```

## Tenuring и Promotion

```bash
-XX:MaxTenuringThreshold=15   # max Minor GC до promotion (G1 default)
-XX:InitialTenuringThreshold=7
-XX:TargetSurvivorRatio=50    # % Survivor space target occupancy
                               # если больше → снижает tenuring threshold
```

## Диагностические флаги

```bash
# GC логирование (Java 9+ unified logging)
-Xlog:gc                          # базовый GC лог
-Xlog:gc*                         # всё о GC
-Xlog:gc*:file=gc.log:time,uptime,level,tags  # в файл с деталями
-Xlog:gc+ergo*=debug              # adaptive sizing решения
-Xlog:gc+phases=debug             # детальные фазы

# Heap dump при OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof

# Safepoint логирование
-Xlog:safepoint=info

# GC-причины
-Xlog:gc+cause=info

# JFR (рекомендуется для production)
-XX:StartFlightRecording=duration=1h,filename=recording.jfr,settings=profile
```

## Флаги для контейнеров (Docker/Kubernetes)

```bash
# Java 8u191+ / Java 10+ автоматически читает cgroup limits
-XX:+UseContainerSupport    # default true в Java 10+

# Явное ограничение CPU
-XX:ActiveProcessorCount=2  # принудительно 2 CPU (если cgroup неточен)

# Heap как % доступной памяти контейнера
-XX:InitialRAMPercentage=50.0   # initial heap = 50% RAM контейнера
-XX:MaxRAMPercentage=75.0       # max heap = 75% RAM
-XX:MinRAMPercentage=25.0       # min heap для маленьких контейнеров
# Лучше -XX:MaxRAMPercentage чем -Xmx в K8s (динамический sizing)
```

## Флаги для тюнинга производительности

```bash
# String Deduplication (G1 only) — дедупликация одинаковых строк
-XX:+UseStringDeduplication
-XX:StringDeduplicationAgeThreshold=3  # возраст до дедупликации

# Compressed Oops (heap < 32 GB → 4-byte refs вместо 8-byte)
-XX:+UseCompressedOops   # default true если heap < 32 GB
-XX:+UseCompressedClassPointers

# Large Pages (улучшает TLB hit rate)
-XX:+UseLargePages
-XX:LargePageSizeInBytes=2m

# NUMA awareness (multi-socket серверы)
-XX:+UseNUMA
```

## Сводная таблица: что и когда менять

| Проблема | Флаг | Действие |
|----------|------|----------|
| Частые Full GC | `-XX:InitiatingHeapOccupancyPercent` | Уменьши до 35-40 |
| Большие паузы G1 | `-XX:MaxGCPauseMillis` | Уменьши до 100-150 |
| OOM Metaspace | `-XX:MaxMetaspaceSize` | Увеличь, поставь лимит |
| OOM heap | `-Xmx` | Увеличь heap |
| Долгий time-to-safepoint | `-XX:+UseCountedLoopSafepoints` | Проверь что включён |
| Heap resize паузы | `-Xms` = `-Xmx` | Зафиксируй |
| Контейнер не видит память | `-XX:MaxRAMPercentage` | Замени `-Xmx` |

---

## Вопросы на интервью
- Почему рекомендуется `Xms=Xmx` в production?
- Что такое `-XX:MaxRAMPercentage` и зачем он нужен в Kubernetes?
- Для чего нужен `-XX:MaxMetaspaceSize`?
- Что изменит уменьшение `InitiatingHeapOccupancyPercent`?

## Подводные камни
- `-XX:MaxMetaspaceSize` не установлен по дефолту → unlimited → JVM может занять всю память сервера при ClassLoader leak
- Не ставь слишком много флагов — modern GC (G1, ZGC) хорошо самонастраивается. Tuning нужен только при доказанных проблемах
- `-Xmx` и `-XX:MaxRAMPercentage` — взаимоисключающие: если оба заданы, `-Xmx` побеждает
- `-XX:ParallelGCThreads` влияет на STW паузы — слишком много потоков на занятом CPU может создать contention
