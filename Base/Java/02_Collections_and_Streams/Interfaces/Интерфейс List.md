# Интерфейс List в Java

> **List** — упорядоченная коллекция, допускает дубликаты, доступ по индексу. `ArrayList`: O(1) get/set, рост x1.5, cache-friendly. `LinkedList`: O(1) add/remove по краям, O(n) get(i), медленнее на итерации из-за cache misses. В большинстве случаев `ArrayList` предпочтительнее.
> На интервью: `remove(int)` vs `remove(Object)`, механика resize в ArrayList, почему LinkedList медленнее на итерации, `subList()` — view или копия.

## Связанные темы
[[Общая иерархия коллекций]], [[Интерфейсы Iterator и Iterable]], [[Производительность коллекций]]

---

## ArrayList

**Внутренняя структура:** `Object[] elementData` + `int size`.

```java
// Механизм роста:
// При add() в полный массив: newCapacity = oldCapacity + (oldCapacity >> 1) → +50%
// 10 → 15 → 22 → 33 → 49 → 73 → ...
// Arrays.copyOf() = System.arraycopy() = memmove → O(n) но быстрый

// Правильная инициализация:
List<String> list = new ArrayList<>(expectedSize); // избегаем resize
// После bulk-добавления:
list.trimToSize(); // освобождает лишнее место (если важна память)
```

**Big-O:**

| Операция | Сложность | Причина |
|---|---|---|
| `get(i)` / `set(i, v)` | O(1) | Прямой доступ к массиву |
| `add(e)` — в конец | O(1) amortized | Иногда resize |
| `add(i, e)` — в середину | O(n) | `System.arraycopy` сдвиг |
| `remove(i)` | O(n) | Сдвиг элементов влево |
| `contains(o)` / `indexOf(o)` | O(n) | Линейный поиск |
| `size()` | O(1) | Поле |

**Ловушка remove(int) vs remove(Object):**
```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));

list.remove(1);                // remove(int index=1) → удаляет "2", список: [1, 3]
list.remove(Integer.valueOf(1)); // remove(Object) → удаляет значение 1, список: [3]
```

---

## LinkedList

**Внутренняя структура:** `Node<E> { E item; Node<E> prev; Node<E> next; }` + `first` + `last`.

```java
// Реализует и List, и Deque — полноценная двусторонняя очередь
LinkedList<String> list = new LinkedList<>();
list.addFirst("a");  // O(1) — обновление first
list.addLast("b");   // O(1) — обновление last
list.removeFirst();  // O(1)
list.peekFirst();    // O(1) — без удаления
list.get(5);         // O(n) — проход с ближайшего конца
```

**Почему медленнее ArrayList на итерации:** каждый `Node` — отдельный heap-объект (~48 байт). При итерации CPU cache miss на каждом шаге. ArrayList хранит Object[] непрерывно → CPU prefetcher работает.

**Когда LinkedList разумен:** частые insert/delete в начало/конец при наличии Iterator в позиции. Для очередей — `ArrayDeque` лучше.

---

## subList — view, не копия

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c", "d", "e"));
List<String> sub = list.subList(1, 4); // [b, c, d] — view!

sub.set(0, "X");    // list становится [a, X, c, d, e]
sub.add("Y");       // list становится [a, X, c, d, Y, e]
list.remove(0);     // ConcurrentModificationException при следующем обращении к sub!

// Для независимой копии:
List<String> copy = new ArrayList<>(list.subList(1, 4));
// Или очистить диапазон:
list.subList(1, 4).clear(); // O(n) — удаляет элементы с 1 по 3
```

---

## Полезные методы List

```java
List<String> list = new ArrayList<>(List.of("c", "a", "b", "a"));

Collections.sort(list);                      // мутирует список: [a, a, b, c]
list.sort(Comparator.reverseOrder());        // [c, b, a, a]
Collections.binarySearch(sorted, "b");       // O(log n) — только для sorted!
Collections.reverse(list);                  // мутирует список
Collections.shuffle(list);                  // случайный порядок
Collections.frequency(list, "a");           // 2 — количество вхождений
Collections.nCopies(3, "x");               // [x, x, x] — иммутабельный

// Java 8+:
list.replaceAll(String::toUpperCase);        // on-place transform
list.sort(null);                             // использует natural order (Comparable)
```

---

## Collections.unmodifiableList vs List.of

```java
// unmodifiableList — обёртка: оригинальный список виден через неё
List<String> mutable = new ArrayList<>(List.of("a", "b"));
List<String> unmod = Collections.unmodifiableList(mutable);
mutable.add("c");   // unmod теперь содержит ["a", "b", "c"]!

// List.of — полностью иммутабельный, нет backing list
List<String> immutable = List.of("a", "b");
// Ни set(), ни add(), ни remove()
```

---

## Вопросы на интервью

- Чем `remove(int index)` отличается от `remove(Object o)` для `List<Integer>`?
- Как ArrayList расширяется? Что происходит при достижении capacity?
- Почему `subList()` возвращает view? Что произойдёт если модифицировать оригинальный список?
- Когда `LinkedList` предпочтительнее `ArrayList`? (практически никогда — используй `ArrayDeque`)
- Чем `Collections.unmodifiableList()` отличается от `List.of()`?
- Как отсортировать List по нескольким критериям?

---

## Подводные камни

- **`Arrays.asList()` — фиксированный размер** — `add()`/`remove()` → `UnsupportedOperationException`, но `set()` работает. Не путай с `List.of()` (там и `set()` не работает).
- **`subList()` после структурных изменений** — добавление/удаление в оригинальный список после `subList()` → `ConcurrentModificationException` при обращении к sub. Создавай `new ArrayList<>(subList)` если нужна независимость.
- **`ArrayList` и многопоточность** — `add()` без синхронизации из двух потоков может привести к потере данных или бесконечному loop в resize. Используй `CopyOnWriteArrayList` или явную синхронизацию.
- **LinkedList как List передаётся в API требующий RandomAccess** — некоторые алгоритмы (Collections.sort до Java 7, бинарный поиск) проверяют `RandomAccess` и меняют стратегию. LinkedList без RandomAccess → fallback алгоритм.
- **`sort()` vs `Collections.sort()`** — `List.sort()` (Java 8) эффективнее для `ArrayList` (использует `Arrays.sort()` прямо на внутреннем массиве). `Collections.sort()` вызывает `List.sort()` внутри с Java 8. Оба используют TimSort (O(n log n), стабильный).
