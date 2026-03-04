---
title: "Современные возможности Java — Records, Sealed, Pattern Matching"
tags: [java, modern-java, records, sealed-classes, pattern-matching, java17, java21]
updated: 2026-03-04
---

# Современные возможности Java (ООП)

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

Point p = new Point(1, 2);
System.out.println(p.x()); // 1
System.out.println(p);     // Point[x=1, y=2]
```

- Автоматически реализуются `equals`, `hashCode`, `toString`.
- Все поля final, класс неизменяемый.
- Можно реализовывать интерфейсы, использовать в sealed-иерархиях.

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

## Связанные темы

- [[Java Stream API]] — использование Pattern Matching в stream-операциях
- [[Функциональное программирование с коллекциями]] — лямбды и функциональные интерфейсы
- [[Наследование]] — как Sealed-классы ограничивают иерархию
- [[Интерфейсы]] — Sealed интерфейсы
- [[Процессы и Потоки, Thread, Runnable, состояния потоков|Virtual Threads]] — Java 21 Project Loom
- [[Scoped Values (Java 21, JEP 446)]] — ThreadLocal alternative
- [[Корпоративная социальная сеть - микросервисная архитектура, выбор языка и идеи для вовлечённости|Корпоративная социальная сеть]] — применение Records в DTO-слое микросервисов