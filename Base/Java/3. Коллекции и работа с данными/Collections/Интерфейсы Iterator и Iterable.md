## 1. Обзор интерфейсов `Iterator` и `Iterable`

### 1.1. Что такое `Iterable`?

Интерфейс `Iterable<T>` определяет объект, который может быть перебираем, то есть предоставляет возможность последовательно получать его элементы. Он используется в конструкциях `for-each` и интеграции со стримами.

**Основные методы**:

- `Iterator<T> iterator()`: Возвращает итератор для перебора элементов.
- `default void forEach(Consumer<? super T> action)` (Java 8+): Выполняет действие для каждого элемента.
- `default Spliterator<T> spliterator()` (Java 8+): Возвращает сплитератор для параллельной обработки (используется в стримах).

**Назначение**:

- Позволяет коллекциям (например, `List`, `Set`, `Map`) поддерживать цикл `for-each`.
- Обеспечивает совместимость со стримами и другими API.

**Пример**:

```java
List<String> list = Arrays.asList("A", "B", "C");
for (String s : list) { // Использует Iterable
    System.out.println(s);
}
```

### 1.2. Что такое `Iterator`?

Интерфейс `Iterator<T>` предоставляет методы для последовательного доступа к элементам коллекции и, при необходимости, их удаления во время перебора.

**Основные методы**:

- `boolean hasNext()`: Проверяет, есть ли следующий элемент.
- `T next()`: Возвращает следующий элемент (выбрасывает `NoSuchElementException`, если элементов нет).
- `default void remove()`: Удаляет последний возвращенный элемент (опционально, может выбрасывать `UnsupportedOperationException`).
- `default void forEachRemaining(Consumer<? super T> action)` (Java 8+): Выполняет действие для оставшихся элементов.

**Назначение**:

- Обеспечивает низкоуровневый контроль над перебором.
- Позволяет безопасно удалять элементы во время итерации.

**Пример**:

```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    String s = iterator.next();
    if (s.equals("B")) {
        iterator.remove(); // Безопасное удаление
    }
}
System.out.println(list); // [A, C]
```

### 1.3. Зачем нужны?

- **`Iterable`**:
    - Упрощает перебор коллекций через `for-each`.
    - Обеспечивает совместимость со стримами и функциональным программированием.
    - Делает код читаемым и декларативным.
- **`Iterator`**:
    - Предоставляет гибкость для ручного перебора.
    - Поддерживает безопасное удаление элементов.
    - Используется в коллекциях и других структурах данных.

**Иерархия**:

```
java.lang.Iterable<T>
└── java.util.Iterator<T>
```

Все стандартные коллекции (`List`, `Set`, `Map.entrySet()`) реализуют `Iterable`.

## 2. Как работают `Iterator` и `Iterable`

### 2.1. Механизм работы `Iterable`

- Реализация `Iterable` обязывает класс предоставить метод `iterator()`, возвращающий объект `Iterator`.
- Цикл `for-each` автоматически вызывает `iterator()`:
    
    ```java
    for (T element : iterable) {
        // Код
    }
    ```
    
    Эквивалентно:
    
    ```java
    Iterator<T> iterator = iterable.iterator();
    while (iterator.hasNext()) {
        T element = iterator.next();
        // Код
    }
    ```
    
- `forEach` и `spliterator` (Java 8+) интегрируют `Iterable` со стримами и параллельной обработкой.

### 2.2. Механизм работы `Iterator`

- `Iterator` отслеживает текущую позицию в коллекции (обычно через индекс или указатель).
- **Состояние**:
    - `hasNext()`: Проверяет, не достигнут ли конец.
    - `next()`: Возвращает элемент и сдвигает позицию.
    - `remove()`: Удаляет последний элемент, возвращенный `next()`, обновляя коллекцию.
- Реализация `Iterator` зависит от структуры данных:
    - `ArrayList`: Использует индекс массива.
    - `LinkedList`: Указатель на узел.
    - `HashMap`: Перебор корзин и узлов (списков/деревьев).

**Пример реализации (упрощенно)**:

```java
class MyList<T> implements Iterable<T> {
    private T[] items;
    private int size;

    @Override
    public Iterator<T> iterator() {
        return new MyIterator();
    }

    private class MyIterator implements Iterator<T> {
        private int current = 0;

        @Override
        public boolean hasNext() {
            return current < size;
        }

        @Override
        public T next() {
            if (!hasNext()) throw new NoSuchElementException();
            return items[current++];
        }

        @Override
        public void remove() {
            throw new UnsupportedOperationException();
        }
    }
}
```

### 2.3. Взаимодействие

- `Iterable` предоставляет `Iterator` через `iterator()`.
- Коллекции реализуют `Iterable`, а их итераторы — `Iterator`.
- Пример: `ArrayList` возвращает `Itr` (внутренний класс), реализующий `Iterator`.

## 3. Внутренняя реализация в JVM

- **`Iterable`**:
    
    - Интерфейс, не содержит состояния.
    - Байт-код: Вызов `iterator()` через `invokeinterface`.
    - `forEach` и `spliterator` используют функциональные интерфейсы (`Consumer`, `Spliterator`).
- **`Iterator`**:
    
    - Реализация зависит от коллекции:
        - `ArrayList.Itr`: Хранит индекс (`cursor`), проверяет модификации (`expectedModCount`).
        - `HashMap.HashIterator`: Перебирает корзины и узлы (списки/деревья).
    - Байт-код: `hasNext` и `next` используют `invokevirtual`.
    - `remove`: Проверяет `modCount` для защиты от `ConcurrentModificationException`.
- **Память**:
    
    - `Iterator` хранит минимальное состояние (например, индекс или указатель).
    - Накладные расходы низкие, зависят от структуры данных.

**Пример (ArrayList.Itr, упрощенно)**:

```java
class ArrayList<T> implements Iterable<T> {
    transient Object[] elementData;
    int size;
    int modCount;

    public Iterator<T> iterator() {
        return new Itr();
    }

    private class Itr implements Iterator<T> {
        int cursor;
        int lastRet = -1;
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor != size;
        }

        public T next() {
            if (cursor >= size) throw new NoSuchElementException();
            if (modCount != expectedModCount) throw new ConcurrentModificationException();
            return (T) elementData[lastRet = cursor++];
        }

        public void remove() {
            if (lastRet < 0) throw new IllegalStateException();
            if (modCount != expectedModCount) throw new ConcurrentModificationException();
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        }
    }
}
```

## 4. Производительность

- **Операции**:
    - `hasNext`: O(1) (проверка индекса/указателя).
    - `next`: O(1) для `ArrayList`, `LinkedList`; O(1) среднее, O(log n) худшее для `HashMap` (при treeification).
    - `remove`: O(1) для `ArrayList`, O(1) для `LinkedList` (с указателем), O(log n) для деревьев в `HashMap`.
- **Итерация**:
    - `ArrayList`: O(n), последовательный доступ к массиву.
    - `LinkedList`: O(n), перебор узлов.
    - `HashMap`: O(n), перебор корзин и узлов (списков/деревьев).
    - `TreeMap`: O(n), in-order обход дерева.
- **Подводные камни**:
    - Частое использование `remove` в `LinkedList` может быть медленным из-за поиска узлов.
    - Итерация по `HashMap` с большим числом коллизий может деградировать до O(n log n) при treeification.

**Рекомендация**: Используйте `for-each` для простоты, но `Iterator` для удаления элементов.

## 5. Многопоточность

`Iterator` и `Iterable` не являются потокобезопасными в стандартных коллекциях (`ArrayList`, `HashMap`, и т.д.). Проблемы:

- **ConcurrentModificationException**: Возникает при модификации коллекции во время итерации (кроме `remove` через итератор).
- **Решения**:
    - Используйте `ConcurrentHashMap` или `CopyOnWriteArrayList`, где итераторы не выбрасывают исключений.
    - Синхронизируйте доступ:
        
        ```java
        List<String> list = Collections.synchronizedList(new ArrayList<>());
        synchronized (list) {
            Iterator<String> iterator = list.iterator();
            while (iterator.hasNext()) {
                System.out.println(iterator.next());
            }
        }
        ```
        
    - Используйте стримы для параллельной обработки:
        
        ```java
        List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
        list.parallelStream().forEach(System.out::println);
        ```
        

