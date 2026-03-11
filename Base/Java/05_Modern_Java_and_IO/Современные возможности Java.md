---
title: "Современные возможности Java — Records, Sealed, Pattern Matching"
tags: [java, modern-java, records, sealed-classes, pattern-matching, java17, java21]
updated: 2026-03-11
---

# Современные возможности Java (ООП)

> [!QUOTE] Суть
> **Records** (Java 16+) — иммутабельные data-классы с автогенерацией equals/hashCode/toString/getters. **Sealed classes** (Java 17+) — закрытая иерархия (`permits`), исчерпывающий `switch`. **Pattern Matching** `instanceof` (Java 16+) и `switch` (Java 21+) — безопасный downcasting без явного cast.

## Содержание

1. [Pattern Matching (Java 16+)](#pattern-matching-java-16)
2. [Sealed-классы (Java 17+)](#sealed-классы-java-17)
3. [Records (Java 14+)](#records-java-14)
4. [Где узнать больше](#где-узнать-больше)

---

## Pattern Matching (Java 16+)

Pattern Matching (сопоставление с образцом) упрощает работу с операторами `instanceof` и `switch`, делая код короче и безопаснее.

**Пример с instanceof:**
```java
Object obj = "Hello";
if (obj instanceof String s) {
    System.out.println(s.toUpperCase()); // HELLO
}
```

**Пример с switch (Java 17+):**
```java
Object obj = 42;
String result = switch (obj) {
    case String s -> "Строка: " + s;
    case Integer i -> "Число: " + i;
    default -> "Неизвестный тип";
};
System.out.println(result); // Число: 42
```

**Преимущества:**
- Меньше явных приведений типов
- Безопасность типов на этапе компиляции
- Более читаемый код

---

## Sealed-классы (Java 17+)

Sealed-классы позволяют явно ограничить круг наследников класса или интерфейса.

**Пример:**
```java
public sealed class Shape permits Circle, Rectangle {}
public final class Circle extends Shape {}
public final class Rectangle extends Shape {}
```

- Класс `Shape` может быть расширен только классами `Circle` и `Rectangle`.
- Нарушение ограничения приведёт к ошибке компиляции.

**Преимущества:**
- Явный контроль иерархии
- Безопасность при использовании pattern matching
- Упрощение поддержки и рефакторинга

---

## Records (Java 14+)

Records — это компактный способ объявления неизменяемых классов-«контейнеров» для данных.

**Пример:**
```java
public record Point(int x, int y) {}

// Автогенерируется компилятором:
// - Конструктор: Point(int x, int y)
// - Аксессоры: int x(), int y()  (не getX() — именно x()!)
// - equals(Object), hashCode(), toString()

Point p = new Point(1, 2);
System.out.println(p.x()); // 1
System.out.println(p);     // Point[x=1, y=2]
```

- Все компонентные поля `private final` — класс иммутабельный.
- Не может `extends` другой класс (неявно `extends Record`), но может реализовывать интерфейсы.
- Подходит для sealed-иерархий и Pattern Matching.

---

## var — локальный вывод типов (Java 10+)

`var` позволяет компилятору вывести тип локальной переменной автоматически:

```java
var list = new ArrayList<String>();     // List<String>
var map = new HashMap<String, Integer>(); // Map<String, Integer>
var name = "Alice";                     // String
var count = 42;                         // int

// В for-each:
for (var entry : map.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
}

// В try-with-resources:
try (var stream = Files.lines(Path.of("file.txt"))) {
    stream.forEach(System.out::println);
}
```

**Ограничения `var`:**
- Только **локальные переменные** — нельзя для полей, параметров методов, возвращаемых типов
- Обязательная инициализация при объявлении (компилятор должен вывести тип)
- `var x = null;` — запрещено (тип неопределён)
- `var` — не ключевое слово, а reserved type name: можно использовать `var` как имя переменной (но не стоит)

---

## Text Blocks (Java 15+)

Text Blocks — многострочные строковые литералы без escape-последовательностей:

```java
// Старый способ:
String json = "{\n    \"name\": \"Alice\",\n    \"age\": 30\n}";

// Text Block (Java 15+):
String json = """
        {
            "name": "Alice",
            "age": 30
        }
        """;

// SQL:
String sql = """
        SELECT u.name, u.email
        FROM users u
        WHERE u.active = true
          AND u.age > 18
        ORDER BY u.name
        """;

// HTML:
String html = """
        <html>
            <body>
                <p>Hello, %s!</p>
            </body>
        </html>
        """.formatted("World");
```

**Специальные escape-последовательности в Text Blocks:**
- `\` в конце строки — убрать перевод строки (line continuation)
- `\s` — пробел (для сохранения trailing spaces)

```java
// Без \s — trailing spaces обрезаются:
String s = """
        Alice
        Bob
        """;

// С \s — trailing spaces сохранятся до позиции \s:
String s = """
        Alice  \s
        Bob    \s
        """;
```

---

## Record Patterns (Java 21)

Record Patterns расширяют pattern matching для деструктуризации Records:

```java
record Point(int x, int y) {}
record Circle(Point center, double radius) {}

// Java 16: instanceof с именованной переменной
if (shape instanceof Point p) {
    System.out.println(p.x() + ", " + p.y());
}

// Java 21: Record Pattern — деструктуризация в instanceof
if (shape instanceof Point(var x, var y)) {
    System.out.println(x + ", " + y); // x и y — напрямую
}

// Вложенные Record Patterns:
if (shape instanceof Circle(Point(var x, var y), var r)) {
    System.out.println("Center: " + x + ", " + y + ", radius: " + r);
}
```

**Record Patterns в switch (Java 21):**

```java
static String describe(Object shape) {
    return switch (shape) {
        case Point(var x, var y) -> "Point at (" + x + ", " + y + ")";
        case Circle(Point(var x, var y), var r) ->
            "Circle at (" + x + ", " + y + ") with radius " + r;
        case null -> "null";
        default -> "Unknown shape";
    };
}
```

---

## Guard Patterns в switch (Java 21)

**Guarded patterns** (`when`) позволяют добавить условие к case:

```java
// Java 21: case с guard condition
static String classify(Object obj) {
    return switch (obj) {
        case Integer i when i < 0  -> "negative integer: " + i;
        case Integer i when i == 0 -> "zero";
        case Integer i             -> "positive integer: " + i;
        case String s when s.isEmpty() -> "empty string";
        case String s              -> "string: " + s;
        case null                  -> "null";
        default                    -> "other: " + obj;
    };
}

// Комбинация с Record Patterns:
static double area(Shape shape) {
    return switch (shape) {
        case Circle(_, var r) when r > 0 -> Math.PI * r * r;
        case Circle(_, var r)            -> throw new IllegalArgumentException("r=" + r);
        case Rectangle(var w, var h) when w > 0 && h > 0 -> w * h;
        default -> 0;
    };
}
```

---

## Sequenced Collections (Java 21)

Java 21 добавил три новых интерфейса для коллекций с **определённым порядком** элементов:

```
SequencedCollection<E>  (extends Collection<E>)
SequencedSet<E>         (extends Set<E>, SequencedCollection<E>)
SequencedMap<K,V>       (extends Map<K,V>)
```

**Новые методы `SequencedCollection`:**

```java
// Получить первый/последний элемент (без итерации!):
list.getFirst();   // эквивалент list.get(0)
list.getLast();    // эквивалент list.get(list.size() - 1)

// Добавить первым/последним:
list.addFirst(element);
list.addLast(element);

// Удалить первый/последний:
list.removeFirst();
list.removeLast();

// Перевёрнутый вид (view, не копия):
SequencedCollection<E> reversed = list.reversed();
```

**Какие классы реализуют:**

|Класс|Интерфейс|
|---|---|
|`ArrayList`, `LinkedList`|`SequencedCollection`|
|`LinkedHashSet`|`SequencedSet`|
|`LinkedHashMap`|`SequencedMap`|
|`TreeSet`|`SequencedSet`|
|`TreeMap`|`SequencedMap`|

**Пример:**

```java
LinkedHashSet<String> set = new LinkedHashSet<>();
set.add("B"); set.add("A"); set.add("C");

System.out.println(set.getFirst());   // "B" (первый добавленный)
System.out.println(set.getLast());    // "C"

// Reversed view:
for (String s : set.reversed()) {
    System.out.println(s); // C, A, B
}

// SequencedMap:
LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("one", 1); map.put("two", 2); map.put("three", 3);

Map.Entry<String, Integer> first = map.firstEntry(); // {one=1}
Map.Entry<String, Integer> last = map.lastEntry();   // {three=3}
```

**Зачем нужны:** раньше `LinkedHashSet` не имел метода получить первый/последний элемент без итерации (`iterator().next()`). `LinkedHashMap` требовал `entrySet().iterator().next()`. Новые интерфейсы унифицируют API.

---

## Scoped Values (Java 21)

Новая альтернатива `ThreadLocal` для передачи иммутабельного контекста в виртуальные потоки. Подробнее: [[Scoped Values (Java 21, JEP 446)]].

```java
static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

ScopedValue.where(CURRENT_USER, user).run(() -> {
    // user доступен в этом scope и дочерних задачах
    processRequest();
});
// Вне scope: CURRENT_USER.isBound() == false
```

---

## Unnamed Patterns и Unnamed Variables (Java 22+)

**Unnamed Pattern** (`_`) в `switch` и `instanceof` — игнорируем переменную без имени:

```java
// Java 22: Unnamed Pattern Variable (JEP 456)
// _ вместо именованной переменной, если значение не нужно
if (obj instanceof Point(var x, _)) {          // игнорируем y
    System.out.println("x=" + x);
}

// В switch — unnamed pattern:
switch (shape) {
    case Circle(_, var r) when r > 0 -> Math.PI * r * r;  // center не нужен
    case Circle(_, _)                -> 0;                  // ничего не нужно
    case Rectangle(var w, var h)     -> w * h;
}

// Unnamed Variable в блоках кода (не только patterns):
try {
    riskyOperation();
} catch (Exception _) {      // исключение поймано, но не используется
    log("operation failed");
}

for (var _ : list) {         // итерируем N раз, элемент не нужен
    counter++;
}
```

> [!INFO] Зачем `_`?
> До Java 22 нужно было придумывать имя (`ignored`, `unused`, `e`). Теперь компилятор явно знает "эта переменная намеренно игнорируется" — улучшает читаемость и позволяет JIT более агрессивно оптимизировать.

---

## Primitive Types in Patterns (Java 23, Preview)

Java 23 вводит поддержку примитивных типов в pattern matching (JEP 455, preview):

```java
// До Java 23: примитивы нельзя использовать в switch pattern matching
// Object value = 42;
// case int i -> ... // ошибка компиляции

// Java 23 Preview: примитивные типы в switch
static String describe(Object obj) {
    return switch (obj) {
        case int i    when i < 0   -> "negative int: " + i;
        case int i                 -> "positive int: " + i;
        case long l                -> "long: " + l;
        case float f               -> "float: " + f;
        case double d              -> "double: " + d;
        case String s              -> "string: " + s;
        default                    -> "other";
    };
}

// Также в instanceof:
if (obj instanceof int i) {  // Java 23 Preview
    System.out.println("int: " + i);
}
```

> [!WARNING] Статус в Java 25
> Primitive Types in Patterns был Preview в Java 23-24. В Java 25 (сентябрь 2025) ожидается финализация. Проверяй актуальный статус через [JEP Index](https://openjdk.org/jeps/).

---

## Records: Senior-уровень — что генерирует javac

> [!INFO] Senior: понимание Records на уровне байт-кода
> Records — не просто синтаксический сахар. Разберём что именно генерирует компилятор.

```java
// Исходник:
public record Point(int x, int y) implements Comparable<Point> {
    // Compact constructor (валидация без повторения полей)
    public Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("negative coords");
        // this.x = x; this.y = y; — неявно добавляется компилятором в конец!
    }

    // Кастомный accessor (переопределяем x()):
    @Override
    public int x() { return Math.abs(x); }  // можно, если нужно

    // Статические фабричные методы — можно добавлять
    public static Point origin() { return new Point(0, 0); }

    // Можно реализовывать интерфейсы
    @Override
    public int compareTo(Point other) { return Integer.compare(x, other.x); }
}
```

**Что генерирует javac (байт-код эквивалент):**

```java
// Скомпилированный эквивалент (упрощённо):
public final class Point extends java.lang.Record implements Comparable<Point> {
    private final int x;  // private final — всегда!
    private final int y;

    // Canonical constructor:
    public Point(int x, int y) {
        // super() — неявно вызывает Record()
        if (x < 0 || y < 0) throw new IllegalArgumentException("negative coords");
        this.x = x;  // неявно добавлено из compact constructor
        this.y = y;
    }

    // Accessors (НЕ getX(), а именно x() — это контракт Record!):
    public int x() { return Math.abs(x); }  // переопределён
    public int y() { return this.y; }       // сгенерирован

    // equals: использует invokedynamic + RecordComponents (не через reflection!)
    // Это делает equals Records значительно быстрее рефлексивного equals
    @Override public boolean equals(Object o) { /* invokedynamic */ }

    // hashCode: аналогично через invokedynamic + bootstrap method
    @Override public int hashCode() { /* invokedynamic */ }

    // toString: "Point[x=1, y=2]" — через invokedynamic
    @Override public String toString() { /* invokedynamic */ }
}
```

**Senior-нюансы Records:**

```java
// 1. Records и сериализация — Records реализуют Serializable корректно!
//    При десериализации ВСЕГДА вызывается canonical constructor
//    (в отличие от обычных классов где конструктор пропускается)
public record Config(String host, int port) implements Serializable {}
// Это БЕЗОПАСНО: валидация в constructor всегда выполняется

// 2. Records с Generic:
public record Pair<A, B>(A first, B second) {}
Pair<String, Integer> p = new Pair<>("hello", 42);
// Type erasure работает как обычно

// 3. Records НЕ могут extends другой класс (кроме Record)
// Records МОГУТ implements любые интерфейсы

// 4. Records и Pattern Matching (Java 21):
static void process(Object obj) {
    if (obj instanceof Point(int x, int y)) {
        // x и y — напрямую без obj.x()
    }
}

// 5. Records как DTO в Spring (JSON через Jackson):
// Jackson 2.12+ автоматически обнаруживает Record constructor
// Не нужны @JsonProperty — имена компонентов = имена JSON полей
@GetMapping("/point")
public Point getPoint() {
    return new Point(1, 2);  // → {"x":1,"y":2}
}
```

> [!WARNING] Ловушка: Records не полностью иммутабельны
> Если компонент — мутабельный объект, Record не защищает от изменений:
> ```java
> record Wrapper(List<String> items) {}
> var w = new Wrapper(new ArrayList<>(List.of("a")));
> w.items().add("b"); // РАБОТАЕТ! Поле final, но список мутабельный
> // Fix: public record Wrapper(List<String> items) {
> //     public Wrapper { items = List.copyOf(items); }  // defensive copy
> // }
> ```

---

## Связанные темы

- [[Java Stream API]] — использование Pattern Matching в stream-операциях
- [[Функциональное программирование с коллекциями]] — лямбды и функциональные интерфейсы
- [[Наследование]] — как Sealed-классы ограничивают иерархию
- [[Интерфейсы]] — Sealed интерфейсы
- [[Процессы и Потоки, Thread, Runnable, состояния потоков|Virtual Threads]] — Java 21 Project Loom
- [[Scoped Values (Java 21, JEP 446)]] — ThreadLocal alternative
- [[Корпоративная социальная сеть - микросервисная архитектура, выбор языка и идеи для вовлечённости|Корпоративная социальная сеть]] — применение Records в DTO-слое микросервисов