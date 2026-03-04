---
title: "Java — Интерфейс Set (HashSet, LinkedHashSet, TreeSet)"
tags: [java, collections, set, hashset, treeset]
updated: 2026-03-04
---

# Интерфейс Set в Java

## 1. Обзор интерфейса `Set<E>`

`Set<E>` — это коллекция, которая:

- **Не допускает дубликаты**: Проверяет уникальность через `equals()` и `hashCode()`.
- **Не поддерживает индексы**: Нет методов вроде `get(int index)`.
- **Поддерживает базовые операции**: Добавление (`add`), удаление (`remove`), проверка наличия (`contains`), итерация.

**Основные методы**:

- `add(E e)`: Добавляет элемент, если его нет (возвращает `true` при успехе).
- `remove(Object o)`: Удаляет элемент.
- `contains(Object o)`: Проверяет наличие элемента.
- `size()`: Возвращает количество элементов.
- `iterator()`: Возвращает итератор для перебора.

**Реализации**:

1. `HashSet<E>`: Хэш-таблица, быстрая, без порядка.
2. `LinkedHashSet<E>`: Хэш-таблица с сохранением порядка вставки.
3. `TreeSet<E>`: Красно-черное дерево, сортировка элементов.

**Иерархия**:

```
Collection<E>
└── Set<E>
    ├── HashSet<E>
    │   └── LinkedHashSet<E>
    └── SortedSet<E>
        └── NavigableSet<E>
            └── TreeSet<E>
```

## 2. Реализации интерфейса `Set`

### 2.1. `HashSet<E>`

`HashSet` — это реализация `Set`, использующая хэш-таблицу на основе `HashMap`. Подходит для задач, где важна высокая производительность операций и порядок элементов не имеет значения.

#### Внутреннее устройство

`HashSet` хранит элементы как ключи в `HashMap`:

```java
public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, Serializable {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();
}
```

- Элементы `HashSet` — ключи в `map`.
- Значение — фиктивный объект `PRESENT`.
- Хэш-таблица состоит из массива корзин (`Node<K,V>[]`), где элементы распределяются по индексу, вычисленному через `hashCode()`.

**Добавление элемента**:

```java
set.add("A") → map.put("A", PRESENT);
```

#### Хранение элементов

- Индекс корзины: `hashCode() % capacity`.
- При коллизиях (несколько элементов в одной корзине):
    - До Java 8: Связный список (O(n) в худшем случае).
    - С Java 8: Если корзина содержит ≥8 элементов и таблица ≥64, используется красно-черное дерево (O(log n)).

#### Механизм расширения

- Начальная емкость: 16, коэффициент загрузки (`loadFactor`): 0.75.
- При `size > capacity * loadFactor`:
    - Емкость увеличивается вдвое (`16 → 32 → 64`).
    - Все элементы рехэшируются (O(n)).
- Рехэширование — затратная операция, но амортизируется.

#### Основные операции и Big-O

|Операция|Метод|Средний случай|Худший случай|Пояснения|
|---|---|---|---|---|
|Добавление|`add(E e)`|O(1)|O(n)|Хэш-таблица, коллизии редки|
|Удаление|`remove(Object o)`|O(1)|O(n)|Поиск по хэшу и `equals()`|
|Проверка наличия|`contains(Object o)`|O(1)|O(n)|Поиск по хэшу|
|Итерация|`iterator()`|O(n)|O(n)|Обход всех элементов|

- **O(1) среднее**: Предполагает хорошую хэш-функцию.
- **O(n) худшее**: Все элементы в одной корзине (плохой `hashCode()`).

#### Пример

```java
import java.util.HashSet;
import java.util.Set;

Set<String> set = new HashSet<>();
set.add("apple");       // O(1)
set.add("banana");
set.add("apple");       // Игнорируется (дубликат)
System.out.println(set.contains("banana")); // O(1), true
set.remove("banana");   // O(1)
System.out.println(set); // [apple], порядок не гарантирован
```

### 2.2. `LinkedHashSet<E>`

`LinkedHashSet` — это расширение `HashSet`, которое сохраняет порядок вставки элементов. Использует `LinkedHashMap` для хранения.

