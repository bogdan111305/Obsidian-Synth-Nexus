---
title: "Java — String, StringBuilder, StringBuffer, Text Blocks, Regex"
tags: [java, string, stringbuilder, text-blocks, regex, stringjoiner]
updated: 2026-08-15
---

# Работа со строками в Java

> [!QUOTE] Суть
> `String` — иммутабельный, хранится в String Pool (heap). `"a" == "a"` — true (пул), `new String("a") == new String("a")` — false! `StringBuilder` — мутабельный, не thread-safe, быстрый. `StringBuffer` — thread-safe, медленнее. Java 9+: compact strings (`byte[]` вместо `char[]`).

> [!WARNING] Ловушка: сравнение строк через ==
> `==` сравнивает **ссылки**, а не содержимое. `"hello" == new String("hello")` → `false`. Всегда используй `.equals()` для сравнения содержимого строк. Исключение: интернированные строки (`intern()`), но это оптимизация, а не рабочая практика.

## 1. Класс `String`

Класс `String` (`java.lang`) представляет неизменяемые строки в кодировке UTF-16.

- **Неизменяемость** — любая операция (`substring`, `replace`, `toUpperCase`...) возвращает новый объект; внутренний массив помечен `final`. Как следствие — `String` потокобезопасен без какой-либо синхронизации.
- **Пул строк (String Pool)** — строковые литералы (`"text"`) кладутся в пул (в heap, начиная с Java 7); `new String("text")` создаёт отдельный объект в куче, минуя пул; `intern()` возвращает (или добавляет и возвращает) ссылку из пула.
- **Хранение** — Java 8: `char[]`, 2 байта/символ. Java 9+: `byte[]` с флагом кодировки (Compact Strings — механика и влияние на производительность разобраны в Senior Insights ниже).
- **Конкатенация `+`/`concat()` в цикле — антипаттерн**: каждая итерация создаёт промежуточный объект. Для многократной сборки строки используй `StringBuilder` (см. §2).

```java
String literal = "Привет";                          // в пуле строк
String fresh   = new String("Привет");               // отдельный объект в куче
System.out.println(literal == fresh);                 // false — разные объекты
System.out.println(literal == fresh.intern());        // true — intern() вернул ссылку из пула

// Антипаттерн — 1000 промежуточных объектов String:
String result = "";
for (int i = 0; i < 1000; i++) result += i;
```

### Основные методы `String`

|Метод|Описание|Пример|
|---|---|---|
|`length()`|Возвращает длину строки.|`"Привет".length()` → 6|
|`charAt(int index)`|Возвращает символ по индексу.|`"Привет".charAt(0)` → 'П'|
|`substring(int begin, int end)`|Извлекает подстроку.|`"Привет".substring(1, 3)` → "ри"|
|`toLowerCase()`, `toUpperCase()`|Преобразует регистр.|`"Привет".toUpperCase()` → "ПРИВЕТ"|
|`trim()`|Удаляет пробелы в начале и конце.|`" Привет ".trim()` → "Привет"|
|`replace(char old, char new)`|Заменяет символы.|`"Привет".replace('П', 'Б')` → "Бривет"|
|`replaceAll(String regex, String replacement)`|Заменяет по регулярному выражению.|`"a1b2".replaceAll("\\d", "_")` → "a_b_"|
|`split(String regex)`|Разделяет строку по регулярному выражению.|`"a,b,c".split(",")` → `["a", "b", "c"]`|
|`startsWith(String prefix)`, `endsWith(String suffix)`|Проверяет начало/конец строки.|`"Привет".startsWith("Пр")` → `true`|
|`contains(CharSequence s)`|Проверяет наличие подстроки.|`"Привет".contains("ри")` → `true`|
|`indexOf(String str)`, `lastIndexOf(String str)`|Возвращает индекс первого/последнего вхождения.|`"Привет".indexOf("р")` → 1|
|`isEmpty()`, `isBlank()`|Проверяет, пуста ли строка или содержит только пробелы (Java 11+).|`" ".isBlank()` → `true`|
|`concat(String str)`|Конкатенирует строки.|`"При".concat("вет")` → "Привет"|
|`equals(Object obj)`, `equalsIgnoreCase(String str)`|Сравнивает строки.|`"Привет".equals("привет")` → `false`|
|`compareTo(String str)`, `compareToIgnoreCase(String str)`|Сравнивает лексикографически.|`"а".compareTo("б")` → -1|
|`intern()`|Возвращает строку из пула строк.|`new String("Привет").intern() == "Привет"` → `true`|
|`join(CharSequence delimiter, CharSequence... elements)`|Объединяет строки с разделителем (Java 8+).|`String.join(",", "a", "b")` → "a,b"|
|`strip()`, `stripLeading()`, `stripTrailing()`|Удаляет пробелы (Unicode-aware, Java 11+).|`" Привет ".strip()` → "Привет"|
|`repeat(int count)`|Повторяет строку (Java 11+).|`"А".repeat(3)` → "ААА"|
|`lines()`|Возвращает Stream строк, разделенных переносами (Java 11+).|`"a\nb".lines().count()` → 2|

