# Shenandoah GC

> **Shenandoah** (JEP 189, Java 12 experimental, Java 15 production) — low-latency concurrent compacting GC от Red Hat. Паузы <10 мс независимо от heap (на практике 1-5 мс). Конкурирует с ZGC по latency, но использует иной подход: **Brooks pointers** вместо colored pointers.

## Связанные темы
[[ZGC и Generational ZGC]], [[G1GC — архитектура и tuning]], [[Write Barriers и Card Table]], [[JVM флаги для GC]]

---

## Brooks Pointers — ключевая идея

Каждый объект получает дополнительное слово в заголовке — **forwarding pointer**:

```
Normal object header layout (Shenandoah):
┌──────────────────────────────────────────────────┐
│ Brooks Pointer (8 bytes) → сам объект или новый │  ← forwarding ptr
├──────────────────────────────────────────────────┤
│ Mark Word (8 bytes)                              │
├──────────────────────────────────────────────────┤
│ Klass Word (8 bytes)                             │
├──────────────────────────────────────────────────┤
│ ... fields ...                                   │
└──────────────────────────────────────────────────┘
```

- До relocation: Brooks pointer → сам объект (self-referential)
- После копирования: Brooks pointer → новое местонахождение

Любой доступ к объекту проходит через Brooks pointer — load barrier читает его:

```java
Object ref = obj.field;
// Barrier:
// Object actual = ref.brooks_ptr;  // следуем forwarding pointer
// return actual.field;
```

## Read vs Write Barriers

Shenandoah использует оба:

- **Load barrier**: разыменование Brooks pointer при чтении
- **Store barrier** (pre-write SATB): для concurrent marking
- **CAS barrier**: при compare-and-swap нужно работать с реальным местонахождением

## Фазы Shenandoah

```
Init Mark       (STW, <2 мс)  — пометить GC Roots
Concurrent Mark (concurrent)  — маркировать граф достижимости
Final Mark      (STW, <2 мс)  — завершить marking, выбрать evacuation set
Concurrent Evac (concurrent)  — копировать объекты, обновлять Brooks ptr
Init Update Refs (STW, <1 мс) — начать обновление ссылок
Concurrent Update Refs        — пройти heap, заменить старые адреса на новые
Final Update Refs (STW, <2 мс)— обновить GC Roots
Concurrent Cleanup            — освободить старые регионы
```

4 STW паузы, каждая очень короткая. Concurrent evacuation — уникальная особенность: и GC, и приложение одновременно работают с объектами в процессе переезда.

## Shenandoah vs ZGC

| Характеристика | Shenandoah | ZGC |
|----------------|-----------|-----|
| Механизм | Brooks ptr (extra word) | Colored pointers (bits) |
| Barrier type | Load + Store | Load only |
| Heap overhead | ~2 слова на объект | Нет per-object overhead |
| VA overhead | Нет triple-mapping | 3x VA space |
| Доступность | OpenJDK + Red Hat | OpenJDK |
| Generational | Нет (Java 21+: экспериментально) | Да (Java 21+) |
| Производительность | Сопоставимо с ZGC | Немного лучше на большом heap |

## Настройка

```bash
# Включение
-XX:+UseShenandoahGC

# Eviction heuristic
-XX:ShenandoahGCHeuristics=adaptive   # default — адаптивный
-XX:ShenandoahGCHeuristics=aggressive # GC как можно чаще — тесты latency
-XX:ShenandoahGCHeuristics=compact    # минимальный footprint
-XX:ShenandoahGCHeuristics=static     # фиксированный порог

# Порог для GC (% heap до начала GC цикла)
-XX:ShenandoahInitFreeThreshold=75  # начать GC если free > 75%
-XX:ShenandoahMinFreeThreshold=10   # срочный GC если free < 10%

# Concurrent threads
-XX:ConcGCThreads=4

# Отключить Generational (если нужен классический Shenandoah)
# В Java 21 можно попробовать экспериментальный Generational
```

## Диагностика

```bash
-Xlog:gc*:file=shenandoah.log:time,uptime,level,tags

# Пример вывода:
# [gc] GC(0) Concurrent reset
# [gc] GC(0) Pause Init Mark 1.234ms  ← STW
# [gc] GC(0) Concurrent marking
# [gc] GC(0) Pause Final Mark 1.123ms ← STW
# [gc] GC(0) Concurrent evacuation
# [gc] GC(0) Pause Init Update Refs 0.543ms ← STW
# [gc] GC(0) Concurrent update references
# [gc] GC(0) Pause Final Update Refs 0.876ms ← STW
# [gc] GC(0) Concurrent cleanup
```

## Когда выбирать Shenandoah

- Приложение на Red Hat OpenJDK (Shenandoah там более протестирован)
- Heap 4-64 GB, latency SLA < 10 мс
- Если ZGC недоступен (Java 11/12 environments)
- Микросервисы где heap overhead (Brooks ptr ~16 байт на объект) приемлем

---

## Вопросы на интервью
- Что такое Brooks pointer и как он обеспечивает concurrent relocation?
- Чем Shenandoah отличается от ZGC по механизму работы?
- Сколько STW пауз в Shenandoah GC цикле?
- Какой overhead несут Brooks pointers?

## Подводные камни
- Brooks pointer = +8 байт на каждый объект → увеличенный heap footprint. На heap с миллиардами мелких объектов это заметно
- Concurrent Update Refs — дорогая фаза при большом heap (нужно пройти весь живой граф ссылок). Если она растёт — allocation rate слишком высокий
- Shenandoah агрессивно использует CPU на concurrent фазы — на ограниченных CPU (контейнеры с 1-2 vCPU) может деградировать
- В OpenJDK сборке Oracle Shenandoah может быть недоступен — нужна Eclipse Temurin/Adoptium или Red Hat build
