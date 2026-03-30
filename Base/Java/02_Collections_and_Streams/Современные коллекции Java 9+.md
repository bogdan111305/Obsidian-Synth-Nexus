# Современные коллекции Java 9+

> `List.of()`, `Set.of()`, `Map.of()` (Java 9) — структурно неизменяемые, null-запрещён, `Set.of()` не гарантирует порядок. `copyOf()` (Java 10) — иммутабельная копия. `SequencedCollection` (Java 21) — унифицированный интерфейс для доступа к первому/последнему элементу.
> На интервью: разница `List.of()` vs `Arrays.asList()`, почему `null` запрещён в `List.of()`, что такое `SequencedCollection` и зачем, внутренняя реализация компактных массивов.

## Связанные темы
[[Интерфейс List]], [[Интерфейс Map]], [[Интерфейс Set]], [[Общая иерархия коллекций]]

---

## Immutable Collections (Java 9+)

```java
List<String> list = List.of("a", "b", "c");          // [a, b, c]
Set<String>  set  = Set.of("x", "y", "z");           // порядок не гарантирован!
Map<String, Integer> map = Map.of("a", 1, "b", 2);   // до 10 пар

// Более 10 пар:
Map<String, Integer> large = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2)
    // ...
);

// Иммутабельная копия существующей (Java 10):
List<String> copy = List.copyOf(mutableList);
Map<String, Integer> mapCopy = Map.copyOf(hashMap);
```

**Гарантии:**
- `add()`, `remove()`, `set()` → `UnsupportedOperationException`
- `null` элементы → `NullPointerException` (явный дизайн — fast-fail)
- Потокобезопасны для чтения (нет синхронизации, нет изменений)

**Внутренняя реализация:** для `List.of(1-2 элемента)` JVM использует специализированные классы `List12`, для больших — компактный массив без перевыделения. Это делает их быстрее и компактнее `ArrayList`.

---

## List.of() vs Arrays.asList()

| | `List.of()` | `Arrays.asList()` |
|---|---|---|
| `add()` / `remove()` | `UnsupportedOperationException` | `UnsupportedOperationException` |
| `set()` | `UnsupportedOperationException` | **Работает!** |
| `null` элементы | `NullPointerException` | Разрешены |
| Backed by array | Нет | Да (изменения в массиве отражаются) |
| `Collections.unmodifiableList()` | Не нужен | Обёртка с overhead |

```java
String[] arr = {"a", "b", "c"};
List<String> asList = Arrays.asList(arr);
asList.set(0, "X");     // работает!
arr[1] = "Y";           // отражается в asList!

List<String> listOf = List.of("a", "b", "c");
listOf.set(0, "X");     // UnsupportedOperationException
```

---

## SequencedCollection (Java 21, JEP 431)

Единый интерфейс для коллекций с определённым порядком (первый/последний элемент):

```java
// Новая иерархия:
// SequencedCollection → List, Deque, LinkedHashSet
// SequencedMap        → LinkedHashMap, SortedMap

interface SequencedCollection<E> extends Collection<E> {
    void addFirst(E e);
    void addLast(E e);
    E getFirst();
    E getLast();
    E removeFirst();
    E removeLast();
    SequencedCollection<E> reversed(); // обратный порядок без копирования
}
```

```java
// Теперь единообразно для всех ordered коллекций:
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.getFirst();    // "a" (раньше: list.get(0))
list.getLast();     // "c" (раньше: list.get(list.size()-1))

LinkedHashSet<String> set = new LinkedHashSet<>(Set.of("x", "y", "z"));
set.getFirst();     // первый добавленный

LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("a", 1); map.put("b", 2);
map.sequencedEntrySet().reversed();  // обратный порядок

// reversed() — view, не копия:
List<String> rev = list.reversed(); // O(1), изменения в list видны в rev
```

**До Java 21** для `LinkedHashSet` получить последний элемент было невозможно без `iterator()` + перебора.

---

## Новые методы Map (Java 8-21)

```java
Map<String, Integer> map = new HashMap<>();

// Java 8:
map.getOrDefault("x", 0);            // безопасное чтение
map.putIfAbsent("key", 42);          // добавить только если нет
map.computeIfAbsent("key", k -> k.length());  // lazy инициализация
map.computeIfPresent("key", (k, v) -> v * 2); // обновить если есть
map.merge("key", 1, Integer::sum);    // upsert с функцией объединения
map.replaceAll((k, v) -> v * 2);      // обновить все значения

// merge — паттерн подсчёта:
words.forEach(word -> freq.merge(word, 1, Integer::sum));
```

---

## Вопросы на интервью

- Чем `List.of()` отличается от `Arrays.asList()`? Можно ли вызвать `set()` на каждом?
- Почему `null` запрещён в `List.of()`? (явный fast-fail, избегает NPE в get() позже)
- Что такое `SequencedCollection`? Какую проблему решает `reversed()`?
- Как `List.of(1, 2)` отличается от `List.of(1, 2, 3, 4, 5)` внутренне?
- Зачем нужен `Map.ofEntries()` если есть `Map.of()`?
- Что произойдёт при `Set.of("a", "a")`? (`IllegalArgumentException` — дубликаты запрещены)

---

## Подводные камни

- **`Set.of()` с дубликатами** — `Set.of("a", "a")` → `IllegalArgumentException` в рантайме. `Arrays.asList` молча создаёт список с дублями, а `Set.of` явно бросает исключение. Проверяй уникальность перед передачей.
- **`Set.of()` порядок не гарантирован** — два запуска одного кода могут давать разный порядок итерации. Никогда не полагайся на порядок элементов `Set.of()`.
- **`Arrays.asList()` backed by array** — изменения в исходном массиве отражаются в списке и наоборот. При конвертации: `new ArrayList<>(Arrays.asList(arr))` для независимой копии.
- **`copyOf()` не всегда создаёт копию** — если передать уже иммутабельную коллекцию `List.of()`, `List.copyOf()` может вернуть тот же объект. Не полагайся на `list != List.copyOf(list)`.
- **`reversed()` — это view** — `list.reversed().add("x")` изменяет оригинальный список (с противоположного конца). Для независимой копии: `new ArrayList<>(list.reversed())`.
- **`Map.of()` ограничен 10 парами** — при 11+ парах: compile error. Используй `Map.ofEntries(Map.entry(...), ...)`. В Java 9 это дизайнерское решение (varargs Map.entry вместо перегруженных методов).
