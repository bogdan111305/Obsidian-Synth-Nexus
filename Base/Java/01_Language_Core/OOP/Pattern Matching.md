# Pattern Matching

> Типобезопасное сопоставление с образцом: `instanceof` (Java 16), switch-паттерны (Java 21), Record Patterns (Java 21), Unnamed Patterns `_` (Java 22). Заменяет цепочки `instanceof + cast + if`.

## Связанные темы
[[Современные возможности — Switch и Pattern Matching]], [[Records]], [[Sealed Classes]], [[Полиморфизм]]

---

> [!INFO] Подробное описание
> Полная документация со всеми видами паттернов, guard-условиями, Unnamed Patterns `_` и Primitive Patterns — в [[Современные возможности — Switch и Pattern Matching]].

---

## Pattern Matching instanceof (Java 16)

```java
// До Java 16:
if (obj instanceof String) {
    String s = (String) obj;  // дублирование
    System.out.println(s.length());
}

// Java 16+:
if (obj instanceof String s) {
    System.out.println(s.length()); // s уже String, cast не нужен
}

// В условии:
if (obj instanceof String s && s.length() > 5) { ... }

// Negation scope:
if (!(obj instanceof String s)) return;
System.out.println(s.toUpperCase()); // s доступна здесь
```

## Pattern Matching in switch (Java 21)

```java
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i < 0 -> "negative int";
        case Integer i            -> "non-negative int";
        case String s             -> "string: " + s;
        case null                 -> "null";
        default                   -> "other";
    };
}
```

**Exhaustiveness:** для sealed-иерархии `default` не нужен — компилятор проверяет полноту.

**Порядок case:** более специфичный (с `when`) — выше общего. Нарушение → compile error "dominated by previous case".

## Record Patterns (Java 21)

Деструктуризация Record прямо в паттерне:

```java
record Point(int x, int y) {}
record Circle(Point center, double radius) {}

// instanceof:
if (shape instanceof Circle(Point(int x, int y), double r)) {
    System.out.println("center=(" + x + "," + y + "), r=" + r);
}

// switch:
switch (shape) {
    case Circle(_, var r) when r > 0 -> Math.PI * r * r;
    case Circle(_, _)                -> 0;
    case Rectangle(var w, var h)     -> w * h;
}
```

## Unnamed Patterns `_` (Java 22)

```java
// Когда компонент не нужен:
case Circle(_, var r)    -> Math.PI * r * r;  // center игнорируется
case Circle(_, _)        -> 0;                 // оба компонента не нужны
```

## Вопросы на интервью

- Что такое pattern variable? Какова её область видимости при `if (!(obj instanceof String s))`?
- Почему порядок case с `when` важен? Что такое "dominated by previous case"?
- Как работает Record Pattern? Что такое деструктуризация?
- Зачем `_` в Unnamed Patterns?
- Как `sealed`-иерархия упрощает exhaustiveness check?
- Что произойдёт, если `null` передать в switch с pattern matching?

## Подводные камни

- **`null` в switch** — без `case null ->` бросает `NullPointerException`. Не забывай явную обработку.
- **Порядок case с `when`** — `case Integer i` без `when` должен быть ниже `case Integer i when i < 0`. Иначе compile error.
- **Record Pattern не матчит `null`** — `case Point(int x, int y)` не сработает для `null`. Нужен отдельный `case null`.
- **Effectively final в pattern variable** — переменную паттерна нельзя переприсвоить внутри блока.