## 2. Класс `StringBuilder`

- **Мутабельный, не synchronized** — быстрее `String` и `StringBuffer` при многократных модификациях; это рабочая лошадка для сборки строк в цикле.
- **Рост буфера** — по достижении capacity внутренний массив расширяется по формуле `newCapacity = oldCapacity * 2 + 2` (аналог механики `ArrayList`, но агрессивнее). `new StringBuilder(expectedSize)` избавляет от лишних resize.
- **Не thread-safe** — при доступе из нескольких потоков нужен либо `StringBuffer`, либо внешняя синхронизация.

|Метод|Описание|Пример|
|---|---|---|
|`append(T value)`|Добавляет значение (поддерживает примитивы, строки, объекты).|`new StringBuilder().append("А").append(1)` → "А1"|
|`insert(int offset, T value)`|Вставляет значение по индексу.|`new StringBuilder("АБ").insert(1, "В")` → "АВБ"|
|`delete(int start, int end)`|Удаляет подстроку.|`new StringBuilder("АБВ").delete(1, 2)` → "АВ"|
|`replace(int start, int end, String str)`|Заменяет подстроку.|`new StringBuilder("АБВ").replace(1, 2, "Г")` → "АГВ"|
|`reverse()`|Переворачивает строку.|`new StringBuilder("АБВ").reverse()` → "ВБА"|
|`toString()`|Преобразует в `String`.|`new StringBuilder("Привет").toString()` → "Привет"|
|`capacity()`|Возвращает текущую емкость внутреннего массива.|`new StringBuilder("А").capacity()` → 16 (по умолчанию)|
|`ensureCapacity(int min)`|Увеличивает емкость, если нужно.|`sb.ensureCapacity(50)`|
|`setLength(int newLength)`|Устанавливает новую длину строки.|`sb.setLength(2)`|

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i); // Эффективно: один изменяемый буфер, без промежуточных String
}
System.out.println(sb.toString()); // 0123...999
```

## 3. Класс `StringBuffer`

- Полный функциональный аналог `StringBuilder` — все методы (`append`, `insert`, `delete`, `replace`, `reverse`, `toString`, `capacity`, `ensureCapacity`, `setLength`) идентичны, но **synchronized**.
- Потокобезопасен ценой производительности (блокировка на каждый вызов). На практике вытеснен `StringBuilder` — используй `StringBuffer`, только если сам объект-накопитель должен быть общим mutable состоянием нескольких потоков одновременно.

```java
StringBuffer sb = new StringBuffer();
ExecutorService executor = Executors.newFixedThreadPool(2);
for (int i = 0; i < 1000; i++) {
    executor.submit(() -> sb.append("A"));
}
executor.shutdown();
executor.awaitTermination(10, TimeUnit.SECONDS);
System.out.println(sb.length()); // 1000 (потокобезопасно, гонок нет)
```

## 4. Класс `StringJoiner` (Java 8+)

- Объединяет строки с разделителем и опциональными префиксом/суффиксом; под капотом использует `StringBuilder`, поэтому **не thread-safe**.
- Удобен для CSV/списков — избавляет от ручной логики "разделитель между элементами, но не перед первым".
- `setEmptyValue()` задаёт представление для пустого джойнера, `merge()` объединяет два `StringJoiner` (сохраняя разделитель/префикс/суффикс текущего).

|Метод|Описание|Пример|
|---|---|---|
|`add(CharSequence)`|Добавляет строку.|`joiner.add("А")` → добавляет "А"|
|`toString()`|Возвращает итоговую строку.|`joiner.toString()` → "[А,Б]"|
|`setEmptyValue(CharSequence)`|Устанавливает значение для пустого `StringJoiner`.|`joiner.setEmptyValue("пусто")`|
|`merge(StringJoiner)`|Объединяет другой `StringJoiner`.|`joiner.merge(anotherJoiner)`|
|`length()`|Возвращает длину текущей строки.|`joiner.length()` → длина результата|

```java
StringJoiner joiner = new StringJoiner(",", "[", "]");
joiner.add("А").add("Б").add("В");
System.out.println(joiner); // [А,Б,В]

