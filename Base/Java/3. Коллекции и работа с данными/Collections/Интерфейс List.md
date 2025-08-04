## 1. Обзор интерфейса `List<E>`

`List<E>` — это интерфейс, определяющий упорядоченную коллекцию, где:

- Элементы хранятся в порядке добавления (или изменения).
- Допускаются дубликаты.
- Доступ к элементам осуществляется по индексу (0-based).
- Поддерживаются операции добавления, удаления, поиска, итерации и модификации.

**Основные методы**:

- `add(E e)`: Добавляет элемент в конец списка.
- `add(int index, E element)`: Добавляет элемент по индексу.
- `get(int index)`: Возвращает элемент по индексу.
- `remove(int index)`: Удаляет элемент по индексу.
- `indexOf(Object o)`: Возвращает индекс первого вхождения элемента.
- `size()`: Возвращает количество элементов.

**Реализации**:

1. `ArrayList<E>`: Динамический массив.
2. `LinkedList<E>`: Двусвязный список, также реализует `Deque` и `Queue`.
3. `Vector<E>`: Синхронизированный динамический массив (устаревший).
4. `Stack<E>`: Стек, основанный на `Vector` (устаревший).

## 2. Реализации интерфейса `List`

### 2.1. `ArrayList<E>`

`ArrayList` — это основная реализация `List`, использующая динамический массив для хранения элементов. Подходит для большинства сценариев, где важен быстрый доступ по индексу.

#### Внутреннее устройство

Внутри `ArrayList` использует массив `Object[]`:

```java
transient Object[] elementData; // Массив для хранения элементов
private int size;              // Количество элементов
private static final int DEFAULT_CAPACITY = 10; // Начальная емкость
```

- `elementData`: Хранит элементы, может быть больше, чем `size`.
- `size`: Текущее количество элементов.
- Массив создается с начальной емкостью 10 (если не указана другая).

#### Механизм расширения

Когда массив заполняется, `ArrayList` увеличивает его размер на ~50%:

```java
int newCapacity = oldCapacity + (oldCapacity >> 1); // Увеличение на 50%
elementData = Arrays.copyOf(elementData, newCapacity);
```

**Пример**:

- Начальная емкость: 10 → 15 → 22 → 33 → и т.д.
- Копирование массива — затратная операция (O(n)).

#### Основные операции и Big-O

|Операция|Метод|Сложность|
|---|---|---|
|Получение по индексу|`get(int index)`|O(1)|
|Установка по индексу|`set(int index, E element)`|O(1)|
|Добавление в конец|`add(E e)`|O(1) (амортизированное)|
|Добавление по индексу|`add(int index, E element)`|O(n)|
|Удаление по индексу|`remove(int index)`|O(n)|
|Поиск по значению|`indexOf(Object o)`|O(n)|
|Размер|`size()`|O(1)|

- **O(1) для `get`/`set`**: Прямой доступ к массиву.
- **O(n) для добавления/удаления в середине**: Требуется сдвиг элементов.
- **O(1) амортизированное для `add` в конец**: Расширение массива происходит редко.

#### Пример

```java
import java.util.ArrayList;
import java.util.List;

List<String> list = new ArrayList<>();
list.add("A");          // O(1)
list.add(0, "B");       // O(n)
System.out.println(list.get(1)); // O(1), вывод: A
list.remove(0);         // O(n)
list.remove(list.size() - 1); // O(1)
```

### 2.2. `LinkedList<E>`

`LinkedList` — это двусвязный список, реализующий `List`, `Deque` и `Queue`. Подходит для сценариев с частыми добавлениями/удалениями в начале или конце списка.

#### Внутреннее устройство

`LinkedList` состоит из узлов (`Node`):

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
transient Node<E> first;
transient Node<E> last;
transient int size;
```

- Каждый узел хранит элемент и ссылки на предыдущий/следующий узлы.
- `first` и `last` указывают на начало и конец списка.

#### Механизм расширения

`LinkedList` не использует массив, поэтому расширения не требуется. Добавление элемента — это создание нового узла и обновление ссылок (O(1) для концов).

#### Основные операции и Big-O

|Операция|Метод|Сложность|
|---|---|---|
|Получение по индексу|`get(int index)`|O(n)|
|Установка по индексу|`set(int index, E element)`|O(n)|
|Добавление в конец|`add(E e)`|O(1)|
|Добавление в начало|`addFirst(E e)`|O(1)|
|Добавление по индексу|`add(int index, E element)`|O(n)|
|Удаление с начала|`removeFirst()`|O(1)|
|Удаление с конца|`removeLast()`|O(1)|
|Удаление по индексу|`remove(int index)`|O(n)|
|Поиск по значению|`indexOf(Object o)`|O(n)|
|Размер|`size()`|O(1)|

- **O(1) для операций с концами**: Обновление ссылок.
- **O(n) для доступа по индексу**: Требуется проход по списку.

#### Пример

```java
import java.util.LinkedList;

