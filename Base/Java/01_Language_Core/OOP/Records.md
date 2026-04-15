# Records (Java 16+)

> Иммутабельный data-класс с автогенерацией canonical constructor, accessors (`x()`, не `getX()`), `equals`, `hashCode`, `toString`. Компилятор реализует их через `invokedynamic`, а не рефлексию — поэтому быстро.

## Связанные темы
[[Современные возможности — Records и Sealed]], [[Sealed Classes]], [[Pattern Matching]], [[Инкапсуляция]]

---

## Синтаксис и что генерирует javac

```java
public record Point(int x, int y) {}

// javac генерирует:
// public final class Point extends java.lang.Record {
//     private final int x;
//     private final int y;
//     public Point(int x, int y) { this.x=x; this.y=y; }
//     public int x() { return x; }
//     public int y() { return y; }
//     public boolean equals(Object o) { /* invokedynamic */ }
//     public int hashCode() { /* invokedynamic */ }
//     public String toString() { /* invokedynamic */ } // "Point[x=1, y=2]"
// }
```

## Compact Constructor

Позволяет добавить валидацию без повторения параметров. Присваивание полей компилятор добавляет **в конце** неявно:

```java
public record Range(int min, int max) {
    public Range {                              // нет параметров в скобках!
        if (min > max) throw new IllegalArgumentException();
        // this.min = min; this.max = max; — добавляется компилятором сюда
    }
}
```

## Кастомизация

```java
public record Wrapper(List<String> items) {
    // Defensive copy — защита от мутации:
    public Wrapper {
        items = List.copyOf(items); // заменяем ссылку до неявного присваивания
    }

    // Переопределение accessor:
    @Override
    public List<String> items() {
        return Collections.unmodifiableList(items); // ещё один способ
    }

    // Дополнительные методы:
    public int size() { return items.size(); }

    // Статические фабричные методы:
    public static Wrapper empty() { return new Wrapper(List.of()); }
}
```

## Ограничения

- Неявно `extends java.lang.Record` → нельзя `extends` другой класс
- Все компонентные поля `private final` — нельзя добавить изменяемые поля экземпляра
- Нельзя объявить `abstract`
- Можно `implements` любые интерфейсы

## Record + сериализация

При десериализации **всегда** вызывается canonical constructor (в отличие от обычных классов, где конструктор пропускается). Это делает Records безопасными для сериализации:

```java
public record Config(String host, int port) implements Serializable {
    public Config {
        Objects.requireNonNull(host);
        if (port < 1 || port > 65535) throw new IllegalArgumentException();
        // валидация выполнится и при десериализации!
    }
}
```

## Record + Jackson

Jackson 2.12+ автоматически определяет Record-конструктор. Имена JSON-полей = имена компонентов:

```java
public record UserDto(String name, int age) {}
// GET → {"name":"Alice","age":30}
// POST body {"name":"Alice","age":30} → UserDto("Alice", 30)
```

## Вопросы на интервью

- Что автогенерирует компилятор для Record?
- Чем compact constructor отличается от canonical? Где добавляется `this.x = x`?
- Почему accessor называется `x()`, а не `getX()`?
- Почему Record с `List` внутри не полностью иммутабелен? Как исправить?
- Как Records реализуют `equals`/`hashCode`? Почему через `invokedynamic`?
- Как Record ведёт себя при сериализации vs обычный класс?

## Подводные камни

- **Мутабельные компоненты** — `record Foo(List<T> list)` позволяет `foo.list().add(...)`. Defensive copy в compact constructor.
- **`getX()` vs `x()`** — accessor — именно `x()`. Lombok, MapStruct и старые фреймворки, ожидающие `getX()`, сломаются.
- **Jackson < 2.12** — нужен `@JsonProperty` или `@JsonAutoDetect(getterVisibility=...)`.
- **Compact constructor без скобок** — `public Point { }` (не `public Point() { }`). Скобки — ошибка компиляции.
- **Нельзя добавить изменяемое instance-поле** — Record не позволяет `private String mutable`.
