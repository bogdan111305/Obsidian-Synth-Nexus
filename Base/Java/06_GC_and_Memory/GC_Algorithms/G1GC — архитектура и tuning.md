# G1GC — архитектура и tuning

> **G1 (Garbage-First)** — дефолтный GC в JDK 9+. Разбивает heap на равные регионы (~1-32 MB), собирает регионы с наибольшим количеством мусора первыми. Цель: предсказуемые паузы с `-XX:MaxGCPauseMillis` при сохранении throughput.

## Связанные темы
[[ZGC и Generational ZGC]], [[Safepoints и Stop-The-World]], [[Write Barriers и Card Table]], [[GC Roots и достижимость объектов]], [[JVM флаги для GC]]

---

## Архитектура: Regions

```
Heap (64 регионы по 4 МБ = 256 МБ пример):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ E │ S │ O │ H │ E │ E │ F │ O │  ...
└───┴───┴───┴───┴───┴───┴───┴───┘
E = Eden    S = Survivor    O = Old    H = Humongous    F = Free
```

- **Eden**: куда аллоцируются новые объекты
- **Survivor**: выжившие после Minor GC
- **Old**: долгоживущие объекты после N promotion threshold
- **Humongous**: объекты > 50% размера региона — аллоцируются в contiguous регионах

Размер региона: `-XX:G1HeapRegionSize=N` (1, 2, 4, 8, 16, 32 МБ). Автовычисляется как `heap / 2048`.

## Фазы GC

### Young-only (Minor GC) — STW
1. Root scan (GC Roots + RSets)
2. Evacuation: живые объекты копируются в новый Survivor/Old регион
3. Старые Eden/Survivor регионы становятся Free

### Concurrent Marking Cycle
Запускается когда Old Gen достигает **IHOP (InitiatingHeapOccupancyPercent)**:

```
1. Initial Mark (STW, <5 мс)  — помечаем объекты достижимые из GC Roots
2. Root Region Scan (concurrent) — сканируем Survivor регионы
3. Concurrent Mark (concurrent)  — marking всего heap
4. Remark (STW, <5 мс)          — SATB завершение, finalization
5. Cleanup (STW, ~мс)           — обновление статистики, reclaim empty regions
6. Concurrent Cleanup (concurrent)
```

### Mixed GC — STW
После marking: evacuate Young + часть Old регионов (наиболее mutable — откуда "Garbage First").

## Ключевые параметры

```bash
# Целевая максимальная пауза (мягкая цель, не гарантия)
-XX:MaxGCPauseMillis=200  # default 200 мс

# Процент heap при котором начинается Concurrent Marking
-XX:InitiatingHeapOccupancyPercent=45  # default 45%
# ↑ уменьши если часто Evacuation Failure (G1 не успевает до Full GC)

# Размер heap
-Xms4g -Xmx4g  # фиксируй Xms=Xmx чтобы не было resize пауз

# Регион
-XX:G1HeapRegionSize=8m  # 1..32 МБ, степень 2

# Процент Young Gen
-XX:G1NewSizePercent=5    # min % heap для Young (default 5%)
-XX:G1MaxNewSizePercent=60 # max % heap для Young (default 60%)

# Сколько Mixed GC циклов для очистки Old
-XX:G1MixedGCCountTarget=8  # default 8

# Порог живого для Mixed GC региона (не трогать регион если живых > N%)
-XX:G1MixedGCLiveThresholdPercent=85  # default 85%
```

## Humongous Allocation

```java
// Объект > G1HeapRegionSize/2 → Humongous
// При регионе 4 МБ: объект > 2 МБ → Humongous
byte[] big = new byte[3 * 1024 * 1024];  // → Humongous region
```

Проблемы:
- Всегда аллоцируется в Old Gen (contiguous регионы)
- Собирается только в Concurrent Marking Cycle, не в Minor GC
- Фрагментирует heap

Решение: увеличь `-XX:G1HeapRegionSize` или избегай больших временных массивов.

## Evacuation Failure

Когда G1 не может найти свободный регион для evacuation (heap full или фрагментирован):

```
[GC (G1 Evacuation Pause) [Eden: 512M->0B(512M) Survivors: 128M->128M]
  [Evacuation Failure]
  [Full GC (Allocation Failure) ...]  ← дорогая Serial Full GC
```

Причины:
1. Heap слишком мал
2. IHOP слишком высокий → Concurrent Marking поздно запускается
3. Много объектов, переживших несколько GC (premature promotion)
4. Много Humongous объектов

## Диагностика

```bash
# GC логирование
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# JFR
-XX:StartFlightRecording=duration=300s,filename=profile.jfr

# Ключевые метрики в логах:
# [Eden: N->0B] — Young evacuation
# [Survivors: N->M] — survivor size
# [Heap: before->after(capacity)]
# To-space exhausted — признак Evacuation Failure
# Concurrent Mark Abort — marking прерван из-за нехватки памяти

# GCViewer / GCEasy.io для визуализации
```

## G1 vs другие GC (когда выбирать)

| Критерий | G1 | ZGC | Parallel |
|----------|----|-----|---------|
| Heap | 4 GB — 100 GB | 8 MB — 16 TB | Любой |
| Паузы | 10-200 мс | <1 мс | >1 сек на большом heap |
| Throughput | Высокий | Высокий | Максимальный |
| Latency | Среднее | Минимальное | Плохое |
| Сложность tuning | Средняя | Низкая | Низкая |

G1 — золотая середина. ZGC выбирай если паузы >50 мс критичны. Parallel — для batch-обработки где throughput важнее latency.

---

## Вопросы на интервью
- Что такое "Garbage First" в названии G1?
- Объясните фазу Concurrent Marking Cycle и её STW паузы.
- Что такое Humongous object и почему это проблема?
- Что такое Evacuation Failure? Как диагностировать и предотвратить?
- Как IHOP влияет на поведение G1?

## Подводные камни
- `MaxGCPauseMillis` — мягкая цель, не жёсткая гарантия. G1 может её превысить.
- Слишком маленький `-XX:InitiatingHeapOccupancyPercent` → частые marking cycles → overhead на throughput
- `Xms != Xmx` → resize паузы при росте heap — всегда устанавливай их равными на проде
- G1 Full GC — Serial, однопоточный, очень дорогой (~секунды). Его появление = что-то сломано
- RegionSize слишком маленький при большом heap → огромное количество регионов → overhead на RSet
