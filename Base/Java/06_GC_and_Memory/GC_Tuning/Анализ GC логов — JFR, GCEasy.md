# Анализ GC логов — JFR, GCEasy

> GC логи и JFR — основные инструменты диагностики GC проблем. GC лог показывает паузы, причины, размеры heap. JFR (Java Flight Recorder) даёт детальные события с нулевым overhead. GCEasy.io — онлайн-парсер логов с визуализацией.

## Связанные темы
[[JVM флаги для GC]], [[GC и латентность — практика]], [[G1GC — архитектура и tuning]], [[ZGC и Generational ZGC]], [[JVM Profiling & Observability]]

---

## Включение GC логирования (Java 9+ Unified Logging)

```bash
# Минимальный лог (рекомендуется всегда на production)
-Xlog:gc:file=gc.log:time,uptime

# Детальный лог (для диагностики)
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# Ротация логов
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=20m

# Только паузы и причины
-Xlog:gc,gc+cause:file=gc.log:time

# Adaptive sizing решения (почему G1 меняет размеры)
-Xlog:gc+ergo*=debug:file=gc.log:time

# Safepoint отдельно (важно для полной картины)
-Xlog:safepoint=info:file=safepoint.log:time
```

## Структура G1 GC лога

```
# Minor GC (Young Collection):
[2026-03-30T10:00:01.234+0000][1.234s][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 512M->128M(2048M) 45.123ms

Разбор:
  GC(0)           — номер GC события
  Pause Young     — тип паузы
  (Normal)        — причина: Normal/Concurrent Start/Last Ditch
  G1 Evacuation Pause — subtype
  512M->128M      — heap before -> after
  (2048M)         — capacity
  45.123ms        — длительность паузы

# Concurrent Marking:
[gc] GC(1) Concurrent Cycle
[gc] GC(1) Pause Remark 1024M->800M(2048M) 12.456ms
[gc] GC(1) Pause Cleanup 800M->800M(2048M) 1.234ms
[gc] GC(1) Concurrent Cycle 234.567ms

# Full GC (плохой знак):
[gc] GC(5) Pause Full (Allocation Failure) 1900M->400M(2048M) 3456.789ms
                       ↑ причина        ↑ freed много    ↑ долго — секунды
```

## Ключевые паттерны в G1 логах

### Evacuation Failure
```
[gc] GC(7) To-space exhausted
[gc] GC(7) Evacuation Failure
→ G1 не смог найти место для evacuation
→ Увеличь heap или уменьши IHOP
```

### Concurrent Mark Abort
```
[gc] GC(8) Concurrent Mark Abort
→ Marking прерван (heap переполнен, срочный GC)
→ allocation rate > reclamation rate
```

### Humongous Allocation
```
[gc+ergo] GC(9) Attempting maximally parallel collection
→ Много Humongous объектов → фрагментация
```

## ZGC лог

```
[gc] GC(0) Garbage Collection (Warmup) 3656M(71%)->356M(7%)
[gc,phases] GC(0) Pause Mark Start 0.023ms    ← STW
[gc,phases] GC(0) Concurrent Mark 8.234ms
[gc,phases] GC(0) Pause Mark End 0.019ms      ← STW
[gc,phases] GC(0) Concurrent Process Non-Strong References 1.234ms
[gc,phases] GC(0) Concurrent Reset Relocation Set 0.234ms
[gc,phases] GC(0) Concurrent Select Relocation Set 2.345ms
[gc,phases] GC(0) Pause Relocate Start 0.022ms ← STW
[gc,phases] GC(0) Concurrent Relocate 4.567ms
[gc,stats ] GC(0) Small Pages: 256 / 512M, Empty: 128 / 256M
```

## Java Flight Recorder (JFR)

JFR — production-grade profiler встроен в HotSpot JVM. Overhead <1%.