LinkedList<String> list = new LinkedList<>();
list.add("A");          // O(1)
list.addFirst("B");     // O(1)
list.add(1, "C");       // O(n)
System.out.println(list.get(1)); // O(n), вывод: C
list.removeFirst();     // O(1)
list.removeLast();      // O(1)
```

### 2.3. `Vector<E>`

`Vector` — это устаревшая, синхронизированная версия `ArrayList`, использующая динамический массив.

#### Внутреннее устройство

Аналогично `ArrayList`, но методы синхронизированы:

```java
synchronized Object[] elementData;
int size;
```

#### Механизм расширения

Увеличивает массив на фиксированную величину (по умолчанию удваивает):

```java
int newCapacity = oldCapacity + capacityIncrement; // или oldCapacity * 2
```

#### Основные операции и Big-O

Аналогичны `ArrayList`, но с дополнительными накладными расходами из-за синхронизации:

- `get`/`set`: O(1)
- `add` в конец: O(1) амортизированное
- `add`/`remove` по индексу: O(n)
- **Синхронизация**: Добавляет затраты на блокировку.

#### Пример

```java
import java.util.Vector;

Vector<String> vector = new Vector<>();
vector.add("A");        // O(1), синхронизировано
vector.add(0, "B");     // O(n), синхронизировано
System.out.println(vector.get(1)); // O(1), вывод: A
```

**Примечание**: `Vector` устарел, предпочтительнее использовать `Collections.synchronizedList(new ArrayList<>())` или `CopyOnWriteArrayList` для многопоточных приложений.

### 2.4. `Stack<E>`

`Stack` — это устаревший класс, расширяющий `Vector`, реализующий стек (LIFO).

#### Внутреннее устройство

Наследует структуру `Vector` (массив с синхронизацией).

#### Основные операции

- `push(E e)`: Добавляет элемент на вершину (O(1)).
- `pop()`: Удаляет и возвращает верхний элемент (O(1)).
- `peek()`: Возвращает верхний элемент без удаления (O(1)).

#### Пример

```java
import java.util.Stack;