**Пример с ConcurrentHashMap**:

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("A", 1);
Iterator<Map.Entry<String, Integer>> iterator = map.entrySet().iterator();
new Thread(() -> map.put("B", 2)).start(); // Безопасно
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

## 6. Современные возможности (Java 8+)

### 6.1. For-Each и `Iterable`

Цикл `for-each` упрощает перебор:

```java
List<String> list = Arrays.asList("A", "B", "C");
for (String s : list) {
    System.out.println(s);
}
```

### 6.2. Метод `forEach`

Метод `Iterable.forEach` интегрируется с лямбда-выражениями:

```java
List<String> list = Arrays.asList("A", "B", "C");
list.forEach(System.out::println);
```

### 6.3. Стримы

`Iterable` поддерживает стримы через `stream()` и `parallelStream()`:

```java
List<String> list = Arrays.asList("A", "B", "C");
list.stream().filter(s -> s.equals("A")).forEach(System.out::println);
```

### 6.4. Метод `forEachRemaining`

Метод `Iterator.forEachRemaining` упрощает перебор оставшихся элементов:

```java
Iterator<String> iterator = Arrays.asList("A", "B", "C").iterator();
iterator.next(); // Пропускаем "A"
iterator.forEachRemaining(System.out::println); // B, C
```

### 6.5. Spliterator

`Iterable.spliterator()` используется для параллельной обработки:

```java
List<String> list = Arrays.asList("A", "B", "C");
Spliterator<String> spliterator = list.spliterator();
spliterator.forEachRemaining(System.out::println);
```

## 7. Подводные камни

1. **ConcurrentModificationException**:
    
    - Модификация коллекции (кроме `remove` через итератор) во время итерации.
    - **Решение**: Используйте `Iterator.remove` или потокобезопасные коллекции.
    
    ```java
    List<String> list = new ArrayList<>(Arrays.asList("A", "B"));
    for (String s : list) {
        list.remove(s); // ConcurrentModificationException
    }
    ```
    
2. **UnsupportedOperationException**:
    
    - `remove` не поддерживается в некоторых итераторах (например, `Arrays.asList()`).
    - **Решение**: Проверяйте документацию коллекции.
3. **Истощение итератора**:
    
    - Повторный перебор требует создания нового итератора.
    - **Решение**: Вызывайте `iterator()` заново.
4. **Плохая производительность**:
    
    - Итерация по `LinkedList` медленнее, чем по `ArrayList` из-за доступа к узлам.
    - **Решение**: Выбирайте подходящую коллекцию.

## 8. Лучшие практики

1. **Используйте `for-each` для простоты**:
    
    ```java
    for (String s : list) {
        System.out.println(s);
    }
    ```
    
2. **Применяйте `Iterator` для удаления**:
    
    ```java
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        if (iterator.next().equals("A")) iterator.remove();
    }
    ```
    
3. **Используйте стримы для функционального стиля**:
    
    ```java
    list.stream().filter(s -> !s.equals("A")).forEach(System.out::println);
    ```
    
4. **Обеспечивайте потокобезопасность**:
    - Используйте `ConcurrentHashMap` или `CopyOnWriteArrayList`.
5. **Создавайте собственные итераторы для кастомных коллекций**:
    - Реализуйте `Iterable` и `Iterator` с учетом структуры данных.
6. **Проверяйте поддержку `remove`**:
    - Избегайте вызовов `remove` в неизменяемых коллекциях.

## 9. Современные возможности Java 21+

### Pattern Matching для Iterator и Iterable

Java 21 улучшил pattern matching, что полезно при работе с итераторами:

```java
Iterable<Object> mixed = List.of("string", 42, 3.14, null);

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

### Structured Concurrency с Iterator

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Iterator<String> iterator = List.of("A", "B", "C").iterator();
    List<Future<String>> futures = new ArrayList<>();
    
    while (iterator.hasNext()) {
        String item = iterator.next();
        futures.add(scope.fork(() -> processItem(item)));
    }
    
    scope.join();
    scope.throwIfFailed();
    
    List<String> results = futures.stream()
        .map(Future::resultNow)
        .collect(Collectors.toList());
}
```

### Виртуальные потоки и Iterator

```java
Iterable<String> data = List.of("a", "b", "c");
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<List<String>> future = CompletableFuture.supplyAsync(() -> {
    List<String> results = new ArrayList<>();
    for (String item : data) {
        results.add(item.toUpperCase());
    }
    return results;
}, executor);
```

## 10. Расширенные примеры использования

### Потокобезопасный Iterator с мониторингом

```java
public class MonitoredIterator<T> implements Iterator<T> {
    private final Iterator<T> delegate;
    private final AtomicLong accessCount = new AtomicLong(0);
    private final AtomicLong modificationCount = new AtomicLong(0);
    
    public MonitoredIterator(Iterator<T> delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public boolean hasNext() {
        accessCount.incrementAndGet();
        return delegate.hasNext();
    }
    
    @Override
    public T next() {
        accessCount.incrementAndGet();
        return delegate.next();
    }
    
    @Override
    public void remove() {
        modificationCount.incrementAndGet();
        delegate.remove();
    }
    
    public long getAccessCount() { return accessCount.get(); }
    public long getModificationCount() { return modificationCount.get(); }
}
```

### Функциональные операции с Iterator

```java
public class FunctionalIteratorOperations {
    
    // Трансформация итератора
    public static <T, R> Iterator<R> transformIterator(
            Iterator<T> iterator, Function<T, R> transformer) {
        return new Iterator<R>() {
            @Override
            public boolean hasNext() {
                return iterator.hasNext();
            }
            
            @Override
            public R next() {
                return transformer.apply(iterator.next());
            }
        };
    }
    
    // Фильтрация итератора
    public static <T> Iterator<T> filterIterator(
            Iterator<T> iterator, Predicate<T> predicate) {
        return new Iterator<T>() {
            private T nextItem = null;
            private boolean hasNext = false;
            
            @Override
            public boolean hasNext() {
                if (hasNext) return true;
                while (iterator.hasNext()) {
                    T item = iterator.next();
                    if (predicate.test(item)) {
                        nextItem = item;
                        hasNext = true;
                        return true;
                    }
                }
                return false;
            }
            
            @Override
            public T next() {
                if (!hasNext()) throw new NoSuchElementException();
                hasNext = false;
                return nextItem;
            }
        };
    }
    
    // Объединение итераторов
    public static <T> Iterator<T> concatIterators(Iterator<T>... iterators) {
        return new Iterator<T>() {
            private int currentIndex = 0;
            
            @Override
            public boolean hasNext() {
                while (currentIndex < iterators.length) {
                    if (iterators[currentIndex].hasNext()) {
                        return true;
                    }
                    currentIndex++;
                }
                return false;
            }
            
            @Override
            public T next() {
                if (!hasNext()) throw new NoSuchElementException();
                return iterators[currentIndex].next();
            }
        };
    }
    
    // Итератор с ограничением
    public static <T> Iterator<T> limitIterator(Iterator<T> iterator, int limit) {
        return new Iterator<T>() {
            private int count = 0;
            
            @Override
            public boolean hasNext() {
                return count < limit && iterator.hasNext();
            }
            
            @Override
            public T next() {
                if (!hasNext()) throw new NoSuchElementException();
                count++;
                return iterator.next();
            }
        };
    }
}
```

### Кастомные Iterable реализации

