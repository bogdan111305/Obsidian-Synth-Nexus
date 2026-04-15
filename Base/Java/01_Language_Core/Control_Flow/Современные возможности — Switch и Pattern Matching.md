# Switch Expressions и Pattern Matching

> Switch Expressions (Java 14+) — выражение вместо инструкции, без fall-through. Pattern Matching (Java 16+) — типобезопасный downcasting без явного cast. Pattern Matching in switch (Java 21) — полная замена цепочек `instanceof + cast`.

## Связанные темы
[[Полиморфизм]], [[Приведение типов, widening conversion, narrowing conversion]], [[Современные возможности — Records и Sealed]], [[Enum]], [[Pattern Matching]], [[Records]], [[Sealed Classes]]

---

## Switch Expressions (Java 14, финал)

До Java 14: `switch` — инструкция с fall-through и обязательным `break`.

```java
// Старый стиль — легко забыть break, всегда инструкция
String day;
switch (num) {
    case 1:
        day = "Mon";
        break;
    case 2:
        day = "Tue";
        break;   // пропустишь — fall-through
    default:
        day = "Other";
}
```

**Switch expression — стрелочный синтаксис (Java 14+):**

```java
// switch как выражение, присваивается напрямую
String day = switch (num) {
    case 1 -> "Mon";
    case 2 -> "Tue";
    case 3, 4, 5 -> "Mid-week"; // несколько значений в одном case
    default -> "Other";
};
// fall-through невозможен: -> означает "и всё"
```

**yield** — возврат значения из блочного case:

```java
String result = switch (status) {
    case 200 -> "OK";
    case 404 -> "Not Found";
    default -> {
        // блок кода — нужен yield, не return
        String msg = "HTTP " + status;
        yield msg.toLowerCase();
    }
};
```

> [!INFO] `yield` vs `return`
> `yield` работает только внутри switch expression. `return` возвращает из метода — нельзя использовать для возврата значения из switch.

**Exhaustiveness:** компилятор требует покрыть все случаи. Для `int` — `default` обязателен. Для `enum` — можно без `default`, если покрыты все константы (компилятор проверит).

---

## Pattern Matching instanceof (Java 16, финал)

**Проблема:** классический `instanceof` требует явного cast:

```java
// Старый код — шум из cast
if (obj instanceof String) {
    String s = (String) obj; // дублирование
    System.out.println(s.length());
}
```

**Pattern Matching instanceof** совмещает проверку и привязку:

```java
if (obj instanceof String s) {
    // s уже типа String, cast не нужен
    System.out.println(s.length());
}

// В условии можно использовать сразу:
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s.toUpperCase());
}
```

**Область видимости переменной паттерна** определяется логикой, а не блоками:

```java
if (!(obj instanceof String s)) {
    return; // если не String — выходим
}
// s доступна здесь: компилятор знает, что obj — это String
System.out.println(s.toUpperCase());
```

---

## Pattern Matching in switch (Java 21, финал)

Расширяет switch до работы с типами (раньше только константы/enum).

```java
static String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "int: " + i;
        case Long l    -> "long: " + l;
        case String s  -> "string: " + s;
        case null      -> "null";       // явная обработка null
        default        -> "other";
    };
}
```

**Exhaustiveness:** если selector типа `sealed`-иерархии — компилятор требует покрыть все подтипы. Если `Object` — нужен `default`.

**Порядок case имеет значение:** более специфичные паттерны должны быть выше:

```java
// ПРАВИЛЬНО: специфичный guard — первым
switch (obj) {
    case Integer i when i < 0 -> "negative";
    case Integer i            -> "non-negative int"; // catch-all для Integer
    default                   -> "other";
}

// ОШИБКА компиляции: первый case доминирует над вторым
switch (obj) {
    case Integer i            -> "int";
    case Integer i when i < 0 -> "negative"; // недостижимо — compile error
}
```

---

## Guard Patterns — `when` (Java 21)

`when` добавляет условие к паттерну:

```java
static String classify(Number n) {
    return switch (n) {
        case Integer i when i < 0    -> "negative int";
        case Integer i when i == 0   -> "zero";
        case Integer i               -> "positive int";
        case Double d when d.isNaN() -> "NaN double";
        case Double d                -> "double: " + d;
        default                      -> "other number";
    };
}
```

**Record Patterns + guard:**