Stack<String> stack = new Stack<>();
stack.push("A");        // O(1)
stack.push("B");        // O(1)
System.out.println(stack.pop()); // O(1), вывод: B
```

**Примечание**: Вместо `Stack` используйте `ArrayDeque` для стека, так как он быстрее и не синхронизирован по умолчанию.

## 3. Сравнение реализаций

|Характеристика|`ArrayList`|`LinkedList`|`Vector`|`Stack`|
|---|---|---|---|---|
|**Структура**|Динамический массив|Двусвязный список|Синхр. массив|Синхр. массив (LIFO)|
|**Доступ по индексу**|O(1)|O(n)|O(1)|O(1)|
|**Добавление/удаление (конец)**|O(1) амортизированное|O(1)|O(1) амортизированное|O(1)|
|**Добавление/удаление (середина)**|O(n)|O(n)|O(n)|N/A|
|**Потокобезопасность**|Нет|Нет|Да (устаревшая)|Да (устаревшая)|
|**Память**|Меньше (массив)|Больше (узлы)|Меньше (массив)|Меньше (массив)|
|**Использование**|Общее, быстрый доступ|Очереди, частые вставки|Многопоточность (устарел)|Стек (устарел)|

## 4. Внутренняя реализация в JVM

- **`ArrayList`**:
    
    - Хранит элементы в массиве `Object[]` в куче (heap).
    - Доступ по индексу (`get`) — простое обращение к массиву (`elementData[index]`).
    - Расширение массива использует `Arrays.copyOf`, вызывая нативный `System.arraycopy`.
    - JIT-компилятор оптимизирует доступ и итерацию.
- **`LinkedList`**:
    
    - Каждый узел (`Node`) — объект с тремя полями (`item`, `next`, `prev`), что увеличивает расход памяти.
    - Доступ по индексу требует прохода по списку, что неэффективно.
    - Операции с концами (`addFirst`, `removeLast`) — обновление ссылок (O(1)).
- **`Vector`**:
    
    - Аналогичен `ArrayList`, но использует `synchronized` для методов, что добавляет накладные расходы на блокировку.
    - Расширение массива менее гибкое (удваивание по умолчанию).
- **`Stack`**:
    
    - Наследует `Vector`, операции `push`/`pop` используют методы `Vector`.

**Байт-код**:  
Операции `get`/`set` компилируются в прямой доступ к массиву (`aaload`, `aastore`) для `ArrayList` и `Vector`, а для `LinkedList` — в цикл по узлам. Итерация через `for-each` использует `Iterator`.

## 5. Производительность

- **`ArrayList`**:
    
    - **Преимущества**: Быстрый доступ по индексу (O(1)), эффективное использование памяти.
    - **Недостатки**: Медленные вставки/удаления в середине (O(n)) из-за сдвига.
    - Расширение массива (O(n)) редко, поэтому добавление в конец амортизировано до O(1).
- **`LinkedList`**:
    
    - **Преимущества**: Быстрые операции с концами (O(1)).
    - **Недостатки**: Медленный доступ по индексу и поиск (O(n)), больший расход памяти из-за узлов.
- **`Vector`**:
    
    - **Недостатки**: Синхронизация замедляет операции, устаревший дизайн.
    - **Сравнение**: Медленнее `ArrayList` из-за блокировок.
- **`Stack`**:
    
    - **Недостатки**: Унаследованные ограничения `Vector`, неэффективен по сравнению с `ArrayDeque`.

**Рекомендация**:

- Используйте `ArrayList` для большинства случаев.
- Используйте `LinkedList` для очередей или частых операций с концами.
- Избегайте `Vector` и `Stack`, используйте `Collections.synchronizedList` или `ArrayDeque`.

## 6. Многопоточность

`ArrayList` и `LinkedList` **не потокобезопасны**. Для многопоточных приложений:

- **Синхронизированные обертки**:
    
    ```java
    List<String> syncList = Collections.synchronizedList(new ArrayList<>());
    ```
    
- **CopyOnWriteArrayList** (Java 5+):
    
    - Создает копию массива при каждой модификации, подходит для частого чтения.
    
    ```java
    import java.util.concurrent.CopyOnWriteArrayList;
    
    List<String> list = new CopyOnWriteArrayList<>();
    new Thread(() -> list.add("A")).start();
    new Thread(() -> System.out.println(list)).start();
    ```
    
- **`Vector` и `Stack`**:
    
    - Потокобезопасны, но устарели из-за низкой производительности (глобальная блокировка).
    - Предпочтительнее `CopyOnWriteArrayList` или `Collections.synchronizedList`.

**Параллельные стримы**:

```java
List<Integer> numbers = Arrays.asList(1, 2, 3);
numbers.parallelStream().forEach(System.out::println);
```

**Ограничение**: Избегайте модификации списка в `parallelStream().forEach()` во избежание гонок данных.

## 7. Современные возможности (Java 8+)

### 7.1. Стримы

`Stream API` упрощает обработку списков.

**Пример**:

```java
List<String> list = Arrays.asList("apple", "banana", "cherry");
list.stream()
    .filter(s -> s.startsWith("a"))
    .map(String::toUpperCase)
    .forEach(System.out::println); // APPLE
