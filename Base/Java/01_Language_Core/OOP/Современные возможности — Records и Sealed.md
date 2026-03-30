# Records и Sealed Classes

> **Records** (Java 16+) — иммутабельные data-классы с автогенерацией equals/hashCode/toString/getters. **Sealed classes** (Java 17+) — закрытая иерархия (`permits`), исчерпывающий `switch`.

## Связанные темы

[[Инкапсуляция]], [[Наследование]], [[Полиморфизм]], [[Современные возможности — Switch и Pattern Matching]]

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
- Компилятор может проверить исчерпывающий `switch` без `default`

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

---

## Вопросы на интервью

- Что автоматически генерирует компилятор для Record? Почему `x()`, а не `getX()`?
- Что такое compact constructor? Где неявно добавляется присваивание полей?
- Почему Record нельзя унаследовать? Какой класс он неявно расширяет?
- Как Records реализуют `equals`/`hashCode`? Что такое `invokedynamic` в этом контексте?
- Почему Record с `List` внутри не полностью иммутабелен? Как это исправить?
- Как Record ведёт себя при сериализации? Чем отличается от обычного класса?
- Что такое Record Pattern? Как работает деструктуризация вложенных Records?
- Что означают `final`, `sealed`, `non-sealed` для permits-классов?
- Почему `sealed` позволяет убрать `default` из switch?
- Как sealed-иерархия влияет на JIT (Class Hierarchy Analysis)?

## Подводные камни

- **Record не полностью иммутабелен** — если компонент мутабельный (`List`, массив), его можно изменить. Defensive copy в compact constructor: `items = List.copyOf(items)`.
- **`x()`, не `getX()`** — accessor-методы Records называются по имени компонента без `get`. Jackson до 2.12 не умел это автоматически — нужен `@JsonProperty` или `@JsonAutoDetect`.
- **Compact constructor не повторяет параметры** — `public Point { ... }` (без `(int x, int y)`). `this.x = x` добавляется компилятором **в конец** compact constructor неявно.
- **Record нельзя `extends`** — `public record Foo(...) extends Bar` — compile error. Только `implements`.
- **`sealed` требует `permits` в том же пакете/модуле** — нельзя разнести реализации по разным модулям без `opens`.
- **`non-sealed` открывает иерархию** — любой класс может наследовать `non-sealed`-класс, даже если родитель `sealed`. Это намеренный escape hatch.
- **Добавление нового permits-класса ломает switch без `default`** — compile error. Это фича: принудительное обновление логики обработки.

## Связанные темы

- [[Современные возможности — Switch и Pattern Matching]] — Pattern Matching instanceof и switch
- [[Наследование]] — как Sealed-классы ограничивают иерархию
- [[Интерфейсы]] — Sealed интерфейсы
- [[Records]], [[Sealed Classes]], [[Pattern Matching]]