```java
record Point(int x, int y) {}

static String quadrant(Object obj) {
    return switch (obj) {
        case Point(int x, int y) when x > 0 && y > 0 -> "Q1";
        case Point(int x, int y) when x < 0 && y > 0 -> "Q2";
        case Point(int x, int y) when x < 0 && y < 0 -> "Q3";
        case Point(int x, int y) when x > 0 && y < 0 -> "Q4";
        case Point(_, _)                              -> "На оси";
        default                                       -> "не точка";
    };
}
```

---

## Record Patterns (Java 21, финал)

Деструктуризация Record прямо в паттерне:

```java
record Circle(Point center, double radius) {}
record Rectangle(double width, double height) {}
sealed interface Shape permits Circle, Rectangle {}

static double area(Shape shape) {
    return switch (shape) {
        // деструктурируем Circle, извлекаем radius
        case Circle(_, var r) when r > 0 -> Math.PI * r * r;
        case Circle(_, var r)            -> throw new IllegalArgumentException("r=" + r);
        case Rectangle(var w, var h)     -> w * h;
    };
}
```

**Вложенные Record Patterns:**

```java
record Address(String city, String country) {}
record Person(String name, Address address) {}

if (obj instanceof Person(var name, Address(var city, _))) {
    System.out.println(name + " живёт в " + city);
}
```

---

## Unnamed Patterns `_` (Java 22, финал)

`_` вместо переменной, когда значение не нужно:

```java
// Без unnamed patterns — нужно придумать имя
case Circle(Point center, var r) -> Math.PI * r * r; // center не используется

// С unnamed patterns — явно показываем "не нужно"
case Circle(_, var r)    -> Math.PI * r * r;
case Circle(_, _)        -> 0;   // ничего не нужно
case Rectangle(var w, _) -> w;   // только ширина
```

**В instanceof:**

```java
if (obj instanceof Point(int x, _)) {
    // только x нужен
}
```

---

## Primitive Types in Patterns (Java 23 Preview → Java 25 финал)

JEP 455: примитивные типы в паттернах `instanceof` и `switch`:

```java
// Java 25: примитивы напрямую в switch (Object-selector)
static String describe(Object obj) {
    return switch (obj) {
        case int i    when i < 0  -> "negative int: " + i;
        case int i                -> "non-negative int: " + i;
        case long l               -> "long: " + l;
        case double d             -> "double: " + d;
        case String s             -> "string: " + s;
        default                   -> "other";
    };
}

// В instanceof:
if (obj instanceof int i) {
    System.out.println("auto-unboxed int: " + i);
}
```

> [!WARNING] Статус
> Preview в Java 23–24. Финализация в Java 25 (сентябрь 2025). До финализации требует `--enable-preview`.

---

## Вопросы на интервью

- Чем Switch Expression отличается от Switch Statement? Что такое `yield`?
- Что такое fall-through и как Switch Expression его устраняет?
- Как работает Pattern Matching `instanceof` — что такое "pattern variable" и какова её область видимости?
- Чем pattern matching в `switch` отличается от `if-instanceof`-цепочки? Когда компилятор требует `default`?
- Что такое guard pattern (`when`)? Почему порядок case с `when` важен?
- Что такое Record Pattern? Как работает деструктуризация вложенных Records?
- Зачем нужен `_` в Unnamed Patterns? В чём отличие от `var _`?
- Как `sealed`-иерархия взаимодействует с exhaustiveness check в switch?
- Что нового в Primitive Types in Patterns (JEP 455)?

## Подводные камни

- **Порядок case:** более специфичный паттерн с `when` обязан быть выше общего — иначе compile error "dominated by previous case".
- **`null` в switch:** по умолчанию `switch` бросает `NullPointerException` на `null`-значение. Явный `case null ->` нужен, если `null` допустим.
- **`yield` vs `return`:** в switch expression нужен `yield`, `return` вернёт из метода, а не из switch.
- **Exhaustiveness при добавлении sealed-подтипа:** если к sealed-иерархии добавить новый permit — все switch по ней без `default` перестанут компилироваться. Это фича, а не баг: вынуждает обновить обработку.
- **Record Pattern + null:** `case Point(int x, int y)` не матчится на `null` — паттерн деструктурирует ненулевой объект. `null` нужно обрабатывать отдельным `case null`.
- **Primitive Patterns требуют `--enable-preview`** до Java 25 — нельзя использовать в production без флага.
