# Интерфейс Set в Java

> **Set** — коллекция уникальных элементов. `HashSet`: O(1) add/contains/remove, нет порядка (под капотом — `HashMap`). `LinkedHashSet`: insertion order + O(1). `TreeSet`: O(log n), элементы sorted, под капотом — `TreeMap`. Уникальность — через `equals()` + `hashCode()`, **оба должны быть переопределены!**
> На интервью: почему `HashSet` — это `HashMap` с `PRESENT`, что произойдёт если не переопределить `hashCode`, `NavigableSet` методы TreeSet, реализация Set операций (union, intersection).

## Связанные темы
[[Общая иерархия коллекций]], [[Интерфейс Map]], [[Производительность коллекций]]

---

## Сравнение реализаций

| | `HashSet` | `LinkedHashSet` | `TreeSet` |
|---|---|---|---|
| Под капотом | `HashMap` | `LinkedHashMap` | `TreeMap` |
| Порядок | Нет | Insertion order | Sorted |
| `null` | 1 элемент | 1 элемент | ✗ (NPE) |
| add/contains/remove | O(1) avg | O(1) avg | O(log n) |
| Итерация | O(n), случайный | O(n), insertion order | O(n), sorted |
| `getFirst()`/`getLast()` | ✗ | Java 21 | `first()`/`last()` |

---

## HashSet

Под капотом — `HashMap<E, Object>`, где элементы Set являются ключами HashMap:

```java
public class HashSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object(); // заглушка-значение
}

set.add("apple") → map.put("apple", PRESENT);
set.contains("apple") → map.containsKey("apple");
```

**Инициализация с известным размером:**
```java
// Дефолт: capacity=16, loadFactor=0.75
// Rehash при size > capacity * 0.75
Set<String> set = new HashSet<>(expectedSize * 2); // избегаем rehash
```

**Требования к hashCode/equals:**
```java
// Без переопределения — объекты сравниваются по ссылке (Object.hashCode = identity)
Set<Point> set = new HashSet<>();
set.add(new Point(1, 1));
set.contains(new Point(1, 1)); // false! — разные объекты

// С переопределением:
record Point(int x, int y) {} // record автоматически генерирует equals/hashCode
set.add(new Point(1, 1));
set.contains(new Point(1, 1)); // true
```

---

## TreeSet и NavigableSet API

`TreeSet` реализует `NavigableSet` — богатый API для работы с отсортированными данными:

```java
TreeSet<Integer> set = new TreeSet<>(List.of(1, 3, 5, 7, 9));

// SortedSet методы:
set.first();          // 1 — минимальный
set.last();           // 9 — максимальный
set.headSet(5);       // [1, 3] — строго меньше 5
set.tailSet(5);       // [5, 7, 9] — >= 5
set.subSet(3, 7);     // [3, 5] — [3, 7)

// NavigableSet методы (дополнительно):
set.floor(4);         // 3 — наибольший <= 4
set.ceiling(4);       // 5 — наименьший >= 4
set.lower(5);         // 3 — строго < 5
set.higher(5);        // 7 — строго > 5
set.pollFirst();      // 1 — извлечь и удалить минимальный
set.pollLast();       // 9 — извлечь и удалить максимальный
set.descendingSet();  // [9, 7, 5, 3] — обратный порядок (view!)
```

**Компаратор:**
```java
// Обратный порядок:
TreeSet<String> desc = new TreeSet<>(Comparator.reverseOrder());
// Кастомный:
TreeSet<Person> byAge = new TreeSet<>(Comparator.comparingInt(Person::age));
```

---

## Set операции

```java
Set<String> a = new HashSet<>(Set.of("a", "b", "c"));
Set<String> b = new HashSet<>(Set.of("b", "c", "d"));

// Union (объединение):
Set<String> union = new HashSet<>(a);
union.addAll(b);    // {a, b, c, d}

// Intersection (пересечение):
Set<String> intersection = new HashSet<>(a);
intersection.retainAll(b);  // {b, c}

// Difference (разность):
Set<String> diff = new HashSet<>(a);
diff.removeAll(b);  // {a}

// Symmetric difference (симметричная разность):
Set<String> symDiff = new HashSet<>(union);
symDiff.removeAll(intersection); // {a, d}

// Проверка подмножества:
a.containsAll(intersection); // true
```

---

## LinkedHashSet — когда нужен

```java
// insertion order + O(1) contains
LinkedHashSet<String> visited = new LinkedHashSet<>();
visited.add("home");
visited.add("work");
visited.add("gym");
visited.add("home"); // дубликат — игнорируется, порядок не меняется

// Java 21: SequencedSet методы
visited.getFirst(); // "home"
visited.getLast();  // "gym"
visited.reversed(); // ["gym", "work", "home"] — view
```

---

## Вопросы на интервью

- Почему `HashSet` под капотом использует `HashMap`? Что хранится как значение?
- Что произойдёт если добавить в `HashSet` объект без `equals`/`hashCode`?
- Чем `TreeSet.floor()` отличается от `ceiling()`?
- Как реализовать пересечение двух `Set`? Какая сложность?
- Почему `TreeSet` не допускает `null`?
- Что такое `NavigableSet`? Какие методы он добавляет к `SortedSet`?

---

## Подводные камни

- **`hashCode` без `equals` (или наоборот)** — нарушение контракта: если два объекта `equals` → они должны иметь одинаковый `hashCode`. Если не выполнено: один объект можно добавить дважды (`HashSet` считает их разными).
- **Изменяемые объекты в `HashSet`** — если изменить поле объекта после добавления в Set → его `hashCode` изменится → `contains()` не найдёт его. Используй иммутабельные объекты как элементы Set.
- **`TreeSet` с `Comparator` vs `equals`** — `TreeSet` считает элементы одинаковыми если `comparator.compare(a, b) == 0`. Если Comparator игнорирует какие-то поля — объекты, равные по компаратору, не добавятся в Set, даже если они не `equals`. Контракт: `(compare(a, b) == 0) == a.equals(b)`.
- **`headSet`/`tailSet`/`subSet` — views** — изменения в view отражаются в оригинальном `TreeSet`. `headSet(5).add(6)` → `IllegalArgumentException` (нарушает bound).
- **`LinkedHashSet` vs `TreeSet` для sorted** — `TreeSet` всегда sorted, `LinkedHashSet` — insertion order. Не используй `LinkedHashSet` ожидая сортированный вывод.
