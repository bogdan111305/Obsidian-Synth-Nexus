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
    

## 10. Пример: Комплексное использование `Set`

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