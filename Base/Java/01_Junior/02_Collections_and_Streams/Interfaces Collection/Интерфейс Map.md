# Интерфейс Map в Java

> **Map** — уникальные ключи → значения. `HashMap`: O(1) amortized, load factor 0.75, buckets + tree при ≥8 коллизий (Java 8+). `TreeMap`: O(log n), sorted. `LinkedHashMap`: insertion/access order. `ConcurrentHashMap`: lock-free get, CAS/synchronized на бакете.
> На интервью: механика HashMap (hash spreading, treeification), load factor 0.75 из распределения Пуассона, Hash DoS атака, ConcurrentHashMap Java 7 vs Java 8, мутабельный ключ → потеря элемента.

## Связанные темы
[[Общая иерархия коллекций]], [[Интерфейс Set]], [[Производительность коллекций]], [[Reference Types (Weak, Soft, Phantom)]]

---

## Сравнение реализаций

| | `HashMap` | `LinkedHashMap` | `TreeMap` | `ConcurrentHashMap` | `WeakHashMap` |
|---|---|---|---|---|---|
| Порядок | Нет | Insertion/Access | Sorted | Нет | Нет |
| null ключ | 1 | 1 | ✗ | ✗ | Да |
| null значение | ✓ | ✓ | ✓ | ✗ | ✓ |
| get/put | O(1) avg | O(1) avg | O(log n) | O(1) avg | O(1) avg |
| Потокобезопасность | ✗ | ✗ | ✗ | ✓ | ✗ |

---

## HashMap — внутренности

**Структура:** `Node<K,V>[] table` — массив бакетов. Каждый бакет: linked list или TreeNode.

**Алгоритм `put(key, value)`:**
```
1. hash = key.hashCode() ^ (h >>> 16)  // spreading: смешиваем старшие биты с младшими
2. index = (capacity - 1) & hash       // O(1) — битовая маска (capacity всегда степень 2)
3. В бакете index: обход list/tree, сравниваем hash + equals
4. При ≥8 элементов в бакете И table.length ≥ 64 → treeify (list → TreeNode)
5. При size > capacity * 0.75 → resize: capacity *= 2, rehash всех элементов O(n)
```

```java
// Spreading hash — зачем:
// Плохой hashCode дающий одинаковые нижние биты → все в один бакет
// h ^ (h >>> 16) "смешивает" старшие биты → равномернее распределение
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**Правильная инициализация:**
```java
// При известном размере n: нужна capacity = n / 0.75 + 1
Map<String, Data> map = new HashMap<>((int)(n / 0.75) + 1);
// Или: Guava Maps.newHashMapWithExpectedSize(n)
```

---

## Load Factor 0.75 и TREEIFY_THRESHOLD = 8

При случайном хешировании количество элементов в бакете → **Распределение Пуассона** со средним λ ≈ 0.5 (при loadFactor=0.75):

```
P(k) = (λ^k * e^(-λ)) / k!   при λ = 0.5:

P(0) ≈ 0.606  — бакет пуст
P(1) ≈ 0.303
P(2) ≈ 0.076
P(3) ≈ 0.013
...
P(8) ≈ 6×10⁻⁸  ← именно это TREEIFY_THRESHOLD = 8!
```

Если в бакете оказалось 8 элементов при нормальном хешировании — это вероятность 1 на 16 миллионов. Значит hashCode плохой, и создание дерева (дороже в 2x по памяти) оправдано.

| LoadFactor | Коллизии | Resize частота |
|---|---|---|
| 0.5 | Минимальные | Часто (расточительно по памяти) |
| **0.75** | Низкие (λ≈0.5) | **Умеренно (оптимум)** |
| 1.0 | Больше | Редко (деградация до O(n)) |

---

## Hash Collision DoS Attack

**Вектор (2011–2012, 28C3):** До Java 8 — только linked list в бакетах (O(n)).

```java
// Строки с одинаковым hashCode():
"Aa".hashCode() == "BB".hashCode() == 2112
// "AaAa", "AaBB", "BBAa", "BBBB" — все дают hashCode = 2031744
// 1000 таких строк в HTTP POST → HashMap за O(n²) на обработку → CPU 100%
```

**Защита Java 8+:**
1. **Treeification** при ≥8 коллизий → O(log n) вместо O(n)
2. `TREEIFY_THRESHOLD = 8` из Пуассона — при нормальном `hashCode` никогда не достигается

---

## ConcurrentHashMap — Java 7 vs Java 8

**Java 7:** `Segment extends ReentrantLock` — 16 независимых lock'ов (concurrencyLevel=16).
```
put() → lock(segment[i]) → изменить → unlock(segment[i])
size() → lock ALL 16 segments → count → unlock ALL (точно)
```

**Java 8+ — убраны Segment, добавлены CAS + Node lock:**
```java
// put() — упрощённо:
if (bucket == null) {
    casTabAt(tab, i, null, newNode); // CAS — lock-free для пустых бакетов!
} else {
    synchronized (firstNodeInBucket) { // lock только на один Node
        // добавить в list/tree
    }
}
```

| | Java 7 (Segment) | Java 8+ (CAS + Node lock) |
|---|---|---|
| put() в пустой бакет | lock(segment) | CAS — **lock-free** |
| put() при коллизии | lock(segment = 1/16 table) | synchronized(firstNode) |
| get() | volatile read | volatile read |
| size() | lock ALL segments (точно) | `baseCount + counterCells` (приблизительно) |
| Параллелизм записи | ≤ 16 потоков | ≤ capacity потоков |

**`size()` не точный!** При конкуренции использует `LongAdder`-style счётчик: `baseCount + sum(counterCells[])`. Для Long: `mappingCount()`.

---

## LinkedHashMap как LRU Cache

```java
// accessOrder=true: при get() элемент перемещается в конец
Map<String, Data> lru = new LinkedHashMap<>(capacity, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Data> eldest) {
        return size() > capacity; // удалять при превышении размера
    }
};
// Thread-safe нужно обернуть: Collections.synchronizedMap(lru)
// Для production: Caffeine cache
```

---

## TreeMap — NavigableMap API

```java
TreeMap<Integer, String> map = new TreeMap<>();
// ... put elements ...

map.floorKey(5);      // наибольший ключ <= 5
map.ceilingKey(5);    // наименьший ключ >= 5
map.lowerKey(5);      // строго < 5
map.higherKey(5);     // строго > 5
map.headMap(5);       // view: ключи строго < 5
map.tailMap(5);       // view: ключи >= 5
map.subMap(2, 8);     // view: [2, 8)
map.firstKey() / map.lastKey();
map.pollFirstEntry() / map.pollLastEntry(); // извлечь и удалить
map.descendingMap();  // обратный порядок — view!
```

---

## Вопросы на интервью

- Как HashMap вычисляет индекс бакета? Зачем нужен `h ^ (h >>> 16)`?
- Почему loadFactor = 0.75? При каком условии происходит treeification?
- Что такое Hash Collision DoS атака? Как Java 8 от неё защищается?
- Чем ConcurrentHashMap Java 8 отличается от Java 7?
- Почему `ConcurrentHashMap.size()` не точный?
- Что произойдёт если изменить ключ HashMap после `put()`?

---

## Подводные камни

- **Мутабельный ключ** — изменение поля влияющего на `hashCode()` после `put()` → `get()` вернёт `null`, элемент "потерян" в старом бакете. Всегда иммутабельные ключи: `String`, `Integer`, `record`.
- **`ConcurrentHashMap` без `null`** — `put(null, v)` и `put(k, null)` → `NullPointerException`. Дизайн: `null` не может отличить "нет ключа" от "ключ с null значением" при concurrent get.
- **`size()` ConcurrentHashMap как условие** — `if (map.size() == 0)` в многопоточном коде ненадёжно. Между `size()` и следующим действием другой поток может добавить элемент. Используй `isEmpty()` или другие атомарные операции.
- **`compute()` блокирует бакет** — `computeIfAbsent(key, fn)` держит lock на бакет пока выполняется `fn`. Если `fn` вызывает другие операции на той же CHM → **deadlock**. Не делай рекурсивных обращений в `compute`.
- **`HashMap` при resize в многопоточном коде (Java 7)** — старый bagged: circular link в linked list при concurrent resize → `get()` в бесконечном цикле. Java 8 исправил, но не делает HashMap thread-safe. Симптом: 100% CPU на `get()`.
- **`TreeMap` headMap/tailMap/subMap — views** — изменения во view изменяют оригинал. Вставка элемента за пределами диапазона → `IllegalArgumentException`.