StringJoiner emptyJoiner = new StringJoiner(",");
emptyJoiner.setEmptyValue("пусто");
System.out.println(emptyJoiner); // пусто

StringJoiner other = new StringJoiner(",");
other.add("Г").add("Д");
joiner.merge(other);
System.out.println(joiner); // [А,Б,В,Г,Д]
```

## 5. `String.format` и `String.formatted` (Java 15+)

- C-style `printf`-спецификаторы (`%s`, `%d`, `%f`, ...) для читаемых форматированных строк (логи, отчёты, денежные суммы).
- `formatted(Object...)` — тот же механизм, но вызывается прямо на строке-шаблоне (`"...".formatted(args)`), без статического `String.format(template, args)`.
- Каждый вызов парсит шаблон и создаёт новый объект — потокобезопасно, но медленнее прямой сборки через `StringBuilder`. Не используй в горячих циклах, только для читаемости точечных операций.

|Спецификатор|Описание|Пример|
|---|---|---|
|`%s`|Строка|`String.format("Имя: %s", "Алексей")` → "Имя: Алексей"|
|`%d`|Целое число|`String.format("Возраст: %d", 30)` → "Возраст: 30"|
|`%f`|Число с плавающей точкой|`String.format("Цена: %.2f", 12.345)` → "Цена: 12.35"|
|`%n`|Перенос строки|`String.format("А%nБ")` → "А\nБ"|
|`%x`|Шестнадцатеричное число|`String.format("Число: %x", 255)` → "Число: ff"|

```java
String name = "Алексей";
int age = 30;
double price = 123.45678;

System.out.println(String.format("Имя: %s, Возраст: %d", name, age)); // Имя: Алексей, Возраст: 30
System.out.println("Цена: %10.2f рублей".formatted(price));           // Цена:     123.46 рублей
```

## 6. Текстовые блоки (Java 15+)

- Синтаксис `"""..."""` для многострочных строк без ручного экранирования `\n`.
- Результат — обычный (иммутабельный) объект `String`; блок автоматически убирает общий минимальный отступ у всех строк.
- Хорошо сочетается с `.formatted()` для интерполяции переменных; годится для статичных шаблонов (HTML, JSON, SQL), но не для динамической построчной сборки — для этого нужен `StringBuilder`.

```java
String name = "Алексей";
String block = """
    Пользователь: %s
    Статус: активен
    """.formatted(name);
System.out.println(block);
// Пользователь: Алексей
// Статус: активен
```

## 7. Stream API и `Collectors.joining` (Java 8+)

- Функциональный способ объединить строки из `Stream` с разделителем/префиксом/суффиксом — под капотом тот же `StringBuilder`, что и у `StringJoiner`.
- Естественно комбинируется с `filter`/`map` в одном пайплайне, что делает его удобнее ручного `StringJoiner` при обработке коллекций.

|Метод|Описание|Пример|
|---|---|---|
|`joining()`|Объединяет без разделителя.|`stream.collect(Collectors.joining())` → "АБВ"|
|`joining(CharSequence delimiter)`|Объединяет с разделителем.|`stream.collect(Collectors.joining(","))` → "А,Б,В"|
|`joining(CharSequence delimiter, CharSequence prefix, CharSequence suffix)`|Объединяет с разделителем, префиксом и суффиксом.|`stream.collect(Collectors.joining(",", "[", "]"))` → "[А,Б,В]"|

```java
List<String> words = List.of("Привет", "мир", "!");
String result = words.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.joining(",", "<", ">"));
System.out.println(result); // <Привет>
```

## 8. Регулярные выражения

`String` интегрирует regex через `replaceAll`, `replaceFirst`, `split`, `matches`; для многократного использования шаблона — `Pattern`/`Matcher` из `java.util.regex`.

- **`Pattern`** компилирует regex один раз (компиляция дорогая) — храни в `static final` поле, а не создавай на каждый вызов.
- **`Matcher`** выполняет поиск/сопоставление по скомпилированному `Pattern`.
- **Жадные квантификаторы** (`*`, `+`) на больших строках со сложными шаблонами могут привести к катастрофическому backtracking ("зависание"). Используй нежадные (`*?`, `+?`) и тестируй шаблоны на граничных данных.

```java
static final Pattern NUMBER_PATTERN = Pattern.compile("\\d+"); // кэшированный Pattern

