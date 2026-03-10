---
title: "Java — Enum (перечисления)"
tags: [java, enum, enumset, enummap]
updated: 2026-03-04
---

# Enum (перечисления) в Java

> [!QUOTE] Суть
> **Enum** — специальный класс, наследующий `java.lang.Enum`. Каждый элемент — **синглтон**, создаётся при загрузке класса. Поддерживает поля, конструкторы, методы. Thread-safe по природе. Используй вместо `int`/`String` констант для типобезопасности.

Перечисления — это классы, наследующие `java.lang.Enum`, которые ограничивают множество допустимых значений. Каждый элемент `enum` является **синглтон-экземпляром** своего класса, что гарантирует единственность.

```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY
}
```

**Использование**:
```java
Day today = Day.MONDAY;
System.out.println(today); // MONDAY
```

**Основные моменты**:
- **Типобезопасность**: Нельзя присвоить значение вне определённых констант.
- **Наследование**: Все `enum` неявно наследуют `java.lang.Object` через `java.lang.Enum`.
- **Иммутабельность**: Экземпляры неизменяемы.
- **Синглтон**: Каждый элемент — уникальный экземпляр, создаваемый при загрузке класса.
## Как компилятор реализует `enum`?

Компилятор преобразует `enum` в класс, наследующий `java.lang.Enum`. Вот пример для `enum Day`:

**Исходный код**:
```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY;
}
```

**Эквивалентный класс**:
```java
public final class Day extends java.lang.Enum<Day> {
    public static final Day MONDAY = new Day("MONDAY", 0);
    public static final Day TUESDAY = new Day("TUESDAY", 1);
    public static final Day WEDNESDAY = new Day("WEDNESDAY", 2);
    private static final Day[] VALUES = {MONDAY, TUESDAY, WEDNESDAY};

    private Day(String name, int ordinal) {
        super(name, ordinal);
    }

    public static Day[] values() {
        return VALUES.clone();
    }

    public static Day valueOf(String name) {
        return Enum.valueOf(Day.class, name);
    }
}
```

**Байт-код (упрощённый, `javap -c Day.class`)**:
```java
.class public final enum Day
.super java/lang/Enum
  .field public static final enum LDay; MONDAY
  .field public static final enum LDay; TUESDAY
  .field public static final enum LDay; WEDNESDAY
  .field private static final [LDay; VALUES
  .method static <clinit>()V
    new Day
    ldc "MONDAY"
    iconst_0
    invokespecial Day/<init>(Ljava/lang/String;I)V
    putstatic Day/MONDAY LDay;
    new Day
    ldc "TUESDAY"
    iconst_1
    invokespecial Day/<init>(Ljava/lang/String;I)V
    putstatic Day/TUESDAY LDay;
    new Day
    ldc "WEDNESDAY"
    iconst_2
    invokespecial Day/<init>(Ljava/lang/String;I)V
    putstatic Day/WEDNESDAY LDay;
    iconst_3
    anewarray Day
    dup
    iconst_0
    getstatic Day/MONDAY LDay;
    aastore
    dup
    iconst_1
    getstatic Day/TUESDAY LDay;
    aastore
    dup
    iconst_2
    getstatic Day/WEDNESDAY LDay;
    aastore
    putstatic Day/VALUES [LDay;
    return
  .end method
```

**Ключевые моменты**:
- **Статические финальные поля**: Каждый элемент (`MONDAY`, `TUESDAY`) — это `static final` экземпляр.
- **Приватный конструктор**: Вызывает `super(name, ordinal)` из `java.lang.Enum`.
- **Массив `VALUES`**: Хранит все константы, возвращается копией в `values()`.
- **Метод `valueOf`**: Использует рефлексию для поиска по имени.
## Базовый класс `java.lang.Enum`

Класс `java.lang.Enum` предоставляет базовые методы и поля для всех перечислений:

```java
public abstract class Enum<E extends Enum<E>> implements Comparable<E>, Serializable {
    private final String name;    // Имя константы (например, "MONDAY")
    private final int ordinal;    // Позиция (0, 1, 2, ...)

    protected Enum(String name, int ordinal) {
        this.name = name;
        this.ordinal = ordinal;
    }

    public final String name() {
        return name;
    }

    public final int ordinal() {
        return ordinal;
    }

    public String toString() {
        return name;
    }

    public final boolean equals(Object other) {
        return this == other;
    }

    public final int compareTo(E o) {
        return this.ordinal - o.ordinal;
    }
    // ... hashCode, clone (запрещает клонирование), и др.
}
```

**Особенности**:
- **Поля**: `name` и `ordinal` хранят имя и позицию константы.
- **Методы**:
  - `name()`: Возвращает имя константы.
  - `ordinal()`: Возвращает позицию (0-based).
  - `toString()`: По умолчанию возвращает `name`.
  - `compareTo`: Сравнивает по `ordinal`.
  - `equals`: Сравнивает по ссылке (синглтон).
- **Ограничения**:
  - `final` методы (`name`, `ordinal`, `equals`) нельзя переопределить.
  - `clone()` запрещён (`CloneNotSupportedException`).
## Синглтон-гарантия

Перечисления обеспечивают синглтон-семантику:
- **Приватный конструктор**: Запрещает создание экземпляров извне.
- **Статические финальные поля**: Инициализируются один раз при загрузке класса в Method Area.
- **Единственные экземпляры**: Все ссылки на `Day.MONDAY` указывают на один объект.

```java
Day d1 = Day.MONDAY;
Day d2 = Day.MONDAY;
System.out.println(d1 == d2); // true (один и тот же объект)
```

**JVM-гарантия**:
- Константы создаются в статическом блоке `<clinit>` при загрузке класса.
- Класс загружается один раз в метаспейсе (Metaspace).
## Добавление полей и методов

Перечисления поддерживают поля, конструкторы и методы, как обычные классы.

```java
public enum Color {
    RED("#FF0000"),
    GREEN("#00FF00"),
    BLUE("#0000FF");

    private final String hexCode;

    private Color(String hexCode) {
        this.hexCode = hexCode;
    }

    public String getHexCode() {
        return hexCode;
    }
}
```

**Байт-код (упрощённый)**:
```java
.class public final enum Color
.super java/lang/Enum
  .field private final hexCode Ljava/lang/String;
  .field public static final enum LColor; RED
  .field public static final enum LColor; GREEN
  .field public static final enum LColor; BLUE
  .method private <init>(Ljava/lang/String;Ljava/lang/String;)V
    aload_0
    aload_1
    iload_2
    invokespecial java/lang/Enum/<init>(Ljava/lang/String;I)V
    aload_0
    aload_3
    putfield Color/hexCode Ljava/lang/String;
    return
  .end method
  .method public getHexCode()Ljava/lang/String;
    aload_0
    getfield Color/hexCode Ljava/lang/String;
    areturn
  .end method
```

**Использование**:
```java
Color color = Color.RED;
System.out.println(color.getHexCode()); // #FF0000
```

**Особенности**:
- Поля инициализируются через конструктор.
- Методы доступны для каждого элемента `enum`.
## Абстрактные методы в `enum`

Каждый элемент `enum` может предоставлять свою реализацию абстрактного метода.

```java
public enum Operation {
    PLUS {
        @Override
        public double apply(double x, double y) {
            return x + y;
        }
    },
    MINUS {
        @Override
        public double apply(double x, double y) {
            return x - y;
        }
    };

    public abstract double apply(double x, double y);
}
```

**Использование**:
```java
Operation op = Operation.PLUS;
System.out.println(op.apply(5, 3)); // 8.0
```

**Байт-код (упрощённый)**:
```java
.class public abstract enum Operation
.super java/lang/Enum
  .field public static final enum LOperation; PLUS
  .field public static final enum LOperation; MINUS
  .method public abstract apply(DD)D
```
## Методы `values()` и `valueOf()`

- **`values()`**:
  - Возвращает копию массива всех констант (`VALUES.clone()`).
  - Копия защищает внутренний массив от модификаций.
- **`valueOf(String)`**:
  - Вызывает `Enum.valueOf(Class<T>, String)` через рефлексию.
  - Бросает `IllegalArgumentException`, если имя не найдено.

```java
for (Day day : Day.values()) {
    System.out.println(day.name() + ": " + day.ordinal());
}
// MONDAY: 0
// TUESDAY: 1
// WEDNESDAY: 2

Day day = Day.valueOf("MONDAY");
System.out.println(day); // MONDAY
```
## Таблица характеристик `enum`

| Что                 | Как реализовано в enum                                   |
|---------------------|---------------------------------------------------------|
| Экземпляры enum     | Статические финальные поля (синглтоны)                   |
| Конструктор         | Приватный, вызывает конструктор `Enum`                   |
| Дополнительные поля | Обычные поля, инициализируемые через конструктор         |
| Методы              | Можно определять как в обычном классе                    |
| Хранение            | В куче, как объекты, с классом, загруженным в метаспейсе |
| `values()`          | Возвращает копию массива всех элементов                  |
| `valueOf()`         | Поиск по имени через `Enum.valueOf`                      |
## Использование в конструкциях
### Switch
Перечисления идеальны для `switch`, особенно с pattern matching (Java 17+).

```java
public String describeDay(Day day) {
    return switch (day) {
        case MONDAY, TUESDAY, WEDNESDAY -> "Рабочий день";
        default -> throw new IllegalArgumentException("Неизвестный день");
    };
}
```
### Коллекции: `EnumSet` и `EnumMap`

Перечисления часто используются с `EnumSet` и `EnumMap`, которые оптимизированы для работы с `enum`.
#### `EnumSet`

`EnumSet` — это специализированное множество, использующее битовую карту для хранения элементов `enum`. Оно быстрее и компактнее, чем `HashSet`, благодаря фиксированному числу элементов `enum`.

**Особенности**:
- **Реализация**: Использует битовую карту (`long` для ≤64 элементов, `long[]` для >64).
- **Производительность**: O(1) для операций (`add`, `remove`, `contains`).
- **Память**: Очень компактно (1 бит на элемент).
- **Потокобезопасность**: Не потокобезопасен, требует синхронизации для многопоточной работы.

```java
import java.util.*;

EnumSet<Day> workdays = EnumSet.of(Day.MONDAY, Day.TUESDAY, Day.WEDNESDAY);
System.out.println(workdays); // [MONDAY, TUESDAY, WEDNESDAY]

// Добавление всех элементов
EnumSet<Day> allDays = EnumSet.allOf(Day.class);
System.out.println(allDays); // [MONDAY, TUESDAY, WEDNESDAY]

// Диапазон
EnumSet<Day> weekdays = EnumSet.range(Day.MONDAY, Day.WEDNESDAY);
System.out.println(weekdays); // [MONDAY, TUESDAY, WEDNESDAY]
```

**Методы**:
- `EnumSet.of(E e1, ...)`: Создаёт множество с указанными элементами.
- `EnumSet.allOf(Class<E>)`: Включает все элементы `enum`.
- `EnumSet.range(E from, E to)`: Включает элементы в диапазоне.
- `EnumSet.noneOf(Class<E>)`: Пустое множество.
- `add`, `remove`, `contains`: Стандартные операции множества.

**Сравнение с `HashSet`**:

| Характеристика | `EnumSet` | `HashSet` |
|----------------|-----------|-----------|
| Производительность | O(1) (битовые операции) | O(1) среднее, O(n) в худшем случае |
| Память | 1 бит на элемент | ~32–48 байт на элемент |
| Типобезопасность | Только `enum` | Любые объекты |
| Итерация | Порядок `enum` | Неопределённый порядок |

**Байт-код (упрощённый, для `EnumSet.of`)**:
```java
invokestatic java/util/EnumSet/of(Ljava/lang/Enum;)Ljava/util/EnumSet;
```
#### `EnumMap`
`EnumMap` — это специализированная карта, где ключи — элементы `enum`. Она быстрее и компактнее, чем `HashMap`, благодаря использованию массива, индексированного `ordinal`.

**Особенности**:
- **Реализация**: Массив (`Object[]`), где индекс — `ordinal` ключа.
- **Производительность**: O(1) для операций (`put`, `get`, `remove`).
- **Память**: Компактнее `HashMap` (нет узлов, только массив).
- **Потокобезопасность**: Не потокобезопасен, требует синхронизации.

```java
import java.util.*;

EnumMap<Day, String> descriptions = new EnumMap<>(Day.class);
descriptions.put(Day.MONDAY, "Начало недели");
descriptions.put(Day.TUESDAY, "Второй день");
System.out.println(descriptions.get(Day.MONDAY)); // Начало недели

// Итерация
descriptions.forEach((day, desc) -> System.out.println(day + ": " + desc));
// MONDAY: Начало недели
// TUESDAY: Второй день
```

**Методы**:
- `put(K key, V value)`: Добавляет пару ключ-значение.
- `get(Object key)`: Возвращает значение по ключу.
- `keySet()`, `values()`, `entrySet()`: Возвращают ключи, значения или пары.
- `clear()`, `remove()`: Очистка или удаление.

**Сравнение с `HashMap`**:

| Характеристика | `EnumMap` | `HashMap` |
|----------------|-----------|-----------|
| Производительность | O(1) (массив) | O(1) среднее, O(n) в худшем случае |
| Память | ~8–16 байт на запись | ~32–48 байт на запись |
| Тип ключей | Только `enum` | Любые объекты |
| Порядок итерации | Порядок `enum` | Неопределённый порядок |

**Байт-код (упрощённый, для `EnumMap.put`)**:
```java
invokevirtual java/util/EnumMap/put(Ljava/lang/Enum;Ljava/lang/Object;)Ljava/lang/Object;
```
#### Примеры использования `EnumSet` и `EnumMap`

**Фильтрация рабочих дней**:
```java
EnumSet<Day> workdays = EnumSet.of(Day.MONDAY, Day.TUESDAY, Day.WEDNESDAY);
if (workdays.contains(Day.MONDAY)) {
    System.out.println("Понедельник — рабочий день");
}
```

**Агрегация данных**:
```java
EnumMap<Day, Integer> hoursWorked = new EnumMap<>(Day.class);
hoursWorked.put(Day.MONDAY, 8);
hoursWorked.put(Day.TUESDAY, 7);
int totalHours = hoursWorked.values().stream().mapToInt(Integer::intValue).sum();
System.out.println("Всего часов: " + totalHours); // Всего часов: 15
```
#### Подводные камни коллекций

- **Потокобезопасность**: `EnumSet` и `EnumMap` не потокобезопасны. Используйте `Collections.synchronizedSet`/`Map` или `ConcurrentHashMap`.
```java
Set<Day> syncSet = Collections.synchronizedSet(EnumSet.of(Day.MONDAY));
```
- **Модификация при итерации**: Вызывает `ConcurrentModificationException`.
  - **Решение**: Используйте итераторы или стримы.
- **Ограничение ключей**: `EnumMap` принимает только `enum` в качестве ключей.
- **Пустой `EnumSet`**: Используйте `EnumSet.noneOf(Day.class)` вместо `new EnumSet<>()`.
## Внутренняя реализация в JVM

- **Хранилище**:
  - Константы: `static final` экземпляры в Metaspace.
  - Поля: Хранятся в куче (Heap) как часть объекта.
- **Инициализация**:
  - Выполняется в статическом блоке `<clinit>` при загрузке класса.
  - Гарантирует создание синглтонов.
- **Байт-код (для `Color`)**:
  ```java
  .class public final enum Color
  .super java/lang/Enum
    .field private final hexCode Ljava/lang/String;
    .method private <init>(Ljava/lang/String;Ljava/lang/String;)V
      aload_0
      aload_1
      iload_2
      invokespecial java/lang/Enum/<init>(Ljava/lang/String;I)V
      aload_0
      aload_3
      putfield Color/hexCode Ljava/lang/String;
      return
    .end method
  ```