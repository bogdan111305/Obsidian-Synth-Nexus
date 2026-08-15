# Структура памяти JVM

> JVM память: **Heap** (объекты, GC; Eden→S0/S1→Old Gen), **Stack** (frames, per-thread), **Metaspace** (метаданные классов, Java 8+), **Code Cache** (JIT-код), **Direct Memory** (NIO off-heap). `StackOverflowError` = стек переполнен. `OutOfMemoryError` = heap/metaspace/direct memory переполнен.
> На интервью: GC алгоритмы и их отличия, G1 Humongous/Evacuation Failure, ZGC colored pointers, ThreadLocal leaks, где хранятся static-поля.

## Связанные темы
[[Модель памяти Java (JMM) и барьеры памяти]], [[JIT Compiler & Optimizations]], [[ClassLoaders]], [[Reference Types (Weak, Soft, Phantom)]]

---

## Карта памяти JVM

```
Heap (GC-управляемая):          Stack (per-thread):
  Eden → S0/S1 → Old Gen          frame: main()
  + Humongous (G1)                frame: foo()
                                  (локальные переменные, ссылки)
Metaspace (вне Heap):           Code Cache:
  метаданные классов              JIT-скомпилированный код
  vtable/itable
  Runtime Constant Pool         Direct Memory (off-heap):
  bytecode                        NIO ByteBuffer.allocateDirect()
```

---

## Heap: жизненный цикл объекта

```
new Object() → TLAB в Eden → Minor GC → Survivor (S0/S1, ping-pong)
→ tenuring: MaxTenuringThreshold (default 15) → Old Gen
```

- **Большинство объектов умирает в Eden** — Minor GC быстрый
- **Крупные объекты** сразу в Old (или Humongous в G1)
- **Minor GC** — STW, копирующий, только Young
- **Full GC** — STW, весь heap + Metaspace, дорогой

**TLAB (Thread-Local Allocation Buffer):** каждый поток имеет буфер в Eden для быстрой аллокации (bump-the-pointer, нет синхронизации). Если объект не помещается — запрашивается новый TLAB.

---

## Stack и Stack Frame

Каждый поток имеет свой Stack. Каждый вызов метода = новый frame (кадр):
- Локальные переменные (примитивы и ссылки)
- Параметры метода
- Операндный стек (промежуточные вычисления)
- `this` (для instance методов)
- Адрес возврата

`StackOverflowError`: бесконечная рекурсия, слишком глубокая вложенность.
Настройка: `-Xss` (default 1MB Linux/Windows).

---

## Metaspace и static-поля

**Хранится в Metaspace:**
- Метаданные классов (имена, поля, методы, аннотации)
- Runtime Constant Pool (символические ссылки, строковые литералы)
- Байт-код методов
- vtable / itable (полиморфная диспетчеризация)
- ClassLoader данные

**String Pool** (`String.intern()`) — в **Heap** (с Java 7).

**Static-поля (Java 8+):** метаданные в Metaspace, **значения** в Heap как часть `java.lang.Class` объекта (class mirror):
```java
class MyClass {
    static String name = "Example";           // "Example" объект → Heap
    static List<String> cache = new ArrayList<>(); // объект → Heap
    static final int MAX = 100;               // значение 100 → в Class mirror в Heap
}
```

Static-поля — GC Roots → объекты, на которые они ссылаются, никогда не удалятся GC.

---

## GC алгоритмы

| GC | Паузы | Применение |
|---|---|---|
| **Serial** | STW, однопоточный | Маленькие heap, 1-core |
| **Parallel** | STW, многопоточный | Максимальный throughput |
| **G1** | STW + конкурентная маркировка | Default Java 9+, предсказуемые паузы |
| **ZGC** | < 1 мс | Большие heap (TB), критичны паузы |
| **Shenandoah** | < 1 мс | Средние heap, RedHat/OpenJDK |

*CMS deprecated Java 9, удалён Java 14.*

### G1 GC: Humongous Objects и Evacuation Failure

**Humongous объект** > 50% размера G1 региона (`-XX:G1HeapRegionSize`, 1-32 MB):
- Размещается в отдельных Humongous регионах (логически Old Generation)
- **Минует Eden/Survivor** — Minor GC не очищает!
- Освобождается только при Concurrent Mark Cleanup или Full GC
- **Проблема:** частые Humongous аллокации фрагментируют heap → нет contiguous регионов

**Evacuation Failure:** нет места (to-space) для копирования живых объектов во время Evacuation Pause:
```
Лог: [GC pause (G1 Evacuation Pause) (young) (to-space exhausted)...]
```
1. G1 помечает невыселенные объекты "self-forwarded"
2. Запускается дополнительный Concurrent Mark
3. Часто приводит к Full GC

Решение: снизить IHOP (`-XX:InitiatingHeapOccupancyPercent=35`), увеличить `-Xmx`.

### ZGC: Colored Pointers + Load Barriers

ZGC использует 4 бита в 64-битном указателе:
```
Marked0 (bit 40) | Marked1 (bit 41) | Remapped (bit 42) | Finalizable (bit 43)
```

Как конкурентное перемещение работает без STW:
1. GC обновляет `Remapped` бит при relocation
2. Не обновляет сразу все ссылки (это потребовало бы STW)
3. **Load Barrier** при каждом чтении ссылки из heap проверяет биты:
   - Биты "актуальные" → возвращаем напрямую
   - Биты "устарели" → slow path: lookup в relocation table, heal указатель

Единственные STW: Initial Mark и Final Mark (< 1 мс).
Overhead: Load Barrier (~1-4% throughput).

**Generational ZGC (Java 21+):** `-XX:+ZGenerational` — Young/Old поколения, снижает CPU overhead.

### Shenandoah vs ZGC

| | ZGC | Shenandoah |
|---|---|---|
| Production с | Java 15 | Java 15 |
| Доступность | OpenJDK + Oracle JDK | OpenJDK (RedHat), нет в Oracle JDK |
| Механизм | Colored Pointers + Load Barrier | Brooks Pointer (+8 байт/объект) |
| Паузы STW | < 1 мс | < 1 мс |
| Оптимален для | Очень большие heap (TB) | Средние heap (до ~100 GB) |
| Overhead | ~1-4% throughput | +8 байт/объект, ~2-5% |

---

## ThreadLocal: утечки

```java
// ThreadLocal хранит значения в ThreadLocalMap потока
// Ключи = WeakReference<ThreadLocal>, но значения — сильные ссылки!

// УТЕЧКА в thread pool:
static final ThreadLocal<Connection> DB_CONN = new ThreadLocal<>();
void handleRequest() {
    DB_CONN.set(openConnection());
    // ... забыли DB_CONN.remove()!
    // Поток вернулся в пул, Connection остаётся в ThreadLocalMap навсегда
}

// ПРАВИЛЬНО:
try {
    DB_CONN.set(openConnection());
    doWork();
} finally {
    DB_CONN.remove(); // обязателен!
}
```

Даже если `ThreadLocal` объект собран GC (ключ стал null), **значение остаётся** в ThreadLocalMap пока жив поток. При миллионах virtual threads — тем более.

---

## OOM: виды и решения

| OOM | Причина | Решение |
|---|---|---|
| `Java heap space` | Heap заполнен | `-Xmx`, поиск утечек |
| `GC overhead limit exceeded` | GC тратит > 98% времени | Увеличить heap, найти утечку |
| `Metaspace` | Загружено много классов | `-XX:MaxMetaspaceSize`, CL leaks |
| `Direct buffer memory` | NIO off-heap исчерпан | `-XX:MaxDirectMemorySize` |
| `Unable to create native thread` | Лимит ОС на потоки/стек | `-Xss` снизить, лимиты ОС поднять |
| `CodeCache is full` | JIT код cache исчерпан | `-XX:ReservedCodeCacheSize` |

---

## Диагностика и ключевые флаги

```bash
# GC логи:
-Xlog:gc*:gc.log:time           # подробные GC логи
-Xlog:gc+humongous*             # G1 humongous аллокации

# Heap/Memory:
-Xms4g -Xmx4g                  # одинаковый начальный и максимальный (no resize overhead)
-XX:MaxMetaspaceSize=256m       # ограничить Metaspace
-XX:ReservedCodeCacheSize=256m  # Code Cache для JIT

# Диагностика:
jstat -gcutil <pid> 1s         # метрики GC в реальном времени
jmap -histo:live <pid>         # топ объектов в heap
jmap -dump:live,format=b,file=heap.hprof <pid>  # дамп heap
jcmd <pid> VM.native_memory summary             # native memory

# GC туning:
-XX:InitiatingHeapOccupancyPercent=35  # G1: начать маркировку раньше
-XX:G1HeapRegionSize=16m               # G1: размер региона
-XX:MaxGCPauseMillis=200               # G1: target pause (цель, не гарантия)
```

---

## Вопросы на интервью

- Назови области памяти JVM и что в каждой хранится.
- Где в JVM хранятся значения static-полей? (Heap, в Class mirror)
- Что такое TLAB? Зачем он нужен?
- Чем G1 отличается от Parallel GC? В каких случаях выбрать каждый?
- Что такое Humongous объект в G1? Почему он опасен?
- Что такое Evacuation Failure? Как избежать?
- Как ZGC достигает пауз < 1 мс? Что такое Colored Pointers?
- Чем Shenandoah отличается от ZGC (Brooks Pointer vs Colored Pointers)?
- Почему ThreadLocal вызывает утечки в thread pool?
- Что такое `OutOfMemoryError: Metaspace`? Когда возникает?

---

## Подводные камни

- **Static коллекции = GC Roots** — `static Map<K,V> cache = new HashMap<>()` объекты в нём никогда не будут собраны GC. Используй `WeakHashMap` или явную инвалидацию.
- **ThreadLocal без `remove()`** — thread pool потоки живут вечно, ThreadLocal значения накапливаются. Правило: всегда `remove()` в `finally`.
- **Humongous аллокации** — большие `byte[]` или `Object[]` в G1 могут приводить к Evacuation Failure. Диагностируй через `-Xlog:gc+humongous*`.
- **`-Xms != -Xmx`** — JVM расширяет heap при нагрузке, сжимает при GC. Каждое расширение — STW. В продакшне: `-Xms == -Xmx`.
- **Metaspace без лимита** — по умолчанию Metaspace неограничен (только RAM). CGLIB/ByteBuddy без кэширования генерируют новые классы при каждом прокси-вызове → OOM Metaspace.
- **ZGC Load Barrier overhead** — на read-heavy workloads ZGC может быть медленнее G1 из-за барьеров. Benchmark перед переходом.
- **Full GC после Evacuation Failure** — симптом что приложение работает near capacity. Не игнорируй, даже если "всё работает" — это тикающая бомба под нагрузкой.
