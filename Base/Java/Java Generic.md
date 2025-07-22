## Зачем нужны дженерики?

Без дженериков коллекции и другие универсальные структуры в Java были "сырыми" (raw types), что требовало явного приведения типов при извлечении данных, что могло привести к ошибкам во время выполнения:

```java
List list = new ArrayList();
list.add("Привет");
list.add(42); // Можно добавить разные типы
String s = (String) list.get(0); // Требуется приведение
// Integer i = (Integer) list.get(1); // ClassCastException во время выполнения
```

С дженериками код становится типобезопасным и читаемым:

```java
List<String> list = new ArrayList<>();
list.add("Привет");
// list.add(42); // Ошибка компиляции
String s = list.get(0); // Приведение не требуется
```

**Преимущества дженериков**:

- **Типобезопасность**: Компилятор проверяет корректность типов, предотвращая ошибки вроде `ClassCastException`.
- **Упрощение кода**: Устраняется необходимость явного приведения типов.
- **Переиспользуемость**: Универсальные классы и методы работают с любыми типами.
- **Информативность**: Код становится более выразительным благодаря явным типам.

**Недостатки**:

- Сложность понимания для новичков, особенно с wildcards и стиранием типов.
- Накладные расходы из-за стирания типов (type erasure) в байт-коде.
- Ограничения, такие как невозможность создавать экземпляры типа `T` напрямую.
## Синтаксис дженериков
### Объявление обобщенных классов

Обобщенный класс объявляется с параметром типа (например, `T`), который указывается в угловых скобках `<T>`:

```java
public class Box<T> {
    private T value;

    public Box(T value) {
        this.value = value;
    }

    public void set(T value) {
        this.value = value;
    }

    public T get() {
        return value;
    }
}
```

Использование:

```java
Box<String> stringBox = new Box<>("Привет");
Box<Integer> intBox = new Box<>(42);
System.out.println(stringBox.get()); // Привет
System.out.println(intBox.get()); // 42
```

**Заметки**:

- `T` — это параметр типа (type parameter), который подставляется при создании экземпляра.
- Можно использовать несколько параметров типа, например, `<K, V>` для ключей и значений.
### Обобщенные интерфейсы

Интерфейсы также могут быть обобщенными:

```java
public interface Pair<K, V> {
    K getKey();
    V getValue();
}

public class OrderedPair<K, V> implements Pair<K, V> {
    private final K key;
    private final V value;

    public OrderedPair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    @Override
    public K getKey() { return key; }

    @Override
    public V getValue() { return value; }
}
```

Использование:

```java
Pair<String, Integer> pair = new OrderedPair<>("Возраст", 25);
System.out.println(pair.getKey() + ": " + pair.getValue()); // Возраст: 25
```
### Обобщенные методы

Методы могут быть обобщенными независимо от класса:

```java
public class Util {
    public static <T> void printArray(T[] array) {
        for (T elem : array) {
            System.out.println(elem);
        }
    }
}
```

Использование:

```java
String[] strings = {"А", "Б", "В"};
Integer[] numbers = {1, 2, 3};
Util.printArray(strings); // А, Б, В
Util.printArray(numbers); // 1, 2, 3
```
### Обобщенные конструкторы

Конструкторы могут иметь собственные параметры типа, отличные от параметров класса:

```java
public class Account {
    private String id;
    private int sum;

    public <T> Account(T id, int sum) {
        this.id = id.toString();
        this.sum = sum;
    }

    public String getId() { return id; }
    public int getSum() { return sum; }
}
```

Использование:

```java
Account acc1 = new Account("ID123", 5000);
Account acc2 = new Account(456, 3000);
System.out.println(acc1.getId()); // ID123
System.out.println(acc2.getId()); // 456
```
## Ограничения типов (Bounded Types)

Дженерики позволяют ограничивать типы, которые могут быть использованы в качестве параметров.
### Верхняя граница (`extends`)

Ограничение типа сверху указывает, что параметр типа должен быть подклассом указанного типа (включая сам тип):

```java
public class NumberBox<T extends Number> {
    private T value;

    public NumberBox(T value) {
        this.value = value;
    }

    public double toDouble() {
        return value.doubleValue();
    }
}
```

Использование:

```java
NumberBox<Integer> intBox = new NumberBox<>(42);
NumberBox<Double> doubleBox = new NumberBox<>(3.14);
// NumberBox<String> stringBox = new NumberBox<>("text"); // Ошибка компиляции
System.out.println(intBox.toDouble()); // 42.0
System.out.println(doubleBox.toDouble()); // 3.14
```
### Множественные ограничения

Параметр типа может быть ограничен несколькими интерфейсами и одним классом:

```java
public class ComparableBox<T extends Number & Comparable<T>> {
    private T value;

    public ComparableBox(T value) {
        this.value = value;
    }

    public int compareTo(T other) {
        return value.compareTo(other);
    }
}
```

Использование:

```java
ComparableBox<Integer> intBox = new ComparableBox<>(42);
System.out.println(intBox.compareTo(50)); // -1
// ComparableBox<String> stringBox = new ComparableBox<>("text"); // Ошибка компиляции
```

**Заметка**: Класс должен быть указан первым в списке ограничений, за ним следуют интерфейсы.
### Ограничение снизу (`super`)

Ограничение снизу с помощью `super` используется только в wildcards (см. ниже), но не в определении параметров типа:

```java
// Неверно
public class Box<T super Number> { // Ошибка компиляции
    private T value;
}
```
## Wildcards (`?`) — Подстановочные типы

Wildcards позволяют работать с обобщениями, когда точный тип неизвестен или не важен. Они повышают гибкость кода, но требуют понимания их поведения.
### Основные виды wildcards

|Wildcard|Описание|Пример|Чтение|Запись|
|---|---|---|---|---|
|`<?>`|Любой тип (неограниченный wildcard)|`List<?>`|Да (как `Object`)|Нет (кроме `null`)|
|`<? extends T>`|Любой тип, наследник T (верхняя граница)|`List<? extends Number>`|Да (как T)|Нет|
|`<? super T>`|Любой тип, родитель T (нижняя граница)|`List<? super Integer>`|Нет (только как `Object`)|Да (T и его подтипы)|
### Неограниченный wildcard (`<?>`)

```java
public void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

List<String> strings = List.of("А", "Б");
List<Integer> numbers = List.of(1, 2);
printList(strings); // А, Б
printList(numbers); // 1, 2
// list.add("В"); // Ошибка компиляции
```

**Особенности**:

- Можно читать элементы как `Object`.
- Можно добавлять только `null`.
### Верхняя граница (`<? extends T>`)

Используется, когда коллекция является **источником** данных (producer):

```java
public double sum(List<? extends Number> list) {
    double result = 0;
    for (Number num : list) {
        result += num.doubleValue();
    }
    return result;
}

List<Integer> ints = List.of(1, 2, 3);
List<Double> doubles = List.of(1.5, 2.5);
System.out.println(sum(ints)); // 6.0
System.out.println(sum(doubles)); // 4.0
// list.add(42); // Ошибка компиляции
```

**Особенности**:

- Можно читать элементы как тип `T` (или его суперкласс).
- Нельзя добавлять элементы, так как точный тип неизвестен.
### Нижняя граница (`<? super T>`)

Используется, когда коллекция является **потребителем** данных (consumer):

```java
public void addNumbers(List<? super Integer> list) {
    list.add(42);
    list.add(100);
    // Object obj = list.get(0); // Только как Object
}

List<Number> numbers = new ArrayList<>();
List<Object> objects = new ArrayList<>();
addNumbers(numbers);
addNumbers(objects);
System.out.println(numbers); // [42, 100]
System.out.println(objects); // [42, 100]
```

**Особенности**:

- Можно добавлять элементы типа `T` и его подтипы.
- Чтение ограничено типом `Object`.
### Правило PECS (Producer Extends, Consumer Super)

**PECS** — это мнемоническое правило для выбора wildcard:

- **Producer (`extends`)**: Если коллекция производит данные (чтение), используйте `<? extends T>`.
- **Consumer (`super`)**: Если коллекция потребляет данные (запись), используйте `<? super T>`.

Пример:

```java
// Producer: читаем из источника
public void printNumbers(List<? extends Number> source) {
    for (Number num : source) {
        System.out.println(num);
    }
}

// Consumer: записываем в коллекцию
public void copyIntegers(List<? super Integer> destination, List<Integer> source) {
    destination.addAll(source);
}
```
## Стирание типов (Type Erasure)

Дженерики в Java реализованы через **стирание типов** (type erasure), чтобы обеспечить обратную совместимость с кодом до Java 5. Во время компиляции информация о параметрах типа удаляется, и обобщенные типы заменяются их верхними границами (обычно `Object`).
### Как работает стирание типов?

Рассмотрим класс:

```java
public class Box<T> {
    private T value;

    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
```

После компиляции:

```java
public class Box {
    private Object value;

    public void set(Object value) { this.value = value; }
    public Object get() { return value; }
}
```

Если есть ограничение:

```java
public class Box<T extends Number> {
    private T value;

    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
```

После компиляции:

```java
public class Box {
    private Number value;

    public void set(Number value) { this.value = value; }
    public Number get() { return value; }
}
```

**Последствия стирания типов**:

**Потеря информации о типе во время выполнения**:

```java
List<String> strings = new ArrayList<>();
List<Integer> numbers = new ArrayList<>();
System.out.println(strings.getClass() == numbers.getClass()); // true
```