String text = "Контакт: +7-123-456-7890, другой: +7-987-654-3210";
Matcher matcher = NUMBER_PATTERN.matcher(text);
while (matcher.find()) {
    System.out.println(matcher.group());
}

String replaced = text.replaceAll("\\d{3}-\\d{4}", "XXX-XXXX");
System.out.println(replaced); // Контакт: +7-XXX-XXXX, другой: +7-XXX-XXXX
```

## 9. Сравнение инструментов

|Инструмент|Изменяемость|Потокобезопасность|Производительность|Основное использование|
|---|---|---|---|---|
|`String`|Нет|Да|Низкая при конкатенации|Постоянные строки, пул строк|
|`StringBuilder`|Да|Нет|Высокая|Однопоточная конкатенация|
|`StringBuffer`|Да|Да|Средняя|Многопоточная конкатенация|
|`StringJoiner`|Да|Нет|Высокая|Объединение с разделителями|
|`String.format`/`formatted`|Нет|Да|Средняя|Форматированные строки|
|Text Blocks|Нет|Да|Низкая при модификациях|Многострочные шаблоны|
|`Collectors.joining`|Да|Зависит от потока|Средняя|Объединение строк в Stream API|

## 10. Типичные ошибки

- **Конкатенация в цикле через `+`** — используй `StringBuilder`/`StringJoiner`/`Collectors.joining` (см. таблицу §9 для выбора инструмента).
- **`new String("text")` без причины** и избыточный `intern()` — лишние объекты и (для `intern()`) конкуренция за глобальный пул строк (см. Senior Insights).
- **Незакэшированный `Pattern.compile()`** в цикле — компиляция regex дорогая; храни `Pattern` в `static final`.
- **Жадные квантификаторы regex** на непроверенном вводе — риск катастрофического backtracking; используй нежадные варианты и тестируй на граничных данных.
- **Отсутствие проверки на `null`/пустоту** перед `String.format`/regex-операциями — источник `NullPointerException`/`PatternSyntaxException` в проде.

## 11. Пример: комбинированное использование

```java
import java.util.List;
import java.util.StringJoiner;
import java.util.regex.Pattern;
import java.util.stream.Collectors;

public class TextProcessor {
    private static final Pattern NUMBER_PATTERN = Pattern.compile("\\d+");

    public static String processComplexText(List<String> words) {
        StringJoiner joiner = new StringJoiner(", ", "[", "]");
        words.forEach(joiner::add);

        String filtered = words.stream()
            .filter(s -> s.length() > 3)
            .collect(Collectors.joining(";"));

        return """
            Исходный список: %s
            Отфильтрованные слова: %s
            Обработанные числа: %s
            """.formatted(
                joiner.toString(),
                filtered.isEmpty() ? "пусто" : filtered,
                NUMBER_PATTERN.matcher(joiner.toString()).replaceAll("#")
            );
    }

    public static void main(String[] args) {
        List<String> words = List.of("Привет", "мир", "123", "Java");
        System.out.println(processComplexText(words));
        // Исходный список: [Привет, мир, 123, Java]
        // Отфильтрованные слова: Привет;Java
        // Обработанные числа: [Привет, мир, #, Java]
    }
}
```

## Senior Insights

### String Deduplication (G1 GC)

String Deduplication — механизм G1 GC для устранения дубликатов внутренних `byte[]` массивов у одинаковых строк.

```java
// Включение:
// -XX:+UseStringDeduplication  (требует G1 GC)
// -XX:+PrintStringDeduplicationStatistics  (статистика)

// Как работает:
// 1. GC инспектирует String объекты во время Minor GC
// 2. Вычисляет hash для каждого byte[] массива
// 3. Если находит дублирующиеся массивы — заменяет их ссылкой на один
// 4. Старые дубликаты освобождаются GC

// ВАЖНО: Работает только для byte[] (Java 9+ Compact Strings)
// String объекты остаются разными — только внутренний массив общий!

String s1 = new String("Hello World"); // byte[] #1
String s2 = new String("Hello World"); // byte[] #2
// После G1 String Deduplication:
// s1 и s2 разные объекты, но оба ссылаются на byte[] #1 (byte[] #2 GC-нут)
// s1 == s2 → false (разные String объекты)
// s1.equals(s2) → true (одинаковое содержимое)

