# Enum (перечисления) в Java

> **Enum** — специальный класс, наследующий `java.lang.Enum`. Каждый элемент — **синглтон** (`public static final`), создаётся при загрузке класса. Поддерживает поля, конструкторы, методы, abstract методы. `EnumSet` хранится как битовая маска — O(1) операции.
> На интервью: как компилятор реализует enum, `EnumSet` vs `HashSet`, Singleton pattern через enum, enum в switch exhaustive.

## Связанные темы

[[Java Static]], [[Инициализация объектов с учётом наследования]], [[Современные возможности — Records и Sealed]]

---

## Как компилятор реализует enum

```java
// Исходник:
public enum Planet { MERCURY, VENUS, EARTH }

// Компилятор генерирует (упрощённо):
public final class Planet extends java.lang.Enum<Planet> {
    public static final Planet MERCURY = new Planet("MERCURY", 0);
    public static final Planet VENUS   = new Planet("VENUS", 1);
    public static final Planet EARTH   = new Planet("EARTH", 2);

    private static final Planet[] $VALUES = { MERCURY, VENUS, EARTH };

    // Приватный конструктор — нельзя создать новые экземпляры
    private Planet(String name, int ordinal) {
        super(name, ordinal);  // java.lang.Enum хранит name и ordinal
    }

    public static Planet[] values() { return $VALUES.clone(); } // клон!
    public static Planet valueOf(String name) { return Enum.valueOf(Planet.class, name); }
}
```

**`ordinal()`** — 0-based порядковый номер. Зависит от порядка объявления. **Никогда не используй ordinal() в логике** — при изменении порядка объявления ломается сериализация/БД.

---

## Enum с полями и методами

```java
public enum HttpStatus {
    OK(200, "OK"),
    NOT_FOUND(404, "Not Found"),
    INTERNAL_SERVER_ERROR(500, "Internal Server Error");

    private final int code;
    private final String message;

    HttpStatus(int code, String message) {  // конструктор всегда private/package
        this.code = code;
        this.message = message;
    }

    public int code() { return code; }
    public String message() { return message; }

    // Static factory по коду:
    public static HttpStatus fromCode(int code) {
        for (HttpStatus s : values()) {
            if (s.code == code) return s;
        }
        throw new IllegalArgumentException("Unknown code: " + code);
    }
}

HttpStatus.OK.code();          // 200
HttpStatus.fromCode(404);      // NOT_FOUND
```

---

## Enum с abstract методами

```java
public enum Operation {
    PLUS  { @Override public double apply(double x, double y) { return x + y; } },
    MINUS { @Override public double apply(double x, double y) { return x - y; } },
    TIMES { @Override public double apply(double x, double y) { return x * y; } };

    public abstract double apply(double x, double y);
}

// Каждый элемент — anonymous subclass Operation с собственной реализацией apply()
// Использование: Operation.PLUS.apply(2, 3) → 5.0
```

---

## EnumSet и EnumMap

```java
// EnumSet — битовая маска (long), максимум 64 элемента → O(1) операции
EnumSet<Day> weekdays = EnumSet.range(Day.MONDAY, Day.FRIDAY);
EnumSet<Day> weekend  = EnumSet.complementOf(weekdays);
weekdays.contains(Day.MONDAY); // O(1) — битовая операция!

// vs HashSet: EnumSet в 2-3x быстрее + меньше памяти

// EnumMap — массив индексированный по ordinal() → O(1) get/put
EnumMap<Day, String> schedule = new EnumMap<>(Day.class);
schedule.put(Day.MONDAY, "Meeting");
// vs HashMap: EnumMap в 2x быстрее + нет boxing ordinal'а
```

---

## Singleton через enum — идиоматично

```java
// Самый надёжный Singleton в Java:
public enum AppConfig {
    INSTANCE;

    private final Properties props = new Properties();

    AppConfig() {
        // инициализация при загрузке класса
        props.setProperty("db.url", "jdbc:mysql://localhost/mydb");
    }

    public String get(String key) { return props.getProperty(key); }
}

AppConfig.INSTANCE.get("db.url");

// Почему enum Singleton лучше:
// 1. JVM гарантирует однократную инициализацию (thread-safe)
// 2. Защищён от рефлексии (нельзя вызвать newInstance для enum)
// 3. Защищён от десериализации (Serializable у enum особый)
```

---

## Enum в switch

```java
// Enum + switch = exhaustive в Java 21 (без default если все case покрыты):
HttpStatus status = getStatus();
String msg = switch (status) {
    case OK              -> "Success";
    case NOT_FOUND       -> "Resource missing";
    case INTERNAL_SERVER_ERROR -> "Server error";
    // default не нужен — компилятор знает все элементы enum
};
```

---

## Вопросы на интервью

- Как компилятор реализует enum? Что хранит `java.lang.Enum`?
- Почему нельзя использовать `ordinal()` в постоянной логике?
- Почему `EnumSet` быстрее `HashSet`?
- Почему Singleton через enum лучше double-checked locking?
- Как реализовать enum с разным поведением для каждого элемента?

---

## Подводные камни

- **`ordinal()` в БД/сериализации** — при добавлении нового элемента между существующими все ordinal сдвигаются → данные в БД становятся неверными. Используй `name()` или кастомное поле `code`.
- **`values()` создаёт клон массива** — не вызывай в tight loop. Кэшируй: `private static final HttpStatus[] VALUES = values();`.
- **enum не наследуется** — `final` неявно. Нельзя `class MyStatus extends HttpStatus`.
- **enum implements** — enum может реализовывать интерфейсы. Полезно для Strategy Pattern.
- **Рефлексия не может создать enum** — `Constructor.newInstance()` для enum → `IllegalArgumentException`. Это одна из защит Singleton through enum.
