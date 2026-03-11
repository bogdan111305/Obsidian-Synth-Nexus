---
title: "Java — Интерфейс List (ArrayList, LinkedList)"
tags: [java, collections, list, arraylist, linkedlist]
updated: 2026-03-04
---

# Интерфейс List в Java

> [!QUOTE] Суть
> **List** — упорядоченная коллекция с дубликатами. `ArrayList`: O(1) get/set, O(n) insert/remove посередине, рост x1.5. `LinkedList`: O(1) add/remove по краям, O(n) get(i). В большинстве случаев `ArrayList` быстрее за счёт cache locality.

> [!WARNING] Ловушка: remove(int) vs remove(Object) в List
> `list.remove(1)` удаляет по индексу, `list.remove(Integer.valueOf(1))` — по значению. С autoboxing можно получить неожиданное поведение.

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

## Связанные темы

- [[Общая иерархия коллекций]] — полная иерархия коллекций
- [[Интерфейс Set]] — уникальные элементы без порядка
- [[Производительность коллекций]] — Big-O для List операций
- [[Java Stream API]] — обработка List через stream