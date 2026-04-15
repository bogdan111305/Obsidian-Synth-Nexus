# Java Syntax Reference

> Справочник по базовому синтаксису Java: примитивные типы, переменные, массивы, управляющие конструкции. Фокус — на нюансах, которые проверяют на Senior-интервью: autoboxing-кэш, IEEE 754, tableswitch vs lookupswitch, хвостовая рекурсия.

## Связанные темы
[[Приведение типов, widening conversion, narrowing conversion]], [[Java Memory Structure]], [[Модель памяти Java (JMM) и барьеры памяти]], [[Java String]], [[Java Stream API & Functional Programming]]

---

## Примитивные типы

| Тип | Размер | Диапазон | Default |
|---|---|---|---|
| `byte` | 8 бит | -128 .. 127 | 0 |
| `short` | 16 бит | -32 768 .. 32 767 | 0 |
| `int` | 32 бит | -2³¹ .. 2³¹-1 | 0 |
| `long` | 64 бит | -2⁶³ .. 2⁶³-1 | 0L |
| `float` | 32 бит | IEEE 754, ~7 знаков | 0.0f |
| `double` | 64 бит | IEEE 754, ~15 знаков | 0.0d |
| `char` | 16 бит | 0 .. 65 535 (unsigned!) | '\u0000' |
| `boolean` | JVM-зависим | true / false | false |

**Размеры фиксированы** — не зависят от платформы (в отличие от C/C++).

**Two's complement:** отрицательные числа хранятся как инверсия + 1. Следствие: переполнение не исключение, а молчаливый wrap-around:

```java
byte b = 127;
b++;           // -128, не ошибка!
int max = Integer.MAX_VALUE;
max++;         // Integer.MIN_VALUE
```

**IEEE 754 — ловушка float/double:**

```java
System.out.println(0.1 + 0.2 == 0.3); // false — погрешность представления
System.out.println(0.1 + 0.2);        // 0.30000000000000004

// Правильное сравнение:
Math.abs((0.1 + 0.2) - 0.3) < 1e-9   // true

// Для точных финансовых расчётов:
new BigDecimal("0.1").add(new BigDecimal("0.2")); // 0.3 точно
// BigDecimal("0.1"), не BigDecimal(0.1) — конструктор от double уже неточен!
```

**`char` — единственный беззнаковый тип в Java.** Может неявно конвертироваться в `int`:

```java
char c = 'A';
int i = c;     // 65 — widening без потерь
c = (char) 66; // 'B' — narrowing, нужен явный cast
```

---

## Autoboxing и Integer Cache

Autoboxing: примитив ↔ wrapper, выполняется через `Type.valueOf()` / `.typeValue()`.

```java
Integer a = 127;  // Integer.valueOf(127) → из кэша
Integer b = 127;
System.out.println(a == b);      // true  — один объект из кэша!

Integer c = 128;  // новый объект в heap
Integer d = 128;
System.out.println(c == d);      // false — разные объекты
System.out.println(c.equals(d)); // true  — всегда используй equals для wrapper
```

**Кэшируются:** `Integer`, `Byte`, `Short`, `Long` — диапазон `-128..127`. `Character` — `0..127`. `Boolean` — оба значения.

**Ловушки autoboxing:**

```java
// NPE при анбоксинге:
Integer x = null;
int y = x;  // NullPointerException в runtime!

// Производительность в hot path:
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(i); // autoboxing на каждой итерации — 1M allocation!
}
// Лучше: IntStream, int[], или примитивные коллекции (Eclipse Collections, Trove)
```

---

## Переменные

### Виды переменных и хранение

| Вид | Где хранится | Default | Время жизни |
|---|---|---|---|
| Локальная | Stack потока | Нет (compile error без init) | До выхода из блока |
| Поле экземпляра | Heap (внутри объекта) | 0 / null / false | Пока жив объект |
| Статическое поле | Metaspace | 0 / null / false | Пока загружен класс |