#### Внутреннее устройство

```java
public class LinkedHashSet<E> extends HashSet<E> implements Set<E>, Cloneable, Serializable {
    public LinkedHashSet() {
        super(new LinkedHashMap<>());
    }
}
```

- Основан на `LinkedHashMap`, который сочетает хэш-таблицу и двусвязный список.
- Каждый элемент (`Entry`) содержит ссылки `next` и `prev` для порядка вставки.

#### Механизм расширения

- Аналогичен `HashSet`: удваивает емкость при `size > capacity * loadFactor`.
- Двусвязный список обновляется при рехэшировании.

#### Основные операции и Big-O

|Операция|Метод|Средний случай|Худший случай|Пояснения|
|---|---|---|---|---|
|Добавление|`add(E e)`|O(1)|O(n)|Хэш + обновление списка|
|Удаление|`remove(Object o)`|O(1)|O(n)|Хэш + обновление списка|
|Проверка наличия|`contains(Object o)`|O(1)|O(n)|Поиск по хэшу|
|Итерация|`iterator()`|O(n)|O(n)|Обход двусвязного списка|

- Сохранение порядка добавляет небольшие накладные расходы по сравнению с `HashSet`.

#### Пример

```java
import java.util.LinkedHashSet;
import java.util.Set;

LinkedHashSet<String> set = new LinkedHashSet<>();
set.add("apple");       // O(1)
set.add("banana");
set.add("cherry");
set.add("banana");      // Игнорируется
for (String s : set) {
    System.out.println(s); // apple, banana, cherry (порядок вставки)
}
```

### 2.3. `TreeSet<E>`

`TreeSet` реализует `NavigableSet` (расширение `SortedSet`), основан на красно-черном дереве, поддерживает сортировку элементов.

#### Внутреннее устройство

```java
public class TreeSet<E> extends AbstractSet<E> implements NavigableSet<E>, Cloneable, Serializable {
    private transient NavigableMap<E, Object> m;
    private static final Object PRESENT = new Object();
}
```

- Основан на `TreeMap`, где элементы — ключи, а значения — `PRESENT`.
- Красно-черное дерево — самобалансирующееся бинарное дерево поиска.
- Элементы сортируются по `Comparable` или `Comparator`.

#### Механизм расширения

- Не использует массив, растет динамически с добавлением узлов.
- Балансировка дерева (ротации) обеспечивает O(log n).

#### Основные операции и Big-O

|Операция|Метод|Сложность|Пояснения|
|---|---|---|---|
|Добавление|`add(E e)`|O(log n)|Вставка с балансировкой|
|Удаление|`remove(Object o)`|O(log n)|Удаление с балансировкой|
|Проверка наличия|`contains(Object o)`|O(log n)|Поиск в дереве|
|Итерация|`iterator()`|O(n)|In-order обход дерева|
|Мин/макс элемент|`first()`/`last()`|O(log n)|Доступ к краям дерева|
|Навигация (`ceiling`/`floor`)|`ceiling(E e)`|O(log n)|Поиск ближайшего элемента|

- **O(log n)**: Гарантируется балансировкой дерева.
- **Null**: Не допускается, если элементы используют `Comparable` (иначе `NullPointerException`).

#### Пример

```java
import java.util.TreeSet;

TreeSet<Integer> set = new TreeSet<>();
set.add(30);
set.add(10);
set.add(20);
for (Integer num : set) {
    System.out.println(num); // 10, 20, 30 (отсортировано)
}
System.out.println(set.first()); // 10
System.out.println(set.contains(20)); // true
set.remove(20); // O(log n)
```

## 3. Сравнение реализаций

|Характеристика|`HashSet`|`LinkedHashSet`|`TreeSet`|
|---|---|---|---|
|**Структура**|Хэш-таблица (`HashMap`)|Хэш-таблица + список|Красно-черное дерево (`TreeMap`)|
|**Сортировка**|Нет|Нет|Да (по `Comparable`/`Comparator`)|
|**Порядок вставки**|Нет|Да|Нет (сортированный порядок)|
|**Null**|Да (1)|Да (1)|Нет (если `Comparable`)|
|**Потокобезопасность**|Нет|Нет|Нет|
|**Добавление/удаление**|O(1) среднее|O(1) среднее|O(log n)|
|**Проверка наличия**|O(1) среднее|O(1) среднее|O(log n)|
|**Итерация**|O(n)|O(n)|O(n)|
|**Память**|Средняя|Больше (`prev`/`next`)|Средняя (узлы дерева)|

## 4. Внутренняя реализация в JVM

- **`HashSet`**:
    
    - Хранит элементы в массиве корзин (`Node[]`) в куче.
    - Хэш-код вычисляется через `hashCode()` с дополнительным смешиванием для уменьшения коллизий.
    - В Java 8+ корзины с ≥8 элементами преобразуются в красно-черное дерево.
    - Байт-код: Операции `add`/`contains` используют `invokevirtual` для вызова `HashMap.put`/`get`.
- **`LinkedHashSet`**:
    
    - Аналогично `HashSet`, но каждый узел (`Entry`) содержит ссылки `prev`/`next`.
    - Двусвязный список увеличивает расход памяти.
    - Итерация следует порядку списка, а не корзинам.
- **`TreeSet`**:
    
    - Хранит элементы в красно-черном дереве (узлы с `left`, `right`, `parent`, `color`).
    - Балансировка (ротации, перекраска узлов) обеспечивает O(log n).
    - Байт-код: Операции используют сравнения через `Comparable.compareTo` или `Comparator.compare`.

## 5. Производительность

- **`HashSet`**:
    
    - **Преимущества**: Высокая скорость операций (O(1) в среднем) благодаря хэшированию.
    - **Недостатки**: Плохой `hashCode()` может привести к O(n) из-за коллизий.
    - Оптимизация в Java 8 (деревья вместо списков) снижает худший случай до O(log n).
- **`LinkedHashSet`**:
    
    - **Преимущества**: Сохранение порядка вставки, производительность близка к `HashSet`.
    - **Недостатки**: Дополнительная память для двусвязного списка, небольшие накладные расходы.
- **`TreeSet`**:
    
    - **Преимущества**: Гарантированная сортировка, навигационные методы (`ceiling`, `floor`).
    - **Недостатки**: O(log n) медленнее O(1), не поддерживает `null` с `Comparable`.

**Рекомендация**:

- Используйте `HashSet` для максимальной производительности.
- Используйте `LinkedHashSet` для сохранения порядка вставки.
- Используйте `TreeSet` для сортировки или навигации.

## 6. Многопоточность

`HashSet`, `LinkedHashSet` и `TreeSet` **не потокобезопасны**. Для многопоточных приложений:

- **Синхронизированные обертки**:
    
    ```java
    Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());
    ```
    
- **ConcurrentSkipListSet** (Java 6+):
    
    - Потокобезопасная альтернатива `TreeSet` на основе skip-list.
    
    ```java
    import java.util.concurrent.ConcurrentSkipListSet;
    
    Set<String> set = new ConcurrentSkipListSet<>();
    new Thread(() -> set.add("A")).start();
    new Thread(() -> System.out.println(set.contains("A"))).start();
    ```
    
- **CopyOnWriteArraySet** (Java 5+):
    
    - Копия множества при каждой модификации, подходит для частого чтения.
    
    ```java
    import java.util.concurrent.CopyOnWriteArraySet;
    
    Set<String> set = new CopyOnWriteArraySet<>();
    set.add("A");
    ```
    
- **Параллельные стримы**:
    
    ```java
    Set<Integer> set = new HashSet<>(Arrays.asList(1, 2, 3));
    set.parallelStream().forEach(System.out::println);
    ```
    
    **Ограничение**: Избегайте модификации множества в `parallelStream().forEach()`.
    

## 7. Современные возможности (Java 8+)

### 7.1. Стримы

`Stream API` упрощает обработку множеств:

```java
Set<String> set = new HashSet<>(Arrays.asList("apple", "banana", "cherry"));
set.stream()
   .filter(s -> s.startsWith("a"))
   .map(String::toUpperCase)
   .forEach(System.out::println); // APPLE
```

