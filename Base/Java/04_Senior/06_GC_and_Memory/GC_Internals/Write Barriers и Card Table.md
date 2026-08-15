# Write Barriers и Card Table

> **Write barrier** — код, вставляемый JIT перед каждой записью ссылки (`obj.field = ref`). Нужен GC: без него concurrent collector не знает, что старый объект теперь ссылается на новый, который ещё не отсканирован. **Card Table** — компактная структура для отслеживания dirty regions heap.

## Связанные темы
[[GC Roots и достижимость объектов]], [[Safepoints и Stop-The-World]], [[G1GC — архитектура и tuning]], [[ZGC и Generational ZGC]]

---

## Проблема, которую решают write barriers

GC должен найти все ссылки из **Old Generation** в **Young Generation** (inter-generational pointers). Иначе при Minor GC объект в Young будет считаться недостижимым, хотя на него ссылается long-lived объект.

```
Old Gen:  [LongLivedObj] ──ref──> [YoungObj]  ← без write barrier GC не знает об этой ссылке
```

Без write barrier нужно сканировать весь Old Gen при каждом Minor GC → O(heap) вместо O(young).

## Card Table

Heap делится на **карточки** по 512 байт. Каждая карточка — 1 байт в Card Table (bitmap). Write barrier при записи ссылки помечает карточку как dirty.

```
Heap:    [   512b   ][   512b   ][   512b   ][   512b   ]
CardTbl: [    0     ][    1     ][    0     ][    0     ]
                           ↑
                    dirty — здесь была запись ссылки
```

При Minor GC сканируются только dirty карточки (содержат ссылки из Old в Young) → GC Roots расширяются.

## Реализация write barrier в HotSpot

```
// Псевдокод что JIT вставляет при obj.field = newValue:

// --- Pre-write barrier (Snapshot-At-The-Beginning, SATB для G1) ---
// запоминает старое значение поля до перезаписи
if (oldValue != null && SATB_active) {
    satb_log_queue.enqueue(oldValue);  // для concurrent marking
}

// --- Собственно запись ---
obj.field = newValue;

// --- Post-write barrier (Card Table dirty marking) ---
if (newValue != null && isOldGen(obj) && isYoungGen(newValue)) {
    card_table[addr(obj) >> 9] = DIRTY;  // 512 = 2^9
}
```

> [!INFO]
> G1 использует **SATB (Snapshot-At-The-Beginning)** barrier: при concurrent marking запоминает ВСЕ старые значения ссылок, которые перезаписываются. Это гарантирует, что объект, на который ссылались в начале фазы, не будет пропущен.

## Remembered Sets (RSet) в G1

G1 вместо глобальной Card Table использует **per-region Remembered Set**: каждый region хранит список card'ов других регионов, которые ссылаются на него.

```
Region A (Old) ──ref──> Region B (Young)
                               ↑
                    В RSet[B] записан card из Region A
```

Преимущество: при Evacuation региона B нужно сканировать только его RSet, а не весь heap.

Цена: write barrier дороже (обновление RSet, concurrent refinement threads), RSet занимает память (~20% overhead в худшем случае).

## ZGC и Shenandoah — Load Barriers

Конкурентные *compacting* GC (в отличие от G1/Serial/Parallel, которые только помечают dirty-ссылки write barrier'ом) используют ещё и **load barriers** — код при каждом чтении ссылки. Это нужно, потому что compacting GC двигает объекты, пока приложение работает: объект может "переехать" между двумя последовательными чтениями одного и того же поля, и barrier обязан на лету это заметить и исправить pointer.

Оба алгоритма прячут состояние relocation в разных местах:

- **ZGC** — в битах самого pointer'а (**colored pointers**): barrier — это проверка нескольких бит без обращения к object header. Полный layout битов и псевдокод barrier'а — [[ZGC и Generational ZGC]].
- **Shenandoah** — в дополнительном forwarding-слове в заголовке объекта (**Brooks pointer**): barrier разыменовывает это слово при каждом доступе. Детали layout заголовка — [[Shenandoah GC]].

Разница в подходе — почему у ZGC нет per-object memory overhead (цена — 3x virtual address space), а у Shenandoah есть (+8 байт на объект), но нет multi-mapping. Сравнение обоих механизмов и цены на throughput — см. таблицу ниже.

## Сравнение подходов

| GC | Pre-write | Post-write | Load | Overhead |
|----|-----------|------------|------|----------|
| G1 | SATB (conditional) | Card Table | Нет | ~10-15% throughput |
| ZGC | Нет | Нет | Colored ptr check | ~5-10% throughput |
| Shenandoah | SATB | Нет | Brooks ptr deref | ~10-15% throughput |
| Serial/Parallel | Нет | Card Table | Нет | ~1-3% throughput |

## Concurrent Refinement (G1)

G1 запускает фоновые **Concurrent Refinement Threads** — они обрабатывают dirty cards из буфера и обновляют RSets не во время STW, а во время работы приложения.

```bash
# Количество refinement threads
-XX:G1ConcRefinementThreads=8  # default = количество ядер / 4

# Если refinement не успевает → мутирующие потоки помогают (Mutator Refinement)
# Признак в логах: "Concurrent RS Threads times"
```

---

## Вопросы на интервью
- Зачем нужен write barrier? Что произойдёт без него?
- Чем Card Table отличается от Remembered Set?
- Что такое SATB barrier и почему он нужен concurrent GC?
- Как ZGC обходится без write barrier для old→young references?
- Что такое colored pointers в ZGC?

## Подводные камни
- Write barriers — причина, почему "allocation-free" код всё равно имеет GC overhead при ссылочных присваиваниях
- Большое количество cross-region references в G1 → огромные RSets → OOM в JVM internal memory
- SATB barrier может создавать лишние объекты в очереди при высокочастотных обновлениях ссылок → concurrent marking отстаёт → Evacuation Failure
- В ZGC load barrier в tight loops заметен в профайлере — не нулевой overhead