```java
public class CustomIterables {
    
    // Iterable для диапазона чисел
    public static Iterable<Integer> range(int start, int end) {
        return () -> new Iterator<Integer>() {
            private int current = start;
            
            @Override
            public boolean hasNext() {
                return current < end;
            }
            
            @Override
            public Integer next() {
                if (!hasNext()) throw new NoSuchElementException();
                return current++;
            }
        };
    }
    
    // Iterable для бесконечной последовательности
    public static Iterable<Integer> infiniteSequence(int start) {
        return () -> new Iterator<Integer>() {
            private int current = start;
            
            @Override
            public boolean hasNext() {
                return true;
            }
            
            @Override
            public Integer next() {
                return current++;
            }
        };
    }
    
    // Iterable для файлов в директории
    public static Iterable<Path> filesInDirectory(Path directory) {
        return () -> {
            try {
                return Files.list(directory).iterator();
            } catch (IOException e) {
                throw new RuntimeException("Error listing directory", e);
            }
        };
    }
    
    // Iterable для строк в файле
    public static Iterable<String> linesInFile(Path file) {
        return () -> {
            try {
                return Files.lines(file).iterator();
            } catch (IOException e) {
                throw new RuntimeException("Error reading file", e);
            }
        };
    }
}
```

### Интеграция с внешними системами

```java
public class IteratorIntegration {
    
    // Итератор для базы данных
    public static Iterator<User> databaseIterator(DataSource dataSource) {
        try {
            Connection conn = dataSource.getConnection();
            PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users");
            ResultSet rs = stmt.executeQuery();
            
            return new Iterator<User>() {
                private boolean hasNext = rs.next();
                
                @Override
                public boolean hasNext() {
                    return hasNext;
                }
                
                @Override
                public User next() {
                    if (!hasNext()) throw new NoSuchElementException();
                    try {
                        User user = new User(rs.getString("name"), rs.getInt("age"));
                        hasNext = rs.next();
                        return user;
                    } catch (SQLException e) {
                        throw new RuntimeException("Database error", e);
                    }
                }
            };
        } catch (SQLException e) {
            throw new RuntimeException("Database connection error", e);
        }
    }
    
    // Итератор для HTTP API
    public static Iterator<String> apiIterator(List<String> urls) {
        return new Iterator<String>() {
            private int currentIndex = 0;
            
            @Override
            public boolean hasNext() {
                return currentIndex < urls.size();
            }
            
            @Override
            public String next() {
                if (!hasNext()) throw new NoSuchElementException();
                return fetchData(urls.get(currentIndex++));
            }
            
            private String fetchData(String url) {
                // Реализация HTTP запроса
                return "data from " + url;
            }
        };
    }
    
    // Сериализация итератора
    public static <T extends Serializable> byte[] serializeIterator(Iterator<T> iterator) {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             ObjectOutputStream oos = new ObjectOutputStream(baos)) {
            List<T> list = new ArrayList<>();
            iterator.forEachRemaining(list::add);
            oos.writeObject(list);
            return baos.toByteArray();
        } catch (IOException e) {
            throw new RuntimeException("Serialization error", e);
        }
    }
    
    @SuppressWarnings("unchecked")
    public static <T extends Serializable> Iterator<T> deserializeIterator(byte[] data) {
        try (ByteArrayInputStream bais = new ByteArrayInputStream(data);
             ObjectInputStream ois = new ObjectInputStream(bais)) {
            List<T> list = (List<T>) ois.readObject();
            return list.iterator();
        } catch (IOException | ClassNotFoundException e) {
            throw new RuntimeException("Deserialization error", e);
        }
    }
}
```

## 11. Вопросы для собеседования

### Базовые вопросы

1. **Объясните разницу между Iterator и Iterable**
   - Iterable: интерфейс для объектов, которые можно перебирать
   - Iterator: интерфейс для последовательного доступа к элементам
   - Iterable предоставляет Iterator через метод iterator()

2. **Что такое for-each и как он работает?**
   - Синтаксический сахар для итерации
   - Автоматически вызывает iterator() и использует Iterator
   - Работает с любым объектом, реализующим Iterable

3. **Объясните производительность Iterator**
   - hasNext/next: O(1) для большинства коллекций
   - remove: O(1) для ArrayList, O(n) для LinkedList
   - Итерация: O(n) для всех коллекций

### Продвинутые вопросы

