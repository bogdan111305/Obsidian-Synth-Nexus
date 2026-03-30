# Поля, конструкторы, this, инициализаторы

> **Класс** — шаблон с полями (состояние) и методами (поведение). **Объект** — экземпляр в heap. Конструктор без параметров генерируется компилятором **только если не объявлен явный**. `this(...)` — делегирование между конструкторами. Instance initializer block компилируется в начало каждого конструктора.
> На интервью: порядок инициализации полей, `this(...)` delegation chain, final поля в конструкторе, blank final.

## Связанные темы

[[Инициализация объектов с учётом наследования]], [[Наследование]], [[Java Static]], [[Инкапсуляция]]

---

## Поля: экземплярные vs статические

```java
public class Person {
    // Экземплярные поля — уникальны для каждого объекта (Heap)
    private String name;
    private int age;

    // Статическое поле — одно на весь класс (Metaspace)
    private static int count = 0;

    // Константа
    public static final String SPECIES = "Homo sapiens";

    // Значения по умолчанию (без явной инициализации):
    // int, long, short, byte → 0
    // double, float → 0.0
    // boolean → false
    // Object → null
    // char → '\u0000'
}
```

---

## Конструкторы

```java
public class Person {
    private final String name;  // blank final — должен быть установлен в конструкторе
    private final int age;

    // Основной конструктор:
    public Person(String name, int age) {
        if (name == null) throw new NullPointerException("name");
        if (age < 0) throw new IllegalArgumentException("age < 0");
        this.name = name;
        this.age  = age;
    }

    // Делегирование: this(...) ПЕРВОЙ строкой (как super()):
    public Person(String name) {
        this(name, 0);  // вызывает Person(String, int)
    }

    // Если не определить ни один конструктор — компилятор добавит:
    // public Person() {}
    // Но если определить хоть один — дефолтный больше NOT генерируется!
}
```

**Blank final поля** — `private final int x;` — компилятор требует установить в каждом конструкторе. Если хотя бы один конструктор не устанавливает → ошибка компиляции.

---

## Instance Initializer Block

```java
public class Connection {
    private final String url;
    private final long createdAt;
    private static int totalCreated;

    // Instance initializer — выполняется в начале КАЖДОГО конструктора:
    {
        createdAt = System.currentTimeMillis();
        totalCreated++;  // счётчик для всех конструкторов
    }

    // Static initializer — один раз при загрузке класса:
    static {
        System.out.println("Connection class loaded");
    }

    public Connection(String url) { this.url = url; }
    public Connection() { this.url = "localhost"; }
    // Оба конструктора получат корректный createdAt — из instance initializer
}
```

**Байткод:** instance initializer компилятор вставляет в начало каждого конструктора (после `super()`). Это просто синтаксический сахар, никакого runtime overhead.

---

## this — ключевое слово

```java
public class Builder {
    private String name;
    private int timeout;

    // this — ссылка на текущий объект:
    public Builder setName(String name) {
        this.name = name;      // this.name — поле, name — параметр
        return this;           // для fluent API (method chaining)
    }

    public Builder setTimeout(int timeout) {
        this.timeout = timeout;
        return this;
    }

    // this.getClass() — тип текущего объекта (для factory методов):
    public Builder copy() {
        try {
            return this.getClass().getDeclaredConstructor().newInstance();
        } catch (Exception e) { throw new RuntimeException(e); }
    }
}

// Fluent API благодаря return this:
new Builder().setName("foo").setTimeout(30);
```

---

## Байткод: new + <init>

```
// Person p = new Person("Alice", 25):
new Person                  // выделить память, все поля = default
dup                         // дублировать ссылку (одна для invokespecial, одна для astore)
ldc "Alice"                 // загрузить "Alice"
bipush 25                   // загрузить 25
invokespecial Person.<init>(Ljava/lang/String;I)V  // вызвать конструктор
astore_1                    // сохранить в переменную p
```

`new` только выделяет память и устанавливает default значения. `<init>` выполняет инициализацию.

---

## Вопросы на интервью

- Когда компилятор НЕ генерирует дефолтный конструктор?
- Что такое blank final? Что будет если не инициализировать его в конструкторе?
- `this(...)` — ограничения использования?
- Где в байткоде оказывается instance initializer block?
- Чем `new Person()` отличается от `Person.class.newInstance()` (через рефлексию)?

---

## Подводные камни

- **Добавил параметризованный конструктор → пропал дефолтный** — если подкласс не вызывает `super(args)`, ошибка компиляции. Всегда думай о совместимости с подклассами.
- **`this(...)` должен быть первой строкой** — нельзя до него использовать `this` или вызывать что-либо.
- **Circular constructor delegation** — `A()` вызывает `this(1)`, который вызывает `this()` → `StackOverflowError`. Компилятор не проверяет циклические делегации.
- **Instance initializer порядок** — если полей несколько и они зависят друг от друга, порядок объявления имеет значение: инициализируются сверху вниз.
- **final поле и рефлексия** — `field.setAccessible(true); field.set(obj, value)` обходит `final`. Использовать только в тестах/фреймворках.
