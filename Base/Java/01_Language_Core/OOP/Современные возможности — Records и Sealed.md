## Связанные темы

[[Инкапсуляция]], [[Наследование]], [[Полиморфизм]], [[Современные возможности — Switch и Pattern Matching]]

---

**Records, Sealed Classes и Pattern Matching** — это «святая троица» современной Java (версии 16–22). Вместе они реализуют концепцию **Data-Oriented Programming (DOP)** и привносят в язык мощь алгебраических типов данных (ADT), делая код безопаснее, лаконичнее и быстрее на уровне JVM.

---
## Records: Иммутабельные носители данных (Java 16+)

**Record** — это компактный класс-контейнер, чья единственная цель — прозрачно хранить неизменяемые данные.

### Что генерирует компилятор

Для записи `public record Point(int x, int y) {}` компилятор неявно создает `final` класс, наследующий `java.lang.Record`, и автоматически генерирует:

- `private final` поля.
    
- **Canonical Constructor** (конструктор со всеми аргументами).
    
- **Аксессоры:** `x()` и `y()` (обратите внимание: не `getX()`, это осознанный уход от JavaBeans).
    
- **Методы `equals`, `hashCode`, `toString`:** Генерируются не через медленную рефлексию, а через инструкцию **`invokedynamic`**, что позволяет JVM оптимизировать их выполнение.

### Компактный конструктор (Compact Constructor)

Идеальное место для валидации или защитного копирования. Параметры в скобках не пишутся, а присваивание полей происходит неявно в конце:

```java
public record Wrapper(List<String> items) {
    public Wrapper {
        if (items == null) throw new IllegalArgumentException();
        items = List.copyOf(items); // Защита от мутации оригинального списка
        // this.items = items; <-- компилятор добавит это сам
    }
}
```

### Безопасность десериализации

В отличие от обычных классов, которые при десериализации создаются в обход конструктора (через «магию» `Unsafe`), Records **всегда** пропускают данные через Canonical Constructor. Это гарантирует, что невалидные данные никогда не попадут в объект.

---
## Sealed Classes: Запечатанные иерархии (Java 17+)

Если интерфейсы раньше были открыты для всех, то **sealed-типы** вводят строгий фейсконтроль. Вы явно перечисляете классы, которым разрешено от вас наследоваться.

### Синтаксис и правила

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public final class Circle    implements Shape { ... } // Дальше наследовать нельзя
public sealed class Rectangle implements Shape permits Square { ... } // Продолжает закрытую ветку
public non-sealed class Triangle implements Shape { ... } // Открывает иерархию для всех (escape hatch)
```

_Правило:_ Каждый класс в `permits` обязан выбрать один из трех модификаторов: `final`, `sealed` или `non-sealed`. (Записи/Records неявно `final`, поэтому для них модификатор писать не нужно).
### Оптимизация JIT-компилятора (CHA)

Благодаря метаданным `PermittedSubclasses`, JIT-компилятор использует **Class Hierarchy Analysis (CHA)**. Зная, что у `Shape` ровно три реализации, JVM может девиртуализировать вызовы методов и применить агрессивный инлайнинг (inlining), превращая полиморфные вызовы в прямые.

---
## Pattern Matching: Механизм деструктуризации и проверок

Это клей, который объединяет Records и Sealed Classes в мощный инструмент бизнес-логики, избавляя нас от цепочек `instanceof` + кастование.

### Pattern Matching for `instanceof` (Java 16)

Переменная паттерна (pattern variable) автоматически кастуется, если проверка успешна.

```java
// Классика:
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s.toUpperCase()); 
}

// Трюк с инверсией (Negation Scope):
if (!(obj instanceof String s)) {
    return; // Если не String, выходим
}
System.out.println(s.toUpperCase()); // s доступна здесь!
```

### Pattern Matching in `switch` (Java 21)

`switch` теперь умеет работать с любыми типами, а не только с числами, строками и enum.

```java
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i < 0 -> "Отрицательное число"; // Guard-условие (when)
        case Integer i            -> "Положительное число"; // Общий case ниже специфичного
        case String s             -> "Строка: " + s;
        case null                 -> "Объект null";         // Явная обработка null!
        default                   -> "Другой тип";
    };
}
```

### Record Patterns (Java 21) и Unnamed Patterns (Java 22)

Главная магия — **деструктуризация**. Мы можем «разобрать» Record на составные части прямо в `switch` или `if`. Символ `_` (Java 22) позволяет игнорировать ненужные компоненты.

```java
record Point(int x, int y) {}
record Circle(Point center, double radius) {}

double process(Shape shape) {
    return switch (shape) {
        // Извлекаем только радиус, центр игнорируем (_)
        case Circle(_, var r) when r > 0 -> Math.PI * r * r; 
        
        // Вложенная деструктуризация: достаем координаты x и y
        case Circle(Point(var x, var y), _) -> x + y;       
        
        case Rectangle(var w, var h) -> w * h;
        // default НЕ НУЖЕН! Иерархия Shape запечатана (sealed), 
        // компилятор видит, что мы покрыли все варианты (Exhaustiveness check).
    };
}
```

---
## Вопросы на интервью (Senior Level)

1. **Что такое "dominated by previous case" в `switch`?**
    
    - _Ответ:_ Это ошибка компиляции. Возникает, если вы поставите более общий кейс (например, `case Integer i`) **до** более специфичного (например, `case Integer i when i < 0`). Верхний перехватит всё, и нижний никогда не выполнится.
    
2. **Зачем нужен модификатор `non-sealed`? Разве он не ломает идею `sealed`?**
    
    - _Ответ:_ Он нужен как «предохранительный клапан». Например, у вас есть AST-дерево, где встроенные узлы запечатаны (`sealed`), а узел `CustomNode` сделан `non-sealed`, чтобы пользователи вашей библиотеки могли писать свои расширения.
    
3. **Можно ли изменить переменную паттерна (например, `s` в `obj instanceof String s`)?**
    
    - _Ответ:_ Нет, pattern variables неявно являются `effectively final`.
    
4. **Сработает ли Record Pattern `case Point(int x, int y)` если в `switch` передать `null`?**
    
    - _Ответ:_ Нет. Паттерны типов и Record-паттерны не матчат `null`. Для него нужно писать отдельный `case null`.

---
## Подводные камни (Pitfalls)

- **NullPointerException в `switch`:** Раньше `switch` всегда бросал NPE при `null`. С появлением паттернов это поведение осталось для совместимости. Если вы передадите `null` в `switch` без явного `case null`, вы получите исключение.
    
- **Хрупкость Exhaustive Switch (Исчерпывающего свитча):** Если вы добавите новый класс в `permits` sealed-интерфейса, все `switch` без секции `default` по этому интерфейсу **перестанут компилироваться**. На самом деле это фича, а не баг — компилятор заставляет вас не забыть написать логику для нового класса.
    
- **List внутри Record:** Record гарантирует неизменность ссылок, но не содержимого. `record Wrapper(List<String> list)` позволит сделать `wrapper.list().add("хак")`. Всегда используйте `List.copyOf` в конструкторе и `Collections.unmodifiableList` (или просто возвращайте копию) в аксессоре.
    
- **Фреймворки и `x()`:** Старые версии библиотек (Jackson до 2.12, некоторые версии MapStruct или Hibernate) жестко завязаны на стандарт JavaBeans и ищут геттеры через `get...`. С Records они могут сломаться или потребовать дополнительных аннотаций (вроде `@JsonProperty` или `@JsonAutoDetect`).