4. **Как работает внутренняя реализация Iterator?**
   - Зависит от коллекции (индекс, указатель, итератор по корзинам)
   - Отслеживает модификации через modCount
   - Проверяет ConcurrentModificationException

5. **Объясните многопоточность в Iterator**
   - Iterator не потокобезопасен
   - ConcurrentModificationException при модификации во время итерации
   - ConcurrentHashMap не выбрасывает исключения

6. **Как реализовать кастомный Iterator?**
```java
public class CustomIterator<T> implements Iterator<T> {
    private final T[] data;
    private int index = 0;
    
    public CustomIterator(T[] data) {
        this.data = data;
    }
    
    @Override
    public boolean hasNext() {
        return index < data.length;
    }
    
    @Override
    public T next() {
        if (!hasNext()) throw new NoSuchElementException();
        return data[index++];
    }
}
```

### Практические вопросы

7. **Напишите код для безопасного удаления элементов**
```java
public static <T> void removeElements(Iterable<T> iterable, Predicate<T> predicate) {
    Iterator<T> iterator = iterable.iterator();
    while (iterator.hasNext()) {
        if (predicate.test(iterator.next())) {
            iterator.remove();
        }
    }
}
```

8. **Реализуйте итератор для бесконечной последовательности**
```java
public static Iterator<Integer> infiniteIterator(int start) {
    return new Iterator<Integer>() {
        private int current = start;
        
        @Override
        public boolean hasNext() {
            return true;
        }
        
        @Override
        public Integer next() {
            return current++;
        }
    };
}
```

9. **Как объединить несколько итераторов?**
```java
public static <T> Iterator<T> combineIterators(Iterator<T>... iterators) {
    return new Iterator<T>() {
        private int currentIndex = 0;
        
        @Override
        public boolean hasNext() {
            while (currentIndex < iterators.length) {
                if (iterators[currentIndex].hasNext()) {
                    return true;
                }
                currentIndex++;
            }
            return false;
        }
        
        @Override
        public T next() {
            if (!hasNext()) throw new NoSuchElementException();
            return iterators[currentIndex].next();
        }
    };
}
```

## 12. Пример: Комплексное использование

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Logger;

public class IteratorExample {
    private static final Logger LOGGER = Logger.getLogger(IteratorExample.class.getName());

    public static void main(String[] args) {
        // ArrayList с Iterator
        List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
        Iterator<String> iterator = list.iterator();
        while (iterator.hasNext()) {
            String s = iterator.next();
            if (s.equals("B")) iterator.remove();
        }
        LOGGER.info("ArrayList after removal: " + list);

        // for-each
        for (String s : list) {
            LOGGER.info("for-each: " + s);
        }

        // Stream
        list.stream().filter(s -> s.equals("A")).forEach(s -> LOGGER.info("Stream: " + s));

        // ConcurrentHashMap
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        map.put("X", 1);
        map.put("Y", 2);
        Iterator<Map.Entry<String, Integer>> mapIterator = map.entrySet().iterator();
        new Thread(() -> map.put("Z", 3)).start(); // Безопасно
        while (mapIterator.hasNext()) {
            LOGGER.info("ConcurrentHashMap: " + mapIterator.next());
        }

        // forEachRemaining
        Iterator<String> iterator2 = list.iterator();
        iterator2.next();
        iterator2.forEachRemaining(s -> LOGGER.info("forEachRemaining: " + s));
        
        // Кастомный итератор
        Iterator<Integer> rangeIterator = CustomIterables.range(1, 5).iterator();
        while (rangeIterator.hasNext()) {
            LOGGER.info("Range: " + rangeIterator.next());
        }
    }
}
```

**Предполагаемый вывод**:

```
INFO: ArrayList after removal: [A, C]
INFO: for-each: A
INFO: for-each: C
INFO: Stream: A
INFO: ConcurrentHashMap: X=1
INFO: ConcurrentHashMap: Y=2
INFO: ConcurrentHashMap: Z=3
INFO: forEachRemaining: C
INFO: Range: 1
INFO: Range: 2
INFO: Range: 3
INFO: Range: 4
```