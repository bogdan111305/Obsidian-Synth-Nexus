# Приведение типов в Java (widening / narrowing)

> **Widening** (расширение) — автоматическое, без потери данных (`byte→short→int→long→float→double`). **Narrowing** (сужение) — явное `(int)d`, может потерять данные. **Upcasting** — всегда безопасно. **Downcasting** `(Dog)animal` — `ClassCastException` если объект не того типа. `instanceof` проверяет перед cast.
> На интервью: `int→float` теряет точность (widening!), `checkcast` байткод, Pattern Matching instanceof Java 16+.

## Связанные темы

[[Полиморфизм]], [[Наследование]], [[Интерфейсы]], [[Java Generics]]

---

## Примитивные типы: widening и narrowing

```
Widening (автоматически, без явного cast):
byte → short → int → long → float → double
char → int → long → float → double

Narrowing (явный cast):
double → float → long → int → short → byte/char
```

```java
// Widening — автоматически:
int i = 42;
long l = i;         // int → long: автоматически
double d = i;       // int → double: автоматически
float f = 123456789; // int → float: ТЕРЯЕТ точность! 123456789 → 1.23456792E8

// Narrowing — явный cast:
double pi = 3.14159;
int truncated = (int) pi;   // → 3 (дробная часть отбрасывается, не округляется!)
byte b = (byte) 300;        // 300 = 0x12C → byte берёт младший байт → 0x2C = 44

// char ↔ int:
char c = 'A';
int code = c;       // widening: char → int = 65
char back = (char) 65; // narrowing: int → char = 'A'
```

**Важно: `int → float` — это widening, но может терять точность!** `float` имеет только 23 бита мантиссы, `int` — 32 бита. Числа > 16 миллионов теряют точность.

```java
// Байткод опкоды преобразований:
// i2l, i2f, i2d, l2f, l2d, f2d  — widening (JVM делает автоматически)
// d2f, d2l, d2i, l2i, f2i        — narrowing (требуют явного cast)
```

---

## Ссылочные типы: upcasting и downcasting

```java
// Upcasting — всегда безопасно (неявный или явный):
Dog dog = new Dog("Rex");
Animal animal = dog;            // upcast: неявный
Animal animal2 = (Animal) dog;  // upcast: явный (избыточный cast)

// Downcasting — требует проверки, иначе ClassCastException:
Animal a = new Dog("Rex");

// Плохо:
Dog d = (Dog) a;    // OK если a действительно Dog
Cat c = (Cat) a;    // ClassCastException! a — это Dog, не Cat

// Правильно — проверка через instanceof:
if (a instanceof Dog d) {  // Pattern Matching (Java 16+)
    d.bark();               // d уже типизирован как Dog, не нужен отдельный cast
}

// Старый стиль (до Java 16):
if (a instanceof Dog) {
    Dog d2 = (Dog) a;       // дважды проверяем тип (устарело)
    d2.bark();
}
```

**Байткод downcasting:**
```
// (Dog) animal → checkcast Dog
aload_1          // загрузить animal
checkcast Dog    // если не Dog → ClassCastException
astore_2         // сохранить как Dog
```

---

## Интерфейсный cast и instanceof

```java
interface Flyable { void fly(); }
interface Swimmable { void swim(); }

class Duck extends Animal implements Flyable, Swimmable { ... }

Animal a = new Duck();

// Можно cast к интерфейсу если объект его реализует:
Flyable f = (Flyable) a;    // OK — Duck implements Flyable
Swimmable s = (Swimmable) a; // OK

// Pattern matching с несколькими типами (Java 21):
if (a instanceof Flyable f && a instanceof Swimmable s) {
    f.fly(); s.swim();
}

// switch с pattern matching (Java 21):
String result = switch (a) {
    case Duck d  -> "quack";
    case Dog d   -> "bark";
    default      -> "unknown";
};
```

---

## Числовые ловушки

```java
// Autoboxing + widening — неожиданный результат:
Integer x = 100;
Long y = (long) x;    // правильно: widening через (long)
// Long y = x;        // ОШИБКА компиляции — нет автоматического boxing widening

// == vs equals при boxing:
Integer a = 127, b = 127;
a == b;  // true — Integer кэширует -128..127 (Integer cache)
Integer c = 200, d = 200;
c == d;  // false — разные объекты (> 127 не кэшируется)
c.equals(d); // true — сравнение значений

// Потеря знака при narrowing:
int minus = -1;
byte b = (byte) minus;  // -1 → 0xFF → 0xFF как signed byte = -1 (OK в этом случае)
int big = 300;
byte b2 = (byte) big;   // 300 = 0x12C → 0x2C = 44 (потеря старших бит)
```

---

## Вопросы на интервью

- `int → float` — это widening или narrowing? Может ли потерять данные?
- Что такое `checkcast`? Когда JVM бросает `ClassCastException`?
- Чем Pattern Matching `instanceof` лучше старого `instanceof` + cast?
- Почему `Integer a = 200; Integer b = 200; a == b` — false?
- Что произойдёт при `(byte) 300`?

---

## Подводные камни

- **`int→float` теряет точность** — это widening, но числа больше 16.7M теряют точность. Используй `double` для целых больше 2^24.
- **Narrowing отбрасывает, не округляет** — `(int)3.9 = 3`, не 4. Для округления — `Math.round()`.
- **`instanceof` перед downcasting** — всегда проверяй. JVM сама не защищает — без проверки получишь `ClassCastException` в runtime.
- **Integer cache -128..127** — `Integer a == Integer b` true только в этом диапазоне. Вне диапазона — всегда `equals()`.
- **Unboxing + null → NPE** — `Integer i = null; int x = i;` → NullPointerException при unboxing.
