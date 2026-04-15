# Value Classes и Primitive Classes

> **Project Valhalla** (JEP 401 preview Java 23, JEP 402 Java 24) вводит **Value Classes** — объекты без identity, хранимые по значению. Это позволяет JVM оптимизировать layout в памяти: избавиться от заголовков объектов и косвенной адресации, хранить данные "flat" в массивах и полях.

## Связанные темы
[[Object Identity — что меняется]], [[Valhalla и коллекции]], [[Структура памяти JVM]], [[Java Bytecode — структура и опкоды]]

---

## Проблема: "The Object Tax"

Каждый Java объект в heap имеет:
```
Object layout (HotSpot, 64-bit, compressed oops):
┌──────────────────────────────┐
│ Mark Word (8 bytes)          │ ← GC, lock, hash, age
│ Klass Pointer (4 bytes)      │ ← тип объекта
│ Padding (4 bytes)            │
│ Fields...                    │
└──────────────────────────────┘
Минимум 16 байт на объект.

Массив Point[] (1 млн точек):
  - 1M ссылок по 4 байта = 4 МБ
  - 1M объектов Point{x,y} по 16 байт = 16 МБ
  - Итого: 20 МБ + cache miss при каждом обращении
           (разыменование pointer = random access)
```

## Value Class — решение

```java
// Java 23+ (JEP 401, Preview)
value class Point {
    int x;
    int y;
    
    Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}

// JVM может хранить Point[] как плоский массив int[]int[]:
// [x0, y0, x1, y1, x2, y2, ...]  ← без заголовков, без pointer, cache-friendly
```

**Главное ограничение**: Value class **не имеет identity** (нет уникального адреса в памяти).

## Identity vs Value

| Свойство | Identity Object (обычный) | Value Object |
|----------|--------------------------|--------------|
| `==` | Сравнивает адреса | Сравнивает состояние |
| `synchronized` | Работает | Запрещено |
| `System.identityHashCode()` | Уникален | Нет смысла |
| `null` | Возможен | Нет (Value не nullable*) |
| Изменяемость | Mutable | Обычно immutable |
| Inheritance | Полное | Нет (implicitly final) |
| Интерфейсы | Да | Да |
| Memory layout | Pointer → heap object | Inlined (flat) |

*Nullable Value существует, но это отдельная концепция.

## Primitive Classes (JEP 402)

**Primitive class** — value class с ещё более ограниченной семантикой, позволяющий JVM обращаться с ним как с примитивом:

```java
primitive class Complex {
    double real;
    double imaginary;
    
    Complex(double real, double imaginary) {
        this.real = real;
        this.imaginary = imaginary;
    }
    
    Complex add(Complex other) {
        return new Complex(real + other.real, imaginary + other.imaginary);
    }
}

// JVM может хранить Complex на стеке без heap allocation
// Возврат Complex из метода = два double в регистрах
Complex a = new Complex(1.0, 2.0);
Complex b = new Complex(3.0, 4.0);
Complex c = a.add(b);  // ← zero heap allocation, zero GC pressure
```

## Текущий статус (2026)

| JEP | Содержание | Статус |
|-----|-----------|--------|
| JEP 401 | Value Classes and Objects (Preview) | Java 23+ Preview |
| JEP 402 | Primitive Types in Patterns, instanceof, switch | Java 23+ Preview |
| JEP 169 | Value Objects (исторический) | Withdrawn, заменён 401 |

Для использования: `--enable-preview -source 23`

## Пример: до и после Valhalla

```java
// ДО Valhalla: wrapper классы = boxing overhead
List<Integer> numbers = new ArrayList<>();
numbers.add(42);  // boxing: int → Integer → heap object

// ПОСЛЕ Valhalla: value class Integer без boxing
// (в будущем, когда Integer станет value class)
List<Integer> numbers = new ArrayList<>();
numbers.add(42);  // JVM хранит int напрямую в массиве ArrayList

// Реальный пример уже сейчас:
record Point(int x, int y) {}  // record близок к value, но с identity

// Value class (preview):
value record Point(int x, int y) {}  // или value class
```

## Совместимость с существующим кодом

```java
// Value class реализует интерфейсы — можно использовать как обычный объект
value class Money implements Comparable<Money> {
    long cents;
    
    @Override
    public int compareTo(Money other) {
        return Long.compare(this.cents, other.cents);
    }
}

// Работает с generics (пока с boxing для примитивных специализаций):
List<Money> prices = new ArrayList<>();
prices.add(new Money(100));
```

## JVM инструкции для Value Types

Новые opcode'ы (планируются):
- `vload`, `vstore` — загрузка/хранение value
- `vreturn` — возврат value из метода
- `vaload`, `vastore` — доступ к элементам value array

Существующий `aload`/`astore` остаётся для reference types.

---

## Вопросы на интервью
- В чём проблема "Object Tax" в Java и как Valhalla её решает?
- Что такое Object Identity и почему Value Class её не имеет?
- Чем Value Class отличается от Primitive Class?
- Почему `synchronized` запрещён на Value Class?
- Как изменится layout памяти для массива Value объектов?

## Подводные камни
- Value class не может быть `null` по умолчанию — нужен специальный nullable wrapper или Optional<ValueClass>
- `==` для value class сравнивает поля, а не адреса — семантически как `equals()` у record. Это меняет поведение при использовании как ключей HashMap (до полной поддержки)
- Нет наследования value classes от других классов (только интерфейсы) — ограничение дизайна
- На 2026 год это Preview feature — нельзя использовать в production без `--enable-preview`
- JVM-оптимизация "scalarization" работает и сейчас для short-lived объектов через Escape Analysis — Valhalla расширяет это на long-lived и array элементы
