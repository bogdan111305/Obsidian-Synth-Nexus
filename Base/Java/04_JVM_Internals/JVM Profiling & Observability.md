# JVM Profiling & Observability

> Инструменты: **JFR** (Java Flight Recorder, < 1% overhead, production-safe), **async-profiler** (CPU/alloc/lock, нет SafePoint bias), **jcmd** (Swiss Army Knife), **jstack/jmap** (threads/heap dump), **MAT** (heap analysis).
> На интервью: SafePoint bias и почему JVMTI профилировщики ненадёжны, JFR Streaming API, Retained vs Shallow Size в MAT, как диагностировать memory leak.

## Связанные темы
[[JIT Compiler & Optimizations]], [[Java Agents & Instrumentation API]], [[Java Memory Structure]]

---

## Java Flight Recorder (JFR)

Встроенный профилировщик с < 1% overhead. Production-safe. Работает через ring buffer в памяти JVM.

```bash
# При старте:
java -XX:StartFlightRecording=duration=60s,filename=app.jfr -jar myapp.jar

# К работающей JVM:
jcmd <pid> JFR.start duration=120s filename=/tmp/app.jfr name=MyRec
jcmd <pid> JFR.stop name=MyRec
jcmd <pid> JFR.dump name=MyRec filename=/tmp/snapshot.jfr  # без остановки
jcmd <pid> JFR.check
```

**Ключевые JFR события:**

| Категория | Событие | Что показывает |
|---|---|---|
| GC | `jdk.GCPhasePause` | Длительность GC пауз |
| GC | `jdk.GarbageCollection` | Тип GC, до/после heap |
| Memory | `jdk.ObjectAllocationInNewTLAB` | Аллокации (sampling) |
| IO | `jdk.SocketRead/Write` | Сетевые операции |
| Thread | `jdk.MonitorWait` | Ожидание на мониторе |
| JIT | `jdk.Compilation` | JIT компиляции |
| VT | `jdk.VirtualThreadPinned` | Pinning virtual threads |

**JFR Streaming API (Java 14+) — real-time без файла:**
```java
try (var stream = RecordingStream.ofConfiguration(Configuration.getConfiguration("profile"))) {
    stream.onEvent("jdk.GCPhasePause", event -> {
        if (event.getDuration("duration").toMillis() > 100)
            alert("Long GC pause: " + event.getDuration("duration"));
    });

    stream.onEvent("jdk.ObjectAllocationInNewTLAB", event -> {
        log("Alloc: %s, size: %d", event.getClass("objectClass").getName(),
            event.getLong("tlabSize"));
    });

    stream.setReuse(true); // переиспользовать объект события
    stream.start();
}
```

**Кастомные JFR события:**
```java
@Label("User Login")
@Category({"Business", "Security"})
@StackTrace(false)  // не записывать stack trace (дешевле)
public class UserLoginEvent extends Event {
    @Label("User ID") public long userId;
    @Label("Auth Method") public String authMethod;
}

// Использование:
UserLoginEvent event = new UserLoginEvent();
event.begin();
try {
    event.userId = user.getId();
    event.authMethod = "JWT";
} finally {
    event.end();
    event.commit(); // записать в буфер
}
```

---

## async-profiler

**Не-SafePoint** профилировщик. Единственный корректный CPU профилировщик для JVM.

**Почему async-profiler, а не JVMTI:**
```
JVMTI sampling: останавливает поток только в SafePoint
  → SafePoint Bias: показывает код около goto/back-edge
  → НЕ видит tight spin-loops (они вне SafePoint)
  → Ложная картина!

async-profiler: прерывает через OS сигналы (SIGPROF)
  → Честный стек трейс в ЛЮБОЙ момент
  → Видит нативный код и JIT-compiled
  → Реальные горячие места
```

```bash
# CPU профилирование:
./asprof -d 30 -f flamegraph.html <pid>

# Heap allocation:
./asprof -e alloc -d 30 -f alloc.html <pid>

# Wall-clock (включает IO/блокирующие потоки):
./asprof -e wall -d 30 -f wall.html <pid>

# Lock профилирование:
./asprof -e lock -d 30 -f lock.html <pid>

# Сохранить как JFR:
./asprof -d 60 -f output.jfr <pid>
```

**Flamegraph:** читается снизу вверх. Ширина плато = % CPU. Широкие верхушки = горячие методы.

---

## Heap Dump анализ

```bash
# Автоматически при OOM:
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/oom.hprof -jar myapp.jar

# Вручную через jcmd:
jcmd <pid> GC.heap_dump filename=/tmp/live.hprof

# Через jmap:
jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>
```

**Eclipse MAT — ключевые отчёты:**
```
Leak Suspects Report → автоматически ищет накопления
Dominator Tree → объекты с наибольшим retained heap
  Shallow Size = только сам объект (заголовок + поля)
  Retained Size = сколько освободится если объект удалить → ВАЖНЕЕ!
OQL → SELECT * FROM java.util.ArrayList a WHERE a.size > 10000
Thread Overview → стеки всех потоков + объекты на стеке
```

**Диагностика Memory Leak:**
```
1. Симптом: Heap растёт после каждого Full GC
2. jcmd <pid> GC.heap_dump ...
3. MAT: Dominator Tree → максимальный retained?
4. MAT: Path to GC Roots → почему объект жив?
5. Типичные виновники:
   - ThreadLocal без remove() в thread pool
   - static Map/List, которые только растут
   - Event listeners не отписанные
   - ClassLoader leaks (Tomcat hot reload, OSGi)
   - Внутренние кэши сторонних библиотек
```

---

## jcmd: Swiss Army Knife

```bash
jcmd -l                                          # список JVM процессов
jcmd <pid> help                                  # все команды
jcmd <pid> GC.heap_info                          # текущее состояние heap
jcmd <pid> GC.run                                # запустить GC
jcmd <pid> GC.class_histogram | head -30         # топ объектов
jcmd <pid> VM.native_memory summary              # native memory (нужен -XX:NativeMemoryTracking=summary)
jcmd <pid> VM.flags                              # флаги JVM
jcmd <pid> Thread.print                          # все стеки потоков
jcmd <pid> VM.class_hierarchy                    # иерархия классов
jcmd <pid> Compiler.directives_print             # JIT директивы
```

---

## JVM Metrics (JMX)

```java
// Heap:
MemoryMXBean memory = ManagementFactory.getMemoryMXBean();
MemoryUsage heap = memory.getHeapMemoryUsage();
// heap.getUsed() / heap.getMax()

// GC:
for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
    // gc.getName(), gc.getCollectionCount(), gc.getCollectionTime()
}

// Потоки + deadlock detection:
ThreadMXBean threads = ManagementFactory.getThreadMXBean();
long[] deadlocked = threads.findDeadlockedThreads();
if (deadlocked != null) {
    ThreadInfo[] info = threads.getThreadInfo(deadlocked, true, true);
    // log deadlock chain
}
```

**Spring Boot + Micrometer:** автоматически экспортирует `jvm_memory_used_bytes`, `jvm_gc_pause_seconds`, `jvm_threads_live_threads` в Prometheus.

---

## JFR vs async-profiler

| | JFR | async-profiler |
|---|---|---|
| Overhead в продакшне | < 1% | ~2-5% |
| SafePoint bias | Минимальный | Нет |
| Аллокации | Sampling | Sampling + точный |
| Wall-clock | Нет | Да |
| Кастомные события | Да | Нет |
| Долгосрочный мониторинг | Да (ring buffer) | Нет |
| Требования | Встроен | root/CAP_SYS_ADMIN в контейнере |
| Применение | Production мониторинг | Целевое исследование |

---

## Вопросы на интервью

- Что такое SafePoint Bias? Почему JVMTI CPU profiling показывает ложную картину?
- Чем async-profiler отличается от встроенного JFR CPU sampling?
- Что такое Retained Size? Почему он важнее Shallow Size при анализе утечек?
- Как JFR Streaming API работает без записи в файл?
- Как диагностировать memory leak? Какие типичные виновники?
- Как получить thread dump без jstack? (jcmd Thread.print)
- Как узнать CPU throttling в k8s через JVM метрики?
- Зачем нужны кастомные JFR события? Пример использования.

---

## Подводные камни

- **SafePoint bias в продакшне** — при диагностике production проблем с VisualVM/YourKit можно получить ложные результаты: tight loops без object access никогда не попадут в SafePoint → профиль показывает "горячим" другой код. Используй async-profiler.
- **Wall-clock vs CPU** — wall-clock профилирование показывает sleeping/waiting потоки. Для CPU-bound: CPU mode. Для I/O latency: wall-clock mode.
- **Heap dump останавливает JVM** — `jmap`/`GC.heap_dump` — STW операция. На больших heap (>8 GB) может занять минуты. В продакшне: автодамп при OOM или делай в off-peak.
- **JFR Streaming overhead** — хотя JFR production-safe, `RecordingStream` с подпиской на `ObjectAllocationInNewTLAB` может добавлять overhead при высоком alloc rate. Используй `setMaxSize`/`setMaxAge` для ограничения буфера.
- **async-profiler в контейнере** — требует `--cap-add=SYS_ADMIN` или `--privileged`. В managed k8s (EKS, GKE) это часто невозможно. Альтернатива: Java агент (без root): `./asprof -agentpath`.
- **MAT Retained Size для объектов в циклических графах** — MAT вычисляет retained через dominance tree. Объекты в циклических ссылках могут показывать неожиданный retained (ни один не доминирует другого). Ориентируйся на "Leak Suspects" отчёт.