// Когда применять:
// - Много String с одинаковым содержимым (логи, имена полей, коды)
// - Heap > 1GB с высокой долей String
// - Типичный выигрыш: 10-25% heap в enterprise приложениях
```

### Compact Strings (Java 9+) — внутренняя оптимизация

```java
// Java 8: String хранит char[] (всегда 2 байта/символ, UTF-16)
// Java 9+: String хранит byte[] с CODER флагом:
//   - LATIN1 (0): 1 байт/символ если все символы ASCII/Latin-1 (U+0000-U+00FF)
//   - UTF16  (1): 2 байта/символ если есть Unicode символы

// Практический эффект:
String ascii = "Hello World";     // LATIN1: 11 байт вместо 22 байт
String unicode = "Привет мир";    // UTF16: 18 байт (кириллица)
String mixed = "Hello Привет";    // UTF16: весь массив UTF16 из-за одного unicode символа

// Проверка CODER через reflection (не для production):
Field coderField = String.class.getDeclaredField("coder");
coderField.setAccessible(true);
byte coder = (byte) coderField.get(ascii);
System.out.println(coder == 0 ? "LATIN1" : "UTF16"); // LATIN1

// Влияние на производительность:
// - substring(), equals(), hashCode() - быстрее для LATIN1 строк
// - String.compareTo() использует SIMD векторизацию для LATIN1
// - Рекомендация: избегать смешивания кириллицы в ASCII-dominated данных
```

### String.intern() — осторожно!

```java
// BAD: intern() для всех строк - убивает производительность
for (String line : largeFile) {
    String interned = line.intern(); // КОНКУРЕНЦИЯ за String pool!
    // String pool — глобальная ConcurrentHashMap (до Java 8: PermGen, теперь Heap)
    // При высоком параллелизме — contention на pool lock!
}

// SENIOR WAY: intern() только для СТАБИЛЬНЫХ, ЧАСТО ПОВТОРЯЮЩИХСЯ строк
// Хорошие кандидаты: имена полей, коды состояний, страны, категории
// Плохие кандидаты: UUID, email, username (каждый уникален)

// Java 21+ альтернатива для кэширования строк:
// StringTable (символьная таблица JVM) — не трогать вручную
// Вместо intern(): использовать explicit HashMap<String, String>
Map<String, String> cache = new HashMap<>();
String intern(String s) { return cache.computeIfAbsent(s, k -> k); }
// Нет глобального lock, контролируемый размер
```

### Senior Interview Q&A

**Q1: Почему `String.hashCode()` кэшируется, и как это влияет на HashMap?**

> `String.hashCode()` вычисляется лениво при первом вызове и кэшируется в поле `int hash` (с Java 8 — дополнительный `boolean hashIsZero` для строк с hash=0). Это означает: первый `hashCode()` — O(n) по длине строки, последующие — O(1). HashMap/HashSet вызывают `hashCode()` при каждом `put()`/`get()`. Для горячих String ключей (заголовки HTTP, имена полей) кэш значительно ускоряет lookup. Подводный камень: после `String.intern()` hash кэш НЕ передаётся интернированной строке — она пересчитает при первом обращении.

**Q2: Как String Deduplication отличается от String Pool (`intern()`)?**

> `String.intern()`: программист явно просит JVM вернуть строку из String Pool (глобальный set). Два разных `String` объекта с одинаковым содержимым → один объект с одним `byte[]`. String Deduplication (G1): GC автоматически находит разные `String` объекты с одинаковым `byte[]` и заменяет дублирующиеся массивы одним. Объекты остаются разными, только внутренний массив общий. Deduplication не требует изменений в коде, работает в фоне во время GC, но: (1) работает только с G1; (2) не уменьшает число String объектов (только массивы); (3) добавляет небольшой overhead GC (~5-10%).

**Q3: Почему конкатенация строк в Java 9+ быстрее чем в Java 8?**

> Java 8: `"Hello " + name + "!"` компилируется в `new StringBuilder().append("Hello ").append(name).append("!").toString()`. Проблема: StringBuilder не знает финальный размер → возможен resize с копированием. Java 9+: `invokedynamic` + `StringConcatFactory.makeConcatWithConstants()`. Bootstrap метод анализирует шаблон конкатенации и генерирует оптимальный код: (1) вычисляет необходимый размер массива заранее; (2) аллоцирует `byte[]` точного размера; (3) копирует части напрямую без промежуточного StringBuilder. Для Latin-1 строк дополнительно использует SIMD. Benchmark: 15-30% быстрее для типичных конкатенаций.
