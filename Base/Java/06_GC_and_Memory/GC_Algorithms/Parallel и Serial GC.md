# Parallel и Serial GC

> **Serial GC** — однопоточный сборщик, STW. **Parallel GC** (Throughput Collector) — многопоточный STW сборщик. Оба: generational (Young/Old), без concurrent фаз. Цель — максимальный throughput, latency не в приоритете.

## Связанные темы
[[G1GC — архитектура и tuning]], [[ZGC и Generational ZGC]], [[JVM флаги для GC]], [[Структура памяти JVM]]

---

## Serial GC

```bash
-XX:+UseSerialGC
```

**Minor GC** (Young): copying collector — живые объекты копируются из Eden+Survivor в другой Survivor или Tenured.

**Major GC** (Old/Full): mark-sweep-compact — однопоточно, на весь heap.

Когда использовать:
- Embedded системы, очень маленький heap (<100 МБ)
- Однопоточное приложение (GC потоки конкурируют с единственным app-потоком)
- Тестирование, CI где GC не важен

## Parallel GC (Throughput Collector)

```bash
-XX:+UseParallelGC  # default до Java 8
```

Всё то же что Serial, но GC-фазы выполняются **несколькими потоками**:

```
STW Minor GC:
  Thread-0 [scan & evacuate Eden partition 0]
  Thread-1 [scan & evacuate Eden partition 1]
  Thread-2 [scan & evacuate Eden partition 2]
  Thread-3 [scan & evacuate Eden partition 3]
  ─────────────────────────────────────────
  Синхронизация → результат

STW Full GC:
  Многопоточный mark + compact Old Gen
```

## Young Generation — Copying Collector

```
Before GC:
  Eden:     [████████ full]
  Survivor0:[███ objects]   ← from-space
  Survivor1:[ empty ]       ← to-space

After Minor GC:
  Eden:     [ empty ]
  Survivor0:[ empty ]       ← теперь to-space
  Survivor1:[██ survived]   ← from-space, возраст +1
  Tenured:  [promoted objects]
```

Объекты, пережившие `MaxTenuringThreshold` Minor GC → promotion в Old Gen.

## Old Generation — Mark-Compact

1. **Mark**: пометить все живые объекты (от GC Roots)
2. **Summary** (Parallel): вычислить новые адреса
3. **Compact**: переместить объекты, обновить ссылки

Нет фрагментации (в отличие от Mark-Sweep). Полная STW пауза, proportional to heap size.

## Ключевые параметры

```bash
# Количество GC потоков
-XX:ParallelGCThreads=8  # default = nCPU если <= 8, иначе nCPU*5/8

# Цель по throughput (99% времени в приложении, 1% в GC)
-XX:GCTimeRatio=99  # default 99 → не более 1% в GC

# Максимальный размер паузы (Parallel адаптирует heap размеры)
-XX:MaxGCPauseMillis=500

# Adaptive sizing (Parallel автоматически регулирует Young/Old размеры)
-XX:+UseAdaptiveSizePolicy  # default true

# Явные размеры Young
-XX:NewRatio=2     # Old:Young = 2:1 (Young = heap/3)
-XX:NewSize=512m   # min Young size
-XX:MaxNewSize=2g  # max Young size

# Tenuring threshold
-XX:MaxTenuringThreshold=15  # default 15 Minor GC до promotion
-XX:InitialTenuringThreshold=7
```

## Adaptive Size Policy

С `-XX:+UseAdaptiveSizePolicy` (default) Parallel GC сам регулирует:
- Размер Eden, Survivor
- Tenuring threshold

Цели (в порядке приоритета):
1. Не превышать `MaxGCPauseMillis`
2. Обеспечить `GCTimeRatio`
3. Минимизировать footprint

```bash
# Видеть решения Adaptive Size Policy
-XX:+PrintAdaptiveSizePolicy  # или -Xlog:gc+ergo*=debug
```

## Throughput vs Latency

```
Heap = 8 GB, Parallel GC:

Minor GC: ~50-200 мс (зависит от Young Gen size, выживших)
Full GC:  2-10 секунд (весь Old Gen compact)

Parallel GC отлично подходит для:
- Batch jobs (ETL, Spark-подобные задачи)
- Overnight reporting
- Задачи где пауза раз в N минут не критична
- Throughput > 95% - главная цель
```

## Parallel vs G1 — когда Parallel быстрее

Parallel GC даёт лучший throughput чем G1 при:
- Высоком allocation rate (много короткоживущих объектов)
- Batch workload без latency SLA
- Heap < 4 GB

G1 выигрывает при:
- Большом heap (>4 GB)
- Latency требованиях
- Смешанном workload (short + long-lived objects)

```bash
# Benchmark (условный):
# Parallel GC: throughput 98%, pause 2 сек раз в 5 минут
# G1 GC:       throughput 95%, pause 50-200 мс регулярно
# Для batch: Parallel лучше
# Для web API: G1 лучше
```

---

## Вопросы на интервью
- В чём разница между Serial и Parallel GC?
- Как работает copying collector в Young Generation?
- Что такое Adaptive Size Policy?
- Когда лучше использовать Parallel GC вместо G1?
- Почему Full GC в Parallel медленнее чем Minor GC?

## Подводные камни
- Parallel GC Full GC — **однопоточный** в старых JVM (до Java 7 UseParallelOldGC отдельный флаг). Всегда включай `-XX:+UseParallelOldGC` если используешь старую JVM
- `UseAdaptiveSizePolicy` может неожиданно уменьшить Young Gen → частые Minor GC → увеличение трафика в Old Gen
- На контейнерах с лимитом CPU/памяти Parallel GC может не видеть реальное количество ядер (Java <8u191). Всегда явно задавай `-XX:ParallelGCThreads`
- `NewRatio` игнорируется если установлен `NewSize`/`MaxNewSize` — конфликт параметров