В байт-коде оба списка — это просто `ArrayList`.

**Невозможность создания экземпляров `T`**:

```java
T obj = new T(); // Ошибка компиляции
```

**Ограничения с массивами**:

```java
List<String>[] array = new ArrayList<String>[10]; // Ошибка компиляции
```

**Ограничения с `instanceof`**:

```java
if (strings instanceof List<String>) { } // Ошибка компиляции
```
### Стирание в обобщенных методах

```java
public class Util {
    public static <T> void print(T value) {
        System.out.println(value);
    }
}
```

После компиляции:

```java
public class Util {
    public static void print(Object value) {
        System.out.println(value);
    }
}
```

Компилятор добавляет неявные приведения типов, чтобы обеспечить типобезопасность.
## Дженерики и наследование

Дженерики не поддерживают ковариантность или контравариантность в полной мере из-за стирания типов. Например:

```java
List<String> strings = new ArrayList<>();
// List<Object> objects = strings; // Ошибка компиляции
```

Хотя `String` наследует `Object`, `List<String>` не является подтипом `List<Object>`. Это связано с тем, что `List<Object>` позволяет добавлять любые объекты, что нарушило бы типобезопасность `List<String>`.

**Решение**: Используйте wildcards:

```java
List<? extends Object> objects = new ArrayList<String>();
```
### Приведение обобщенных типов

```java
public class Account<T> {
    private T id;

    public Account(T id) {
        this.id = id;
    }

    public T getId() {
        return id;
    }
}

public class DepositAccount<T> extends Account<T> {
    public DepositAccount(T id) {
        super(id);
    }
}

DepositAccount<Integer> depAccount = new DepositAccount<>(10);
Account<Integer> account = depAccount; // OK
System.out.println(account.getId()); // 10
// Account<String> stringAccount = depAccount; // Ошибка компиляции
```
## Дженерики и Reflection API

Reflection API позволяет анализировать и работать с обобщенными типами во время выполнения, несмотря на стирание типов. Ключевые классы и интерфейсы:

- **`java.lang.reflect.Type`**: Базовый интерфейс для представления типов, включая обобщенные.
- **`ParameterizedType`**: Представляет параметризованный тип (например, `List<String>`).
- **`GenericArrayType`**: Представляет массив обобщенных типов.
- **`TypeVariable`**: Представляет переменную типа (например, `T`).
- **`WildcardType`**: Представляет wildcard-типы (например, `? extends Number`).

**Пример: Анализ обобщенного типа поля**

```java
import java.lang.reflect.*;

public class GenericExample {
    private List<String> strings;

    public static void main(String[] args) throws NoSuchFieldException {
        Field field = GenericExample.class.getDeclaredField("strings");
        Type type = field.getGenericType();
        if (type instanceof ParameterizedType) {
            ParameterizedType pType = (ParameterizedType) type;
            // interface java.util.List
            System.out.println(pType.getRawType()); 
            // class java.lang.String
            System.out.println(pType.getActualTypeArguments()[0]);
        }
    }
}
```

**Анализ обобщенного метода**:

```java
public class Util {
    public static <T extends Comparable<T>> T max(T a, T b) {
        return a.compareTo(b) > 0 ? a : b;
    }

    public static void main(String[] args) throws NoSuchMethodException {
        Method method = Util.class.getMethod(
	        "max", Comparable.class, Comparable.class
		);
        TypeVariable<Method>[] typeParams = method.getTypeParameters();
        for (TypeVariable<Method> param : typeParams) {
	        // T
            System.out.println(param.getName()); 
            // interface java.lang.Comparable
            System.out.println(param.getBounds()[0]); 
        }
    }
}
```

**Ключевые моменты**:

- Информация о дженериках сохраняется в метаданных класса (доступных через `getGenericType`, `getGenericParameterTypes` и т.д.).
- Стирание типов не влияет на метаданные, но ограничивает доступ к типам во время выполнения.
- Используйте `ParameterizedType` для анализа типов коллекций, `TypeVariable` для параметров типа, и `WildcardType` для wildcard.
## Дженерики в стандартной библиотеке Java

Дженерики широко используются в стандартной библиотеке Java, особенно в коллекциях, компараторах и других утилитах.
### Коллекции

Классы `List`, `Set`, `Map` и другие используют дженерики для типобезопасности:

```java
List<String> list = new ArrayList<>();
Map<Integer, String> map = new HashMap<>();
Set<Double> set = new HashSet<>();
```

**Пример**:

```java
Map<String, Integer> scores = new HashMap<>();
scores.put("Алексей", 95);
scores.put("Мария", 88);
System.out.println(scores.get("Алексей")); // 95
```
### Компараторы