### Effectively Final

Переменная, которая не меняется после инициализации (даже без `final`). Лямбды и анонимные классы могут захватывать только effectively final переменные:

```java
String msg = "hello";
// msg = "bye"; // если раскомментировать — ошибка компиляции ниже
Runnable r = () -> System.out.println(msg); // OK: effectively final
```

**Почему:** лямбда может выполниться в другом потоке позже. Изменяемая переменная создала бы гонку.

### Shadowing

```java
class Foo {
    int x = 10;

    void method(int x) {
        System.out.println(x);      // параметр — shadowing поля
        System.out.println(this.x); // явный доступ к полю
    }

    void bad() {
        int i = 5;
        for (int i = 0; i < 10; i++) {} // compile error: i уже объявлена
    }
}
```

### Модификаторы

- **`final`** — ссылку нельзя переприсвоить, но содержимое объекта изменить можно
- **`volatile`** — видимость между потоками, без атомарности → см. [[Атомарность операций и Volatile]]
- **`transient`** — поле исключается из Java-сериализации

---

## Массивы

```java
int[] a = new int[5];       // элементы = 0
int[] b = {10, 20, 30};     // инициализатор
String[] s = new String[3]; // элементы = null

// Массив — объект в heap, переменная — ссылка в stack
int len = b.length; // поле, не метод!
```

**Многомерные и jagged:**

```java
int[][] matrix = new int[3][4];  // прямоугольный
int[][] jagged = new int[3][];   // строки разной длины
jagged[0] = new int[2];
jagged[1] = new int[5];
```

**`Arrays` API:**

```java
Arrays.sort(arr);                       // dual-pivot quicksort для примитивов, timsort для объектов
Arrays.binarySearch(sortedArr, key);    // O(log n), массив должен быть отсортирован!
Arrays.copyOf(arr, newLen);             // копия с новой длиной (trim или pad нулями)
Arrays.copyOfRange(arr, from, to);      // срез
Arrays.fill(arr, value);
Arrays.equals(a, b);                    // поэлементное сравнение
Arrays.deepEquals(a2d, b2d);           // для многомерных
Arrays.toString(arr);                   // "[1, 2, 3]"
```

**Shallow copy — ловушка:**

```java
String[] orig = {"a", "b"};
String[] copy = orig.clone(); // копируются ссылки, не объекты
// copy[0] и orig[0] — один и тот же объект String

Person[] people = {new Person("Alice")};
Person[] cloned = people.clone();
cloned[0].setName("Bob"); // меняет тот же объект — orig тоже изменился!
```

**Array covariance — ловушка:**

```java
String[] strings = new String[3];
Object[] objects = strings;       // compile OK — arrays are covariant
objects[0] = 42;                  // compile OK, но ArrayStoreException в runtime!
// Обобщения не ковариантны: List<String> → List<Object> — compile error (это лучше)
```

---

## Управляющие конструкции

### if-else, тернарный оператор

```java
// Short-circuit evaluation: правая часть не вычисляется, если результат известен
if (obj != null && obj.isValid()) { ... } // безопасно: если obj==null, isValid() не вызовется

// Тернарный — только для простых выражений:
int max = (a > b) ? a : b;
// Не вкладывай тернарные: нечитаемо
```

### switch — tableswitch vs lookupswitch

Компилятор выбирает инструкцию байткода автоматически:
- **`tableswitch`** — значения последовательные или плотные → O(1), jump table
- **`lookupswitch`** — значения разрозненные → O(log n), binary search

```java
// tableswitch (1,2,3 — последовательно):
switch (x) { case 1: ... case 2: ... case 3: ... }

// lookupswitch (1, 100, 10000 — разрозненно):
switch (x) { case 1: ... case 100: ... case 10000: ... }
```

**Fall-through — классическая ловушка:**