```bash
# Запись на определённое время
java -XX:StartFlightRecording=duration=60s,filename=app.jfr,settings=profile MyApp

# Запись с ротацией (continuous)
java -XX:StartFlightRecording=maxage=1h,maxsize=500m,filename=continuous.jfr,settings=profile MyApp

# Динамически через jcmd
jcmd <pid> JFR.start duration=300s filename=recording.jfr settings=profile
jcmd <pid> JFR.stop name=1 filename=recording.jfr  # остановить с dump

# Просмотр events
jfr print --events jdk.GCPauseL2,jdk.GarbageCollection recording.jfr

# Настройки:
# settings=default  — низкий overhead (~0.1%)
# settings=profile  — средний overhead (~0.5%), GC allocation profiling
```

### GC-релевантные JFR события

```
jdk.GarbageCollection          — каждая GC пауза: duration, cause, GC ID
jdk.GCPauseL1                  — Young collection
jdk.GCPauseL2                  — Old/Mixed collection  
jdk.G1HeapSummary              — состояние heap
jdk.SafepointBegin/End         — safepoint паузы (включая time-to-safepoint)
jdk.ObjectAllocationInNewTLAB  — hot allocation paths
jdk.ObjectAllocationOutsideTLAB — large/slow allocation
jdk.JavaMonitorEnter           — lock contention
```

### Пример: найти top allocation sites

```bash
# JFR с allocation profiling
jcmd <pid> JFR.start settings=profile duration=60s filename=alloc.jfr

# Парсинг через JFR API или JDK Mission Control (JMC)
jfr print --events jdk.ObjectAllocationInNewTLAB alloc.jfr | head -100
```

## GCEasy.io

Онлайн-инструмент для визуализации GC логов:
1. Загружаешь gc.log
2. Получаешь: throughput %, pause distribution, heap usage graph, проблемные паузы
3. Recommendations — автоматические советы по tuning

Что смотреть в первую очередь:
- **GC Throughput**: должен быть >95% (время в приложении vs в GC)
- **Pause Time Distribution**: 95th/99th percentile
- **Heap usage**: не должен расти линейно (утечка)
- **Promotion Rate**: Fast Promotion → утечка в Old Gen

## JDK Mission Control (JMC)

GUI для анализа JFR recordings. Скачать: [jdk.java.net/jmc](https://jdk.java.net/jmc/)

Ключевые разделы:
- **GC** tab: все GC паузы, causes, heap timeline
- **Memory** tab: allocation rate, heap composition
- **Hot Methods**: кто аллоцирует больше всего
- **Lock Instances**: контеншн на мониторах

## Метрики GC для мониторинга (Prometheus/JMX)

```java
// Spring Boot Actuator + Micrometer — автоматически экспортирует:
// jvm_gc_pause_seconds_count{cause, action}
// jvm_gc_pause_seconds_sum{cause, action}
// jvm_gc_memory_allocated_bytes_total
// jvm_gc_memory_promoted_bytes_total
// jvm_memory_used_bytes{area, id}

// Grafana alerts:
// jvm_gc_pause_seconds{quantile="0.99"} > 0.5  → GC проблема
// rate(jvm_gc_memory_promoted_bytes_total[5m]) > threshold → premature promotion
```

---

## Вопросы на интервью
- Как включить GC логирование в Java 9+?
- Что означает "To-space exhausted" в G1 логах?
- Чем JFR лучше обычных GC логов?
- Как найти top allocation sites с помощью JFR?
- Что такое GC throughput и как его интерпретировать?

## Подводные камни
- GC лог показывает только GC паузы, не time-to-safepoint. Для полной картины нужен `-Xlog:safepoint`
- JFR с `settings=profile` включает allocation profiling — это sampling, не 100% точность. Не делай выводы на основе одного sample
- GCEasy бесплатный для логов < 100 МБ. Большие production логи → нужна локальная JMC
- Метрики Micrometer/Actuator не показывают time-to-safepoint — для этого JFR `jdk.SafepointBegin`
