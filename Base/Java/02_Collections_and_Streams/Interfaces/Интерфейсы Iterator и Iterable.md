# Интерфейсы Iterator и Iterable в Java

> `Iterable` = умеет возвращать `Iterator`. `Iterator` = имеет `hasNext()`/`next()`/`remove()`. `for-each` компилируется в вызов `iterator()`. Безопасное удаление при итерации — только через `iterator.remove()` или `removeIf()`.
> На интервью: как `for-each` компилируется в байт-код, механика `ConcurrentModificationException` через `modCount`, `Spliterator` vs `Iterator`, реализация custom `Iterable`.

## Связанные темы
[[Общая иерархия коллекций]], [[Java Stream API & Functional Programming]], [[Интерфейс List]]

---

## Iterable и Iterator

```java
// Iterable<T> — может вернуть Iterator
interface Iterable<T> {
    Iterator<T> iterator();
    default void forEach(Consumer<? super T> action) { ... }  // Java 8
    default Spliterator<T> spliterator() { ... }              // Java 8, для Stream
}

// Iterator<T> — обход с позицией
interface Iterator<T> {
    boolean hasNext();
    T next();                                                  // NoSuchElementException если нет
    default void remove() { throw new UnsupportedOperationException(); }
    default void forEachRemaining(Consumer<? super T> action) { ... } // Java 8
}
```

**for-each компилируется в Iterator:**
```java
for (String s : list) { process(s); }
// ↓ bytecode эквивалент:
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    process(s);
}
```

---

## modCount и ConcurrentModificationException

`ArrayList`, `HashMap` и другие коллекции хранят счётчик `modCount` — инкрементируется при каждом структурном изменении (add, remove, но не set).

```java
// ArrayList.Itr (упрощённо):
private class Itr implements Iterator<E> {
    int cursor = 0;
    int expectedModCount = modCount; // запомнить на момент создания

    public E next() {
        if (modCount != expectedModCount)   // проверка при каждом next()
            throw new ConcurrentModificationException();
        return elementData[cursor++];
    }

    public void remove() {
        ArrayList.this.remove(cursor - 1);
        expectedModCount = modCount; // синхронизировать после своего удаления
    }
}
```

**Безопасные способы изменения при итерации:**
```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// ❌ CME:
for (String s : list) { if (s.equals("b")) list.remove(s); }

// ✓ Iterator.remove():
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // синхронизирует expectedModCount
}

// ✓ removeIf() (Java 8) — проще и эффективнее:
list.removeIf(s -> s.equals("b"));

// ✓ Stream.filter() — создаёт новый список:
List<String> filtered = list.stream().filter(s -> !s.equals("b")).toList();
```

**CME — fail-fast, не гарантия.** Не полагайся на него в многопоточном коде — используй `ConcurrentHashMap`, `CopyOnWriteArrayList`.

---

## Custom Iterable

```java
// Пагинированный источник данных:
public class PagedIterable<T> implements Iterable<T> {
    private final Function<Integer, List<T>> pageLoader;
    private final int pageSize;

    @Override
    public Iterator<T> iterator() {
        return new Iterator<T>() {
            private int page = 0;
            private List<T> current = List.of();
            private int index = 0;

            @Override
            public boolean hasNext() {
                if (index < current.size()) return true;
                current = pageLoader.apply(page++);
                index = 0;
                return !current.isEmpty();
            }

            @Override
            public T next() {
                if (!hasNext()) throw new NoSuchElementException();
                return current.get(index++);
            }
        };
    }
}

// Использование:
for (User user : new PagedIterable<>(page -> userService.getPage(page, 100), 100)) {
    process(user);
}
```

---

## Spliterator — параллельная итерация

`Spliterator` (Java 8) используется `Stream` для разбиения данных на части для `parallelStream()`:

```java
interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);  // один элемент
    void forEachRemaining(Consumer<? super T> action);
    Spliterator<T> trySplit();                        // разделить пополам
    long estimateSize();
    int characteristics();  // SIZED, ORDERED, DISTINCT, SORTED, IMMUTABLE, ...
}
```

```java
// Создание Spliterator из коллекции:
List<String> list = List.of("a", "b", "c", "d");
Spliterator<String> spliterator = list.spliterator();

Spliterator<String> left = spliterator.trySplit();  // первая половина
// spliterator — вторая половина

// parallelStream() использует это для разбиения на задачи ForkJoinPool:
list.parallelStream().forEach(s -> process(s));
```

**Characteristics** определяют возможные оптимизации:
```
SIZED     — размер известен заранее (ArrayList → да, LinkedList → нет в некоторых случаях)
ORDERED   — порядок определён (List → да, HashSet → нет)
DISTINCT  — нет дубликатов (Set → да)
SORTED    — элементы отсортированы (TreeSet → да)
IMMUTABLE — не поддерживает remove (List.of() → да)
```

---

## ListIterator — двунаправленный

```java
// ListIterator (только для List) — движение вперёд и назад:
ListIterator<String> lit = list.listIterator(list.size()); // с конца

while (lit.hasPrevious()) {
    System.out.println(lit.previous()); // обратный порядок
}

// Методы:
// nextIndex() / previousIndex() — текущая позиция
// set(E e) — заменить последний элемент (next() или previous())
// add(E e) — вставить перед следующим элементом
```

---

## Вопросы на интервью

- Как `for-each` компилируется в байт-код? Что происходит если класс не реализует `Iterable`?
- Как работает `ConcurrentModificationException`? Что такое `modCount`?
- Почему `iterator.remove()` не бросает CME, а `list.remove()` внутри цикла — бросает?
- Чем `Spliterator` отличается от `Iterator`? Как он используется в параллельных стримах?
- Можно ли использовать `for-each` для `Map`? (Нет, Map не Iterable, но `entrySet()` — да)
- Что такое `ListIterator`? Какие возможности он добавляет?

---

## Подводные камни

- **`iterator.remove()` без предшествующего `next()`** — `IllegalStateException`. Нельзя вызвать `remove()` дважды подряд или перед первым `next()`.
- **`Arrays.asList()` — remove через Iterator не работает** — `Arrays.asList()` возвращает список фиксированного размера, `iterator.remove()` → `UnsupportedOperationException`.
- **Итератор — одноразовый** — после полного обхода повторный вызов `hasNext()` → `false`. Для нового обхода нужен новый `iterator()` вызов.
- **CME не thread-safe** — `ConcurrentModificationException` — это fail-fast механизм для однопоточного кода. Он НЕ гарантирует обнаружение гонок в многопоточной среде. Для thread-safe итерации — `CopyOnWriteArrayList` или `ConcurrentHashMap`.
- **`forEach` vs `for-each` при модификации** — `list.forEach(item -> list.remove(item))` тоже бросает CME. `forEach` реализован через Iterator внутри.
- **Параллельный поток на не-thread-safe коллекции** — `new ArrayList<>().parallelStream()` может дать некорректные результаты если другой поток модифицирует список. `parallelStream` предполагает что источник не изменяется во время выполнения.