Интерфейс `Comparator<T>` использует дженерики для сравнения объектов:

```java
Comparator<String> lengthComparator = (s1, s2) -> s1.length() - s2.length();
List<String> names = Arrays.asList("Алексей", "Мария", "Анна");
Collections.sort(names, lengthComparator);
System.out.println(names); // [Анна, Мария, Алексей]
```
### Другие утилиты

Класс `Optional<T>` (Java 8) использует дженерики для работы с необязательными значениями:

```java
Optional<String> optional = Optional.of("Привет");
System.out.println(optional.orElse("Пусто")); // Привет
```
## Дженерики в Java 8 и выше (Stream API)

Java 8 ввела Stream API, которая активно использует дженерики для обработки данных:

```java
List<String> names = Arrays.asList("Алексей", "Мария", "Анна");
List<String> filtered = names.stream()
    .filter(s -> s.length() > 4)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
System.out.println(filtered); // [АЛЕКСЕЙ, МАРИЯ]
```

**Ключевые аспекты**:

- `Stream<T>` — обобщенный интерфейс, где `T` — тип элементов потока.
- Методы, такие как `map` и `filter`, используют обобщенные функциональные интерфейсы (`Function<T, R>`, `Predicate<T>`).
- Wildcards используются в методах Stream API для гибкости, например, `Stream<? extends T>`.

**Пример с wildcard**:

```java
public void processNumbers(Stream<? extends Number> stream) {
    stream.forEach(n -> System.out.println(n.doubleValue()));
}

Stream<Integer> intStream = Stream.of(1, 2, 3);
Stream<Double> doubleStream = Stream.of(1.5, 2.5);
processNumbers(intStream); // 1.0, 2.0, 3.0
processNumbers(doubleStream); // 1.5, 2.5
```
## Подводные камни и ограничения

1. **Стирание типов**:
    
    - Типы недоступны во время выполнения, что ограничивает возможности рефлексии и проверок `instanceof`.
    - Решение: Используйте `ParameterizedType` или передавайте `Class<T>` для сохранения информации о типе.
2. **Невозможность создания экземпляров `T`**:
    
    - Нельзя написать `new T()`. Решение: Передавайте `Class<T>` или фабрику:
        
        ```java
        public <T> T create(Class<T> clazz) throws Exception {
            return clazz.getDeclaredConstructor().newInstance();
        }
        ```

3. **Массивы обобщенных типов**:
    
    - Нельзя создать `new T[10]` или `new List<String>[10]`. Решение: Используйте `Object[]` с приведением или `List<T>`.
4. **Static-поля и дженерики**:
    
    - Параметры типа не могут использоваться в статических полях или методах напрямую:
        
        ```java
        public class Box<T> {
            // static T value; // Ошибка компиляции
        }
        ```

5. **Исключения и дженерики**:
    
    - Нельзя создавать обобщенные исключения:
        
        ```java
        public class MyException<T> extends Exception { } // Ошибка компиляции
        ```

6. **Wildcards и сложность**:
    
    - Неправильное использование `extends` или `super` может привести к запутанному коду. Следуйте правилу PECS.

## Лучшие практики

1. **Используйте дженерики для типобезопасности**: Всегда указывайте типы в коллекциях и других обобщенных классах.
2. **Применяйте PECS для wildcards**: Используйте `extends` для источников данных, `super` для потребителей.
3. **Кэшируйте Class для рефлексии**: Сохраняйте объекты `Class<T>` для избежания повторных вызовов `Class.forName`.
4. **Избегайте raw types**:
    
```java
List rawList = new ArrayList(); // Плохо
List<String> typedList = new ArrayList<>(); // Хорошо
```
    
5. **Документируйте ограничения типов**: Указывайте в документации, какие типы ожидаются для `T`.
6. **Тестируйте сложные случаи**: Проверяйте поведение с wildcards и наследованием.
## Пример: Реализация обобщенного хранилища

```java
import java.util.*;

public class GenericRepository<T extends Comparable<T>> {
    private final List<T> items = new ArrayList<>();

    public void add(T item) {
        items.add(item);
    }

    public List<T> getSorted() {
        List<T> sorted = new ArrayList<>(items);
        Collections.sort(sorted);
        return sorted;
    }

    public <U extends T> List<U> filterByType(Class<U> clazz) {
        List<U> result = new ArrayList<>();
        for (T item : items) {
            if (clazz.isInstance(item)) {
                result.add(clazz.cast(item));
            }
        }
        return result;
    }
}

Integer number = 42;
GenericRepository<Number> repo = new GenericRepository<>();
repo.add(number);
repo.add(3.14);
List<Number> sorted = repo.getSorted(); // [3.14, 42]
List<Integer> integers = repo.filterByType(Integer.class); // [42]
```