```

### 7.2. Неизменяемые списки (Java 9+)

Метод `List.of()` создает неизменяемый список:

```java
List<String> immutableList = List.of("A", "B", "C");
immutableList.add("D"); // UnsupportedOperationException
```

### 7.3. Сортировка и преобразование

Методы `Collections.sort()` и стримы для сортировки:

```java
List<Integer> list = new ArrayList<>(Arrays.asList(3, 1, 2));
Collections.sort(list); // [1, 2, 3]
```

## 8. Подводные камни

1. **Модификация при итерации**:
    
    ```java
    List<String> list = new ArrayList<>(Arrays.asList("A", "B"));
    for (String s : list) {
        list.remove(s); // ConcurrentModificationException
    }
    ```
    
    **Решение**: Используйте `Iterator`:
    
    ```java
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        iterator.next();
        iterator.remove();
    }
    ```
    
2. **Неэффективный выбор реализации**:
    
    - Использование `LinkedList` для частого доступа по индексу (O(n)).
    - **Решение**: Используйте `ArrayList` для доступа по индексу.
3. **Переполнение массива в `ArrayList`**:
    
    - При большом числе элементов расширение массива может быть затратным.
    - **Решение**: Задавайте начальную емкость:
    
    ```java
    List<String> list = new ArrayList<>(1000);
    ```
    
4. **Null в списках**:
    
    - Все реализации допускают `null`, но это может вызвать `NullPointerException` при обработке.
    - **Решение**: Проверяйте элементы на `null`.
5. **Многопоточная модификация**:
    
    ```java
    List<String> list = new ArrayList<>();
    new Thread(() -> list.add("A")).start();
    new Thread(() -> list.add("B")).start(); // Возможна ошибка
    ```
    
    **Решение**: Используйте `CopyOnWriteArrayList` или синхронизацию.
    

## 9. Лучшие практики

1. **Выбирайте правильную реализацию**:
    
    - `ArrayList` для быстрого доступа по индексу и общего использования.
    - `LinkedList` для очередей или частых вставок/удалений в концах.
    - Избегайте `Vector` и `Stack`, используйте `ArrayDeque` для стека.
2. **Задавайте начальную емкость для `ArrayList`**:
    
    ```java
    List<String> list = new ArrayList<>(1000);
    ```
    
3. **Используйте стримы для сложных операций**:
    
    ```java
    List<Integer> numbers = Arrays.asList(1, 2, 3);
    int sum = numbers.stream().mapToInt(i -> i).sum();
    ```
    
4. **Применяйте неизменяемые списки**:
    
    ```java
    List<String> list = List.of("A", "B");
    ```
    
5. **Обеспечивайте потокобезопасность**:
    
    ```java
    List<String> syncList = Collections.synchronizedList(new ArrayList<>());
    ```
    
6. **Используйте итераторы для модификации**:
    
    ```java
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        if (iterator.next().startsWith("A")) iterator.remove();
    }
    ```
    
7. **Проверяйте на `null`**:
    
    ```java
    if (list.get(index) != null) { /* обработка */ }
    ```
    

## 10. Современные возможности Java 21+

### Pattern Matching для List

Java 21 улучшил pattern matching, что полезно при работе со списками:

```java
List<Object> mixed = List.of("string", 42, 3.14, null);

// Pattern matching в switch expressions
String result = switch (mixed.get(0)) {
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

### Structured Concurrency с List

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    List<Future<String>> futures = List.of(
        scope.fork(() -> processData("data1")),
        scope.fork(() -> processData("data2")),
        scope.fork(() -> processData("data3"))
    );
    
    scope.join();
    scope.throwIfFailed();
    
    List<String> results = futures.stream()
        .map(Future::resultNow)
        .collect(Collectors.toList());
}
```

### Виртуальные потоки и List

```java
List<String> data = List.of("a", "b", "c");
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<List<String>> future = CompletableFuture.supplyAsync(() -> {
    return data.stream()
        .map(String::toUpperCase)
        .collect(Collectors.toList());
}, executor);
```

## 11. Расширенные примеры использования

### Потокобезопасный список с мониторингом

```java
public class MonitoredList<T> {
    private final List<T> list;
    private final AtomicLong accessCount = new AtomicLong(0);
    private final AtomicLong modificationCount = new AtomicLong(0);
    
    public MonitoredList(List<T> list) {
        this.list = Collections.synchronizedList(list);
    }
    
    public T get(int index) {
        accessCount.incrementAndGet();
        return list.get(index);
    }
    
    public void add(T element) {
        modificationCount.incrementAndGet();
        list.add(element);
    }
    
    public void remove(int index) {
        modificationCount.incrementAndGet();
        list.remove(index);
    }
    
    public long getAccessCount() { return accessCount.get(); }
    public long getModificationCount() { return modificationCount.get(); }
    
    public List<T> snapshot() {
        synchronized (list) {
            return new ArrayList<>(list);
        }
    }
}
```

### Функциональные операции с List

```java
public class FunctionalListOperations {
    
    // Трансформация с индексами
    public static <T, R> List<R> transformWithIndex(
            List<T> list, BiFunction<T, Integer, R> transformer) {
        return IntStream.range(0, list.size())
            .mapToObj(i -> transformer.apply(list.get(i), i))
            .collect(Collectors.toList());
    }
    
    // Фильтрация с предикатом
    public static <T> List<T> filterList(
            List<T> list, Predicate<T> predicate) {
        return list.stream()
            .filter(predicate)
            .collect(Collectors.toList());
    }
    
    // Группировка элементов
    public static <T, K> Map<K, List<T>> groupBy(
            List<T> list, Function<T, K> keyExtractor) {
        return list.stream()
            .collect(Collectors.groupingBy(keyExtractor));
    }
    
    // Слияние списков с разрешением конфликтов
    public static <T> List<T> mergeLists(
            List<T> list1, 
            List<T> list2, 
            BiFunction<T, T, T> conflictResolver) {
        Map<Integer, T> merged = new HashMap<>();
        
        // Добавляем элементы из первого списка
        for (int i = 0; i < list1.size(); i++) {
            merged.put(i, list1.get(i));
        }
        
        // Обрабатываем элементы из второго списка
        for (int i = 0; i < list2.size(); i++) {
            T existing = merged.get(i);
            if (existing != null) {
                merged.put(i, conflictResolver.apply(existing, list2.get(i)));
            } else {
                merged.put(i, list2.get(i));
            }
        }
        
        return merged.entrySet().stream()
            .sorted(Map.Entry.comparingByKey())
            .map(Map.Entry::getValue)
            .collect(Collectors.toList());
    }
}
```

### Кэширование списков

```java
public class ListCache<T> {
    private final Map<String, CachedList<T>> cache = new ConcurrentHashMap<>();
    private final long ttlMillis;
    
    public ListCache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
    }
    
    public List<T> get(String key) {
        CachedList<T> cached = cache.get(key);
        if (cached != null && !cached.isExpired()) {
            return cached.getData();
        }
        cache.remove(key);
        return null;
    }
    
    public void put(String key, List<T> data) {
        cache.put(key, new CachedList<>(data, System.currentTimeMillis()));
    }
    
    public void cleanup() {
        cache.entrySet().removeIf(entry -> entry.getValue().isExpired());
    }
    
    private static class CachedList<T> {
        private final List<T> data;
        private final long timestamp;
        
        public CachedList(List<T> data, long timestamp) {
            this.data = new ArrayList<>(data);
            this.timestamp = timestamp;
        }
        
        public List<T> getData() { return new ArrayList<>(data); }
        
        public boolean isExpired() {
            return System.currentTimeMillis() - timestamp > ttlMillis;
        }
    }
}
```

### Интеграция с внешними системами

```java
public class ListIntegration {
    
    // Загрузка данных из файла
    public static List<String> loadFromFile(String filename) {
        try (Stream<String> lines = Files.lines(Paths.get(filename))) {
            return lines
                .filter(line -> !line.trim().isEmpty())
                .collect(Collectors.toList());
        } catch (IOException e) {
            throw new RuntimeException("Error loading file", e);
        }
    }
    
    // Асинхронная загрузка данных
    public static CompletableFuture<List<String>> loadDataAsync(List<String> urls) {
        List<CompletableFuture<String>> futures = urls.stream()
            .map(url -> CompletableFuture.supplyAsync(() -> fetchData(url)))
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }
    
    // Экспорт в CSV
    public static String toCsv(List<List<String>> data) {
        return data.stream()
            .map(row -> row.stream()
                .map(cell -> "\"" + cell.replace("\"", "\"\"") + "\"")
                .collect(Collectors.joining(",")))
            .collect(Collectors.joining("\n"));
    }
    
    private static String fetchData(String url) {
        // Реализация HTTP запроса
        return "data from " + url;
    }
}
```

## 12. Вопросы для собеседования

### Базовые вопросы

1. **Объясните разницу между ArrayList и LinkedList**
   - ArrayList: O(1) доступ по индексу, O(n) вставка/удаление в середине
   - LinkedList: O(n) доступ по индексу, O(1) вставка/удаление в концах
   - ArrayList использует массив, LinkedList - двусвязный список

2. **Что такое Vector и почему он устарел?**
   - Синхронизированная версия ArrayList
   - Медленнее из-за блокировок
   - Лучше использовать Collections.synchronizedList или CopyOnWriteArrayList

3. **Объясните производительность различных операций в List**
   - get/set: O(1) для ArrayList, O(n) для LinkedList
   - add в конец: O(1) амортизированное для ArrayList, O(1) для LinkedList
   - add в середину: O(n) для обеих реализаций

### Продвинутые вопросы

4. **Как работает CopyOnWriteArrayList?**
   - Создает копию массива при каждой модификации
   - Потокобезопасен для чтения
   - Подходит для частого чтения, редкой записи

5. **Объясните механизм расширения ArrayList**
   - Начальная емкость 10
   - Увеличение на 50% при переполнении
   - Копирование массива O(n)

6. **Как реализовать потокобезопасный список?**
```java
// Вариант 1: Синхронизированная обертка
List<String> syncList = Collections.synchronizedList(new ArrayList<>());

// Вариант 2: CopyOnWriteArrayList
List<String> copyOnWriteList = new CopyOnWriteArrayList<>();

// Вариант 3: Ручная синхронизация
List<String> manualSyncList = new ArrayList<>();
synchronized (manualSyncList) {
    manualSyncList.add("item");
}
```

### Практические вопросы

7. **Напишите код для удаления дубликатов из списка**
```java
public static <T> List<T> removeDuplicates(List<T> list) {
    return list.stream()
        .distinct()
        .collect(Collectors.toList());
}
```

8. **Реализуйте обращение списка на месте**
```java
public static <T> void reverseList(List<T> list) {
    for (int i = 0; i < list.size() / 2; i++) {
        T temp = list.get(i);
        list.set(i, list.get(list.size() - 1 - i));
        list.set(list.size() - 1 - i, temp);
    }
}
```

9. **Как найти k-й элемент с конца в LinkedList за один проход?**
```java
public static <T> T findKthFromEnd(LinkedList<T> list, int k) {
    if (k <= 0 || k > list.size()) return null;
    
    Iterator<T> fast = list.iterator();
    Iterator<T> slow = list.iterator();
    
    // Продвигаем быстрый итератор на k позиций
    for (int i = 0; i < k; i++) {
        if (!fast.hasNext()) return null;
        fast.next();
    }
    
    // Двигаем оба итератора до конца
    while (fast.hasNext()) {
        fast.next();
        slow.next();
    }
    
    return slow.next();
}
```

## 13. Пример: Комплексное использование `List`

```java
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.stream.Collectors;
import java.util.logging.Logger;

public class ListExample {
    private static final Logger LOGGER = Logger.getLogger(ListExample.class.getName());

    public static void main(String[] args) {
        // ArrayList
        List<String> arrayList = new ArrayList<>(100);
        arrayList.add("apple");
        arrayList.add(0, "banana");
        LOGGER.info("ArrayList: " + arrayList);
        LOGGER.info("Элемент по индексу 1: " + arrayList.get(1)); // O(1)
        arrayList.remove(0); // O(n)
        LOGGER.info("После удаления: " + arrayList);

        // LinkedList
        LinkedList<String> linkedList = new LinkedList<>();
        linkedList.add("cherry");
        linkedList.addFirst("date");
        LOGGER.info("LinkedList: " + linkedList);
        linkedList.removeLast(); // O(1)
        LOGGER.info("После удаления: " + linkedList);

        // CopyOnWriteArrayList для многопоточности
        List<String> concurrentList = new CopyOnWriteArrayList<>(Arrays.asList("A", "B"));
        new Thread(() -> concurrentList.add("C")).start();
        new Thread(() -> LOGGER.info("ConcurrentList: " + concurrentList)).start();

        // Стримы
        List<String> filtered = arrayList.stream()
                                        .filter(s -> s.startsWith("a"))
                                        .map(String::toUpperCase)
                                        .collect(Collectors.toList());
        LOGGER.info("Фильтрация и преобразование: " + filtered);

        // Неизменяемый список
        List<String> immutableList = List.of("X", "Y");
        LOGGER.info("Неизменяемый List: " + immutableList);

        // Итератор для безопасного удаления
        Iterator<String> iterator = arrayList.iterator();
        while (iterator.hasNext()) {
            if (iterator.next().startsWith("a")) iterator.remove();
        }
        LOGGER.info("После удаления через итератор: " + arrayList);
    }
}
```

**Предполагаемый вывод**:

```
INFO: ArrayList: [banana, apple]
INFO: Элемент по индексу 1: apple
INFO: После удаления: [apple]
INFO: LinkedList: [date, cherry]
INFO: После удаления: [date]
INFO: ConcurrentList: [A, B, C]
INFO: Фильтрация и преобразование: [APPLE]
INFO: Неизменяемый List: [X, Y]
INFO: После удаления через итератор: []
```