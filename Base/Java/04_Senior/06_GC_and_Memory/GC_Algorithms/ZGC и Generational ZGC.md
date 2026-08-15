# ZGC и Generational ZGC

> **ZGC** (Z Garbage Collector, Java 15 production-ready, JEP 333/377) — low-latency GC с паузами **<1 мс независимо от размера heap** (до 16 ТБ). Достигается через concurrent relocation с colored pointers и load barriers. **Generational ZGC** (Java 21, JEP 439) — добавляет Young/Old поколения для улучшения throughput.

## Связанные темы
[[G1GC — архитектура и tuning]], [[Write Barriers и Card Table]], [[Safepoints и Stop-The-World]], [[JVM флаги для GC]], [[Shenandoah GC]], [[GC Roots и достижимость объектов]], [[Анализ GC логов — JFR, GCEasy]], [[Выбор GC — сравнительная таблица]]

---

## Ключевые идеи ZGC

### 1. Colored Pointers (Окрашенные указатели)

ZGC кодирует метаданные GC прямо в битах pointer:

```
64-bit pointer layout (классический, non-generational ZGC, Java 15-20):
┌────────────────────────────────────────────────┐
│ 18 bits unused │ 4 bits color │ 42 bits address │
└────────────────────────────────────────────────┘

Color bits:
- Marked0    : объект помечен в фазе mark0
- Marked1    : объект помечен в фазе mark1  
- Remapped   : pointer обновлён после relocation
- Finalizable: только finalizable достижим
```

> [!WARNING]
> Это раскладка **классического (non-generational) ZGC**. В Generational ZGC (Java 21+, дефолт с Java 23+) формат colored pointer расширен: отдельные Marked-биты для Young и Old поколений плюс биты remembered set — метаданных больше 4 бит, а не только Marked0/Marked1/Remapped/Finalizable. Конкретная точная бит-раскладка для Generational ZGC меняется между версиями и требует сверки с исходниками (`zAddress.hpp`) под конкретный JDK — не полагайся на цифры выше как на актуальные для дефолтного ZGC сегодня.

Проверка состояния объекта = битовая операция над pointer (без чтения object header).

### 2. Load Barrier

JIT вставляет load barrier при каждом чтении ссылки из heap:

```java
Object ref = obj.field;
// JIT превращает это в:
// Object ref = obj.field;
// if (ref & BAD_MASK != GOOD_COLOR) {
//     ref = slow_path(ref);  // fixup pointer
// }
// return ref;
```

`slow_path` — выполняет relocation или remarking inline. Это позволяет объекту "переехать" между двумя последовательными чтениями одного поля.

### 3. Multi-Mapped Memory

ZGC маппирует одну физическую память по нескольким виртуальным адресам:
- Каждый "color view" (Marked0, Marked1, Remapped) — отдельный диапазон VA
- Физически память одна, но pointer'ы разных цветов указывают в разные VA ranges

## Фазы ZGC

```
Mark Start    (STW, <1 мс)  — создать снимок живых GC Roots
Concurrent Mark              — пометить все живые объекты
Mark End      (STW, <1 мс)  — обработать mutator barriers
Concurrent Process Non-Strong References
Concurrent Reset Relocation Set
Concurrent Select Relocation Set — выбрать регионы для переезда
Relocate Start (STW, <1 мс) — начать перемещение GC Roots
Concurrent Relocate           — переместить объекты
```

Максимум STW — 3 короткие паузы, каждая <1 мс. Всё остальное — concurrent.

## Generational ZGC (Java 21+)

До Java 21 ZGC был "non-generational" — каждый GC cycle обрабатывал весь heap. Generational ZGC добавляет:

```bash
-XX:+UseZGC -XX:+ZGenerational  # Java 21
# В Java 23+ Generational ZGC стал дефолтным для UseZGC
```

**Young Generation** — маленький, собирается часто (Minor GC).
**Old Generation** — большой, собирается редко.

Это даёт:
- ~4x throughput improvement по сравнению с non-gen ZGC
- Сохраняет <1 мс паузы

## Включение и настройка

```bash
# Базовое включение
-XX:+UseZGC

# Generational (Java 21+, рекомендуется)
-XX:+UseZGC -XX:+ZGenerational

# Heap
-Xms16g -Xmx16g  # фиксируй Xms=Xmx

# Concurrent GC threads (default ≈ 25% активных CPU, минимум 1 — эргономика ZHeuristics, без жёсткого потолка)
-XX:ConcGCThreads=4

# GC логирование
-Xlog:gc*:file=zgc.log:time,uptime,level,tags

# Uncommit unused heap memory (возврат ОС)
-XX:+ZUncommit  # default true
-XX:ZUncommitDelay=300  # секунды ожидания перед uncommit

# Heap utilization target (% занятости heap до GC)
# Дефолт автотюнинг — ZGC сам подбирает паузу
```

## ZGC-специфичные метрики в логах

Полный разбор строки `Garbage Collection (Warmup) 3656M(71%)->356M(7%)` и всех фаз (`Mark Start`, `Concurrent Mark`, `Relocate Start`, ...) с пометкой STW/concurrent — см. [[Анализ GC логов — JFR, GCEasy]]. Специфично для (Generational) ZGC:

- Метки `Y:` / `O:` в `[gc,phases]` разделяют записи Young/Old поколения (появилось с Generational ZGC, Java 21+)
- `Small Pages` / `Medium Pages` / `Large Pages` в `[gc,stats]` — статистика по типам ZGC-регионов памяти

## Когда выбирать ZGC

| Сценарий | Рекомендация |
|----------|-------------|
| Latency SLA < 10 мс | ZGC — единственный выбор |
| Heap > 32 GB | ZGC (G1 деградирует) |
| Heap < 1 GB | G1 или ParallelGC |
| Batch processing | ParallelGC (throughput) |
| Стандартное приложение, heap 4-32 GB | G1 |

Полная таблица по всем 6 GC и решающее дерево — [[Выбор GC — сравнительная таблица]].

## Ограничения

- **Throughput**: ~5-15% ниже G1/Parallel из-за load barrier overhead
- **CPU overhead**: concurrent marking/relocation требуют CPU
- **Метаspace**: colored pointers требуют 3x mapping VA space (не физической памяти)
- Java < 15: экспериментальный статус

---

## Вопросы на интервью
- Как ZGC достигает пауз <1 мс независимо от размера heap?
- Что такое colored pointers? Что хранится в этих битах?
- Что такое load barrier и зачем он нужен в ZGC?
- Чем Generational ZGC лучше non-generational?
- В каких случаях стоит выбрать ZGC вместо G1?

## Подводные камни
- ZGC concurrent работает пока приложение работает → надо оставить CPU для GC потоков (ConcGCThreads). На CPU-saturated системе ZGC может не успевать
- Allocation rate > reclamation rate → "Allocation Stall" — ZGC просит мутатора подождать. Это de-facto STW. Решение: увеличь heap или снизь allocation rate
- Colored pointers требуют 48-bit VA space → не работает на 32-bit JVM и некоторых конфигурациях Linux с ограниченным VA space
- Non-generational ZGC (до Java 21) при heap 512 MB работает хуже G1 по throughput — выбирай только при latency требованиях