### 7.2. Неизменяемые множества (Java 9+)

Метод `Set.of()` создает неизменяемое множество:

```java
Set<String> immutableSet = Set.of("A", "B");
immutableSet.add("C"); // UnsupportedOperationException
```

### 7.3. Навигационные методы в `TreeSet`

`TreeSet` поддерживает методы из `NavigableSet`:

```java
TreeSet<Integer> set = new TreeSet<>(Arrays.asList(10, 20, 30));
System.out.println(set.ceiling(15)); // 20
System.out.println(set.floor(25));   // 20
```

## 8. Подводные камни

1. **Дубликаты и `equals`/`hashCode`**:
    
    - Неправильная реализация `hashCode()` или `equals()` может привести к дубликатам.
    
    ```java
    class Key {
        String id;
        @Override public int hashCode() { return id.hashCode(); }
        @Override public boolean equals(Object o) { /* логика */ }
    }
    ```
    
2. **Модификация при итерации**:
    
    ```java
    Set<String> set = new HashSet<>(Arrays.asList("A", "B"));
    for (String s : set) {
        set.remove(s); // ConcurrentModificationException
    }
    ```
    
    **Решение**: Используйте итератор:
    
    ```java
    Iterator<String> iterator = set.iterator();
    while (iterator.hasNext()) {
        iterator.next();
        iterator.remove();
    }
    ```
    
3. **Null в `TreeSet`**:
    
    - `TreeSet` выбрасывает `NullPointerException` для `null` с `Comparable`.
    
    ```java
    TreeSet<String> set = new TreeSet<>();
    set.add(null); // NullPointerException
    ```
    
4. **Плохой `hashCode()`**:
    
    - Плохо распределенный `hashCode()` увеличивает коллизии, снижая производительность `HashSet`/`LinkedHashSet` до O(n).
    - **Решение**: Переопределяйте `hashCode()` и `equals()` корректно.
5. **Многопоточная модификация**:
    
    ```java
    Set<String> set = new HashSet<>();
    new Thread(() -> set.add("A")).start();
    new Thread(() -> set.add("B")).start(); // Возможна ошибка
    ```
    
    **Решение**: Используйте `ConcurrentSkipListSet` или `Collections.synchronizedSet`.
    

## 9. Лучшие практики

1. **Выбирайте правильную реализацию**:
    
    - `HashSet` для высокой производительности.
    - `LinkedHashSet` для сохранения порядка вставки.
    - `TreeSet` для сортировки или навигации.
2. **Переопределяйте `hashCode` и `equals`**:
    
    ```java
    class Key {
        String id;
        @Override public int hashCode() { return Objects.hash(id); }
        @Override public boolean equals(Object o) {
            if (!(o instanceof Key)) return false;
            return id.equals(((Key) o).id);
        }
    }
    ```
    
3. **Используйте стримы для обработки**:
    
    ```java
    Set<Integer> set = new HashSet<>(Arrays.asList(1, 2, 3));
    int sum = set.stream().mapToInt(i -> i).sum();
    ```
    
4. **Применяйте неизменяемые множества**:
    
    ```java
    Set<String> set = Set.of("A", "B");
    ```
    
5. **Обеспечивайте потокобезопасность**:
    
    ```java
    Set<String> set = Collections.synchronizedSet(new HashSet<>());
    ```
    
6. **Используйте итераторы для модификации**:
    
    ```java
    Iterator<String> iterator = set.iterator();
    while (iterator.hasNext()) {
        if (iterator.next().startsWith("A")) iterator.remove();
    }
    ```
    
7. **Избегайте `null` в `TreeSet`**:
    
    ```java
    TreeSet<String> set = new TreeSet<>();
    if (value != null) set.add(value);
    ```
    

## 10. Современные возможности Java 21+

### Pattern Matching для Set

Java 21 улучшил pattern matching, что полезно при работе с множествами:

```java
Set<Object> mixed = Set.of("string", 42, 3.14, null);

// Pattern matching в switch expressions
String result = switch (mixed.iterator().next()) {
    case String s -> "String: " + s;
    case Integer i -> "Number: " + i;
    case Double d -> "Double: " + d;
    case null -> "Null value";
    default -> "Unknown type";
};

// Pattern matching в instanceof
mixed.forEach(item -> {
    if (item instanceof String s && s.length() > 3) {
        System.out.println("Long string: " + s);
    } else if (item instanceof Integer i && i > 25) {
        System.out.println("Adult age: " + i);
    }
});
```

### Structured Concurrency с Set

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Set<Future<String>> futures = Set.of(
        scope.fork(() -> processData("data1")),
        scope.fork(() -> processData("data2")),
        scope.fork(() -> processData("data3"))
    );
    
    scope.join();
    scope.throwIfFailed();
    
    Set<String> results = futures.stream()
        .map(Future::resultNow)
        .collect(Collectors.toSet());
}
```

### Виртуальные потоки и Set

```java
Set<String> data = Set.of("a", "b", "c");
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<Set<String>> future = CompletableFuture.supplyAsync(() -> {
    return data.stream()
        .map(String::toUpperCase)
        .collect(Collectors.toSet());
}, executor);
```

## 11. Расширенные примеры использования

### Потокобезопасный Set с мониторингом

```java
public class MonitoredSet<T> {
    private final Set<T> set;
    private final AtomicLong accessCount = new AtomicLong(0);
    private final AtomicLong modificationCount = new AtomicLong(0);
    
    public MonitoredSet(Set<T> set) {
        this.set = Collections.synchronizedSet(set);
    }
    
    public boolean add(T element) {
        modificationCount.incrementAndGet();
        return set.add(element);
    }
    
    public boolean remove(T element) {
        modificationCount.incrementAndGet();
        return set.remove(element);
    }
    
    public boolean contains(T element) {
        accessCount.incrementAndGet();
        return set.contains(element);
    }
    
    public long getAccessCount() { return accessCount.get(); }
    public long getModificationCount() { return modificationCount.get(); }
    
    public Set<T> snapshot() {
        synchronized (set) {
            return new HashSet<>(set);
        }
    }
}
```

### Функциональные операции с Set

```java
public class FunctionalSetOperations {
    
    // Трансформация элементов
    public static <T, R> Set<R> transformSet(
            Set<T> set, Function<T, R> transformer) {
        return set.stream()
            .map(transformer)
            .collect(Collectors.toSet());
    }
    
    // Фильтрация с предикатом
    public static <T> Set<T> filterSet(
            Set<T> set, Predicate<T> predicate) {
        return set.stream()
            .filter(predicate)
            .collect(Collectors.toSet());
    }
    
    // Группировка элементов
    public static <T, K> Map<K, Set<T>> groupBy(
            Set<T> set, Function<T, K> keyExtractor) {
        return set.stream()
            .collect(Collectors.groupingBy(keyExtractor, Collectors.toSet()));
    }
    
    // Слияние множеств с разрешением конфликтов
    public static <T> Set<T> mergeSets(
            Set<T> set1, 
            Set<T> set2, 
            BiFunction<T, T, T> conflictResolver) {
        Set<T> result = new HashSet<>(set1);
        set2.forEach(item -> {
            if (result.contains(item)) {
                result.remove(item);
                result.add(conflictResolver.apply(item, item));
            } else {
                result.add(item);
            }
        });
        return result;
    }
    
    // Нахождение пересечения множеств
    public static <T> Set<T> intersection(Set<T> set1, Set<T> set2) {
        return set1.stream()
            .filter(set2::contains)
            .collect(Collectors.toSet());
    }
    
    // Нахождение объединения множеств
    public static <T> Set<T> union(Set<T> set1, Set<T> set2) {
        Set<T> result = new HashSet<>(set1);
        result.addAll(set2);
        return result;
    }
    
    // Нахождение разности множеств
    public static <T> Set<T> difference(Set<T> set1, Set<T> set2) {
        return set1.stream()
            .filter(item -> !set2.contains(item))
            .collect(Collectors.toSet());
    }
}
```

### Кэширование множеств

```java
public class SetCache<T> {
    private final Map<String, CachedSet<T>> cache = new ConcurrentHashMap<>();
    private final long ttlMillis;
    
    public SetCache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
    }
    
    public Set<T> get(String key) {
        CachedSet<T> cached = cache.get(key);
        if (cached != null && !cached.isExpired()) {
            return cached.getData();
        }
        cache.remove(key);
        return null;
    }
    
    public void put(String key, Set<T> data) {
        cache.put(key, new CachedSet<>(data, System.currentTimeMillis()));
    }
    
    public void cleanup() {
        cache.entrySet().removeIf(entry -> entry.getValue().isExpired());
    }
    
    private static class CachedSet<T> {
        private final Set<T> data;
        private final long timestamp;
        
        public CachedSet(Set<T> data, long timestamp) {
            this.data = new HashSet<>(data);
            this.timestamp = timestamp;
        }
        
        public Set<T> getData() { return new HashSet<>(data); }
        
        public boolean isExpired() {
            return System.currentTimeMillis() - timestamp > ttlMillis;
        }
    }
}
```

### Интеграция с внешними системами

```java
public class SetIntegration {
    
    // Загрузка уникальных данных из файла
    public static Set<String> loadUniqueFromFile(String filename) {
        try (Stream<String> lines = Files.lines(Paths.get(filename))) {
            return lines
                .filter(line -> !line.trim().isEmpty())
                .collect(Collectors.toSet());
        } catch (IOException e) {
            throw new RuntimeException("Error loading file", e);
        }
    }
    
    // Асинхронная загрузка уникальных данных
    public static CompletableFuture<Set<String>> loadUniqueDataAsync(List<String> urls) {
        List<CompletableFuture<String>> futures = urls.stream()
            .map(url -> CompletableFuture.supplyAsync(() -> fetchData(url)))
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toSet()));
    }
    
    // Экспорт в JSON
    public static String toJson(Set<String> set) {
        return set.stream()
            .map(item -> "\"" + item.replace("\"", "\\\"") + "\"")
            .collect(Collectors.joining(", ", "[", "]"));
    }
    
    // Нахождение дубликатов в списке
    public static <T> Set<T> findDuplicates(List<T> list) {
        Set<T> seen = new HashSet<>();
        Set<T> duplicates = new HashSet<>();
        
        for (T item : list) {
            if (!seen.add(item)) {
                duplicates.add(item);
            }
        }
        
        return duplicates;
    }
    
    private static String fetchData(String url) {
        // Реализация HTTP запроса
        return "data from " + url;
    }
}
```

## 12. Вопросы для собеседования

### Базовые вопросы

1. **Объясните разницу между HashSet, LinkedHashSet и TreeSet**
   - HashSet: O(1) среднее время, неупорядоченный
   - LinkedHashSet: O(1) среднее время, сохраняет порядок вставки
   - TreeSet: O(log n), отсортированный по элементам

2. **Что такое ConcurrentSkipListSet и когда его использовать?**
   - Потокобезопасная альтернатива TreeSet
   - Использует skip-list структуру
   - Подходит для многопоточных приложений с сортировкой

3. **Объясните производительность различных операций в Set**
   - add/remove/contains: O(1) среднее для HashSet, O(log n) для TreeSet
   - Итерация: O(n) для всех реализаций
   - Навигация: O(log n) для TreeSet (ceiling, floor)

### Продвинутые вопросы

4. **Как работает хеширование в HashSet?**
   - Использует HashMap внутри
   - `hashCode()` для распределения элементов
   - Коллизии разрешаются списками или деревьями

5. **Объясните механизм балансировки в TreeSet**
   - Использует красно-черное дерево
   - Автоматическая балансировка при вставке/удалении
   - Гарантирует O(log n) для всех операций

6. **Как реализовать потокобезопасный Set?**
```java
// Вариант 1: Синхронизированная обертка
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());

// Вариант 2: ConcurrentSkipListSet
Set<String> concurrentSet = new ConcurrentSkipListSet<>();

// Вариант 3: CopyOnWriteArraySet
Set<String> copyOnWriteSet = new CopyOnWriteArraySet<>();
```

### Практические вопросы

7. **Напишите код для нахождения пересечения двух множеств**
```java
public static <T> Set<T> intersection(Set<T> set1, Set<T> set2) {
    return set1.stream()
        .filter(set2::contains)
        .collect(Collectors.toSet());
}
```

8. **Реализуйте Set с ограничением размера**
```java
public class BoundedSet<T> extends LinkedHashSet<T> {
    private final int maxSize;
    
    public BoundedSet(int maxSize) {
        this.maxSize = maxSize;
    }
    
    @Override
    public boolean add(T element) {
        if (size() >= maxSize) {
            remove(iterator().next());
        }
        return super.add(element);
    }
}
```

9. **Как найти все подмножества множества?**
```java
public static <T> Set<Set<T>> getAllSubsets(Set<T> set) {
    List<T> list = new ArrayList<>(set);
    Set<Set<T>> subsets = new HashSet<>();
    
    int n = list.size();
    for (int i = 0; i < (1 << n); i++) {
        Set<T> subset = new HashSet<>();
        for (int j = 0; j < n; j++) {
            if ((i & (1 << j)) != 0) {
                subset.add(list.get(j));
            }
        }
        subsets.add(subset);
    }
    
    return subsets;
}
```

## 13. Пример: Комплексное использование `Set`

```java
import java.util.*;
import java.util.concurrent.ConcurrentSkipListSet;
import java.util.stream.Collectors;
import java.util.logging.Logger;

public class SetExample {
    private static final Logger LOGGER = Logger.getLogger(SetExample.class.getName());

    public static void main(String[] args) {
        // HashSet
        Set<String> hashSet = new HashSet<>();
        hashSet.add("apple");
        hashSet.add("banana");
        hashSet.add("apple"); // Игнорируется
        LOGGER.info("HashSet: " + hashSet); // Порядок не гарантирован
        LOGGER.info("Contains banana: " + hashSet.contains("banana")); // true
        hashSet.remove("banana");

        // LinkedHashSet
        LinkedHashSet<String> linkedHashSet = new LinkedHashSet<>();
        linkedHashSet.add("apple");
        linkedHashSet.add("banana");
        linkedHashSet.add("cherry");
        LOGGER.info("LinkedHashSet: " + linkedHashSet); // Порядок вставки

        // TreeSet
        TreeSet<Integer> treeSet = new TreeSet<>();
        treeSet.add(30);
        treeSet.add(10);
        treeSet.add(20);
        LOGGER.info("TreeSet: " + treeSet); // [10, 20, 30]
        LOGGER.info("First: " + treeSet.first()); // 10
        LOGGER.info("Ceiling(15): " + treeSet.ceiling(15)); // 20

        // ConcurrentSkipListSet
        Set<String> concurrentSet = new ConcurrentSkipListSet<>();
        new Thread(() -> concurrentSet.add("A")).start();
        new Thread(() -> LOGGER.info("ConcurrentSet: " + concurrentSet)).start();

        // Стримы
        List<String> filtered = hashSet.stream()
                                      .filter(s -> s.startsWith("a"))
                                      .map(String::toUpperCase)
                                      .collect(Collectors.toList());
        LOGGER.info("Filtered: " + filtered);

        // Неизменяемое множество
        Set<String> immutableSet = Set.of("X", "Y");
        LOGGER.info("Immutable Set: " + immutableSet);

        // Безопасное удаление
        Iterator<String> iterator = hashSet.iterator();
        while (iterator.hasNext()) {
            if (iterator.next().startsWith("a")) iterator.remove();
        }
        LOGGER.info("After removal: " + hashSet);
    }
}
```

**Предполагаемый вывод** (порядок для `parallelStream` и многопоточных операций может варьироваться):

```
INFO: HashSet: [apple, banana]
INFO: Contains banana: true
INFO: LinkedHashSet: [apple, banana, cherry]
INFO: TreeSet: [10, 20, 30]
INFO: First: 10
INFO: Ceiling(15): 20
INFO: ConcurrentSet: [A]
INFO: Filtered: [APPLE]
INFO: Immutable Set: [X, Y]
INFO: After removal: []
```