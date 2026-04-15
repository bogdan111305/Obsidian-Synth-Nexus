# var и Text Blocks

> **`var`** (Java 10+) — локальный вывод типов, компилятор определяет тип из правой части присваивания. Только для локальных переменных. **Text Blocks** (Java 15+) — многострочные строки без escape-символов.

## Связанные темы

[[Java String]], [[Java Syntax Reference]]

---

## var — локальный вывод типов (Java 10+)

`var` позволяет компилятору вывести тип локальной переменной автоматически:

```java
var list = new ArrayList<String>();     // List<String>
var map = new HashMap<String, Integer>(); // Map<String, Integer>
var name = "Alice";                     // String
var count = 42;                         // int

// В for-each:
for (var entry : map.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
}

// В try-with-resources:
try (var stream = Files.lines(Path.of("file.txt"))) {
    stream.forEach(System.out::println);
}
```

**Ограничения `var`:**
- Только **локальные переменные** — нельзя для полей, параметров методов, возвращаемых типов
- Обязательная инициализация при объявлении (компилятор должен вывести тип)
- `var x = null;` — запрещено (тип неопределён)
- `var` — не ключевое слово, а reserved type name: можно использовать `var` как имя переменной (но не стоит)

---

## Text Blocks (Java 15+)

Text Blocks — многострочные строковые литералы без escape-последовательностей:

```java
// Старый способ:
String json = "{\n    \"name\": \"Alice\",\n    \"age\": 30\n}";

// Text Block (Java 15+):
String json = """
        {
            "name": "Alice",
            "age": 30
        }
        """;

// SQL:
String sql = """
        SELECT u.name, u.email
        FROM users u
        WHERE u.active = true
          AND u.age > 18
        ORDER BY u.name
        """;

// HTML:
String html = """
        <html>
            <body>
                <p>Hello, %s!</p>
            </body>
        </html>
        """.formatted("World");
```

**Специальные escape-последовательности в Text Blocks:**
- `\` в конце строки — убрать перевод строки (line continuation)
- `\s` — пробел (для сохранения trailing spaces)

```java
// Без \s — trailing spaces обрезаются:
String s = """
        Alice
        Bob
        """;

// С \s — trailing spaces сохранятся до позиции \s:
String s = """
        Alice  \s
        Bob    \s
        """;
```

---

## Unnamed Patterns и Unnamed Variables (Java 22+)

**Unnamed Pattern** (`_`) в `switch` и `instanceof` — игнорируем переменную без имени:

```java
// Java 22: Unnamed Pattern Variable (JEP 456)
// _ вместо именованной переменной, если значение не нужно
if (obj instanceof Point(var x, _)) {          // игнорируем y
    System.out.println("x=" + x);
}

// Unnamed Variable в блоках кода (не только patterns):
try {
    riskyOperation();
} catch (Exception _) {      // исключение поймано, но не используется
    log("operation failed");
}

for (var _ : list) {         // итерируем N раз, элемент не нужен
    counter++;
}
```

> [!INFO] Зачем `_`?
> До Java 22 нужно было придумывать имя (`ignored`, `unused`, `e`). Теперь компилятор явно знает "эта переменная намеренно игнорируется" — улучшает читаемость и позволяет JIT более агрессивно оптимизировать.

---

## Вопросы на интервью

- Что такое `var`? Является ли он ключевым словом?
- Где можно и нельзя использовать `var`? (поля, параметры, возвращаемые типы)
- Что происходит с типом при `var x = new ArrayList<>()`? Какой тип выводит компилятор?
- Чем Text Block отличается от обычного `String`? Как определяется отступ?
- Что делает `\` в конце строки Text Block? Что делает `\s`?
- Что такое `_` в unnamed patterns и unnamed variables? Зачем это нужно?

## Подводные камни

- **`var` не динамический тип** — тип выводится на этапе компиляции и фиксируется. Это не Python/JavaScript — просто синтаксический сахар.
- **`var x = null`** — compile error: компилятор не может вывести тип из `null`.
- **`var` скрывает тип** — `var list = someMethod()` скрывает реальный тип. Если метод возвращает `LinkedList`, а ты ожидаешь `List` — потеряешь контракт. Применяй `var` только когда тип очевиден из правой части.
- **Отступ Text Block** — определяется по наименее отступленной строке (включая закрывающий `"""`). Сдвинешь закрывающий `"""` — весь блок сдвинется.
- **Text Block — compile-time константа** — Text Block превращается в обычный `String` на этапе компиляции. Никакого overhead в runtime.
- **Trailing whitespace в Text Block** — обрезается автоматически. Для сохранения нужен `\s` в конце строки.
- **`var` в лямбде нельзя** — `(var x) -> x.length()` недопустимо (но `(String x) -> x.length()` — можно). Исключение: аннотированные параметры `(@NonNull var x) -> ...` — здесь `var` разрешён.

## Связанные темы

- [[Современные возможности — Switch и Pattern Matching]] — Pattern Matching instanceof и switch
- [[Java String]] — методы работы со строками