```java
switch (day) {
    case 1:
        System.out.println("Mon");
        // забыли break — выполнится case 2 тоже!
    case 2:
        System.out.println("Tue");
        break;
}
// day=1 → "Mon" + "Tue"
```

Стрелочный синтаксис `->` (Java 14+) fall-through невозможен по дизайну.

### Циклы

```java
// for-each компилируется в iterator-вызовы для Iterable:
for (String s : list) { ... }
// эквивалент:
Iterator<String> it = list.iterator();
while (it.hasNext()) { String s = it.next(); ... }

// Поэтому: нельзя удалять элементы через for-each (ConcurrentModificationException)
// Для удаления: it.remove() через явный Iterator, или removeIf()
```

**break/continue с меткой — для вложенных циклов:**

```java
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 1) break outer; // выходим из внешнего цикла
    }
}
```

### Рекурсия

```java
// Java НЕ оптимизирует хвостовую рекурсию (в отличие от Scala/Kotlin)
// Глубокая рекурсия → StackOverflowError (стек ~512KB по умолчанию)
public long factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1); // хвостовая — но JVM не оптимизирует!
}

// Решение для глубоких структур: итерация с явным Stack<>:
Deque<Node> stack = new ArrayDeque<>();
stack.push(root);
while (!stack.isEmpty()) {
    Node node = stack.pop();
    // обработка...
    if (node.right != null) stack.push(node.right);
    if (node.left != null) stack.push(node.left);
}
```

---

## Вопросы на интервью

- Почему `0.1 + 0.2 != 0.3`? Как правильно сравнивать double?
- Что такое Integer Cache? В каком диапазоне работает? Почему `==` ненадёжен для Integer?
- Что происходит при переполнении `int`? Это исключение?
- Чем `char` отличается от других числовых примитивов? Почему он беззнаковый?
- Где хранятся локальные переменные, поля объекта, статические поля?
- Что такое effectively final? Почему лямбды не могут захватывать изменяемые переменные?
- Что такое array covariance? Почему это считается дефектом языка?
- Чем `Arrays.sort` для примитивов отличается от `Arrays.sort` для объектов?
- Что такое `tableswitch` vs `lookupswitch`? Как компилятор выбирает между ними?
- Почему нельзя удалять элементы в for-each? Как правильно удалять во время итерации?
- Почему Java не оптимизирует хвостовую рекурсию? Как заменить глубокую рекурсию?
- Что произойдёт при анбоксинге `null`? Где это чаще всего встречается?

## Подводные камни

- **`==` для wrapper-типов** — сравнивает ссылки. `Integer a = 1000; Integer b = 1000; a == b` → `false`. Всегда `.equals()`.
- **Autoboxing NPE** — `Integer x = null; int y = x;` → `NullPointerException` в runtime. Компилятор не предупреждает.
- **`BigDecimal(double)`** — `new BigDecimal(0.1)` ≠ `0.1` из-за неточного double. Всегда `new BigDecimal("0.1")`.
- **`byte` overflow** — `(byte)128 == -128`. При explicit cast JVM не проверяет диапазон.
- **Array covariance** — `Object[] a = new String[3]; a[0] = 42;` компилируется, но даёт `ArrayStoreException` в runtime.
- **`Arrays.binarySearch` без сортировки** — результат undefined. Сначала `Arrays.sort`, потом `binarySearch`.
- **clone массива объектов** — shallow copy: копируются ссылки, не объекты. Изменение через клон затрагивает оригинал.
- **for-each + remove** — `ConcurrentModificationException`. Используй `iterator.remove()` или `list.removeIf()`.
- **Fall-through в switch** — пропущенный `break` выполняет следующий case. Используй стрелочный синтаксис `->`.
- **StackOverflow из рекурсии** — Java не оптимизирует хвостовые вызовы. Размер стека по умолчанию ~512KB–1MB. Для глубоких структур — итерация.
