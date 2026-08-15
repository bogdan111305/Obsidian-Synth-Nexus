# GC и латентность — практика

> Снизить GC-паузы = уменьшить их частоту + продолжительность + time-to-safepoint. Три уровня: выбор GC, sizing heap/generations, оптимизация кода (меньше allocation, меньше promotion).

## Связанные темы
[[JVM флаги для GC]], [[Анализ GC логов — JFR, GCEasy]], [[G1GC — архитектура и tuning]], [[ZGC и Generational ZGC]], [[Safepoints и Stop-The-World]]

---

## Источники latency в GC

```
Total pause = time-to-safepoint + actual GC work

Time-to-safepoint:
  ├── Потоки в safepoint-hostile loops
  ├── Потоки в длинных native методах
  └── Много потоков (больше времени на синхронизацию)

Actual GC work:
  ├── Root scanning (стеки, static fields)
  ├── Object graph traversal (marking)
  ├── Evacuation/compaction
  └── Reference processing (finalization, weak refs)
```

## Шаг 1: Выбор GC для latency

```
Latency SLA >= 500 мс:   G1GC (default) достаточно
Latency SLA 50-500 мс:   G1GC с tuning
Latency SLA 10-50 мс:    ZGC или Shenandoah
Latency SLA < 10 мс:     ZGC (Generational, Java 21+)
Latency SLA < 1 мс:      Epsilon + restart strategy (экстремально)
```

## Шаг 2: Диагностика текущих пауз

```bash
# 1. Включить полное логирование
-Xlog:gc*:file=gc.log:time,uptime,level,tags
-Xlog:safepoint=info:file=safepoint.log:time,uptime

# 2. JFR запись
jcmd <pid> JFR.start duration=300s filename=latency.jfr settings=profile

# 3. Анализ: найти P99 паузы
grep "Pause" gc.log | awk '{print $NF}' | sort -n | tail -20

# 4. Найти time-to-safepoint > 10 мс
grep "Total time for which application threads were stopped" safepoint.log | \
  awk '{if ($NF > 0.010) print}'
```

## Шаг 3: Борьба с Allocation Rate

Высокий allocation rate → частые Minor GC → частые паузы.

```bash
# Измерить allocation rate через JFR
jfr print --events jdk.GCHeapSummary latency.jfr | grep "Before GC"

# Или Micrometer:
# rate(jvm_gc_memory_allocated_bytes_total[1m]) → МБ/с
```

### Снизить allocation:

```java
// ❌ Создаёт много мусора в hot path
String result = "";
for (Item item : items) {
    result += item.getName() + ", ";  // новый String на каждой итерации
}

// ✅ StringBuilder — один объект
StringBuilder sb = new StringBuilder(items.size() * 20);
for (Item item : items) {
    sb.append(item.getName()).append(", ");
}
String result = sb.toString();

// ❌ Boxing в hot path
Map<String, Integer> counts = new HashMap<>();
counts.merge(key, 1, Integer::sum);  // Integer boxing

// ✅ Primitive коллекции (Eclipse Collections, Koloboke, Agrona)
MutableObjectIntMap<String> counts = ObjectIntMaps.mutable.empty();
counts.addToValue(key, 1);
```

### Object Pooling

```java
// Для дорогих объектов в hot path
// Пример: пул ByteBuffer для network IO
public class ByteBufferPool {
    private final ArrayDeque<ByteBuffer> pool = new ArrayDeque<>();
    private final int bufferSize;
    
    public ByteBuffer acquire() {
        ByteBuffer buf = pool.poll();
        return buf != null ? buf : ByteBuffer.allocateDirect(bufferSize);
    }
    
    public void release(ByteBuffer buf) {
        buf.clear();
        pool.push(buf);
    }
}

// Netty ByteBuf pooling — уже встроено в Netty
// jemalloc-style для нативной памяти
```

## Шаг 4: Снизить Promotion Rate

Объекты, переживающие Young GC → промоутируются в Old Gen → Full/Mixed GC.

```bash
# Признак premature promotion в логах:
# Promotion rate > allocation rate * 0.1 — проблема

# JFR event: jdk.GCHeapSummary — смотреть рост Old Gen между Minor GC
```

### Причины и решения:

```java
// ❌ Долгоживущий кэш в Young Gen — объекты выживают несколько GC
private static final Map<K, V> cache = new LRUMap<>(1000);
// Решение: переместить в off-heap (Caffeine + soft values, Ehcache)

// ❌ ThreadLocal не очищается → объект живёт столько же сколько поток
ThreadLocal<HeavyContext> ctx = ThreadLocal.withInitial(HeavyContext::new);
// pool thread никогда не завершается → HeavyContext в Old Gen
// Решение: ctx.remove() после каждого запроса

// ❌ Large objects → Humongous (G1) → сразу в Old Gen
byte[] buffer = new byte[2 * 1024 * 1024];  // > G1RegionSize/2 → Old Gen
// Решение: увеличь G1HeapRegionSize или pooling
```

## Шаг 5: G1 Latency Tuning

```bash
# Уменьшить цель паузы
-XX:MaxGCPauseMillis=100  # с 200 до 100 мс

# Ускорить начало Concurrent Marking
-XX:InitiatingHeapOccupancyPercent=35  # с 45 до 35%
# ↑ GC начнёт marking раньше → меньше вероятность Evacuation Failure

# Уменьшить Young Gen (меньше эвакуировать за раз)
-XX:G1MaxNewSizePercent=30  # с 60% до 30%
# Но: более частые Minor GC

# Больше регионов для evacuation в Mixed GC
-XX:G1MixedGCCountTarget=4  # с 8 до 4 → каждый Mixed GC обрабатывает больше

# Concurrent threads (больше для быстрого marking)
-XX:ConcGCThreads=4  # увеличь если CPU позволяет
```

## Шаг 6: Устранить safepoint-hostile код

```java
// Проблема: C2 не вставляет safepoint в "counted loop"
for (int i = 0; i < 100_000_000; i++) {
    process(data[i]);  // без вызовов методов (inlined) → нет safepoint poll
}

// Решение 1: убедись что UseCountedLoopSafepoints включён (Java 10+)
// -XX:+UseCountedLoopSafepoints (default true)

// Решение 2: разбить на части
int chunkSize = 10_000;
for (int i = 0; i < 100_000_000; i += chunkSize) {
    processChunk(data, i, Math.min(i + chunkSize, 100_000_000));
    // вызов метода = safepoint poll opportunity
}
```

## Шаг 7: Off-Heap для latency-critical данных

```java
// Вынести hot data из GC heap
import java.lang.foreign.*;  // Panama FFM API (Java 21+)

try (Arena arena = Arena.ofConfined()) {
    MemorySegment segment = arena.allocate(1024 * 1024);  // 1 МБ off-heap
    // GC не трогает эту память — нет GC pause влияния
    // Освобождается при закрытии Arena
}

// Для больших кэшей: Chronicle Map, MapDB, Ignite off-heap
// Для сетевых буферов: ByteBuffer.allocateDirect() (с pooling)
```

## Checklist для Production

```
□ GC логирование включено с ротацией
□ Safepoint логирование включено
□ JFR continuous recording настроен
□ Prometheus метрики GC экспортируются
□ Xms = Xmx
□ MaxMetaspaceSize установлен
□ HeapDumpOnOutOfMemoryError включён
□ AlertManager: GC pause P99 > threshold
□ AlertManager: heap usage > 85%
□ AlertManager: GC throughput < 95%
```

---

## Вопросы на интервью
- Из чего состоит полная GC latency? (не только GC работа, но и time-to-safepoint)
- Как allocation rate влияет на GC паузы?
- Что такое premature promotion и как его диагностировать?
- Как снизить GC impact для latency-critical приложения пошагово?
- Когда имеет смысл использовать off-heap память?

## Подводные камни
- Уменьшение `MaxGCPauseMillis` может увеличить частоту пауз (G1 эвакуирует меньше за раз → нужно чаще)
- Object pooling без правильной очистки → объект несёт "старые" данные → security/correctness issues
- Off-heap через unsafe/DirectBuffer без pooling → native memory leak (Cleaner не вызвался)
- Слишком агрессивный concurrent GC (много ConcGCThreads) на CPU-bound сервисе → application threads starved → latency хуже
