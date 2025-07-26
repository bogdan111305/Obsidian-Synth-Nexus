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

## Где узнать больше

- Подробнее о работе со стримами: см. [Stream API и коллекции](../3.%20Коллекции%20и%20Stream%20API/Stream%20API.md)
- Лямбда-выражения: см. [Лямбда-выражения и функциональные интерфейсы](../3.%20Коллекции%20и%20Stream%20API/Лямбда-выражения.md)
- Многопоточность: см. [Папку "Многопоточность"](../../4.%20Многопоточность/) 