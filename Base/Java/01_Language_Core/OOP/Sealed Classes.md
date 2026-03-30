# Sealed Classes (Java 17+)

> Закрытая иерархия классов/интерфейсов: только явно перечисленные в `permits` классы могут быть наследниками. Компилятор знает все подтипы → exhaustive switch без `default` → JIT-девиртуализация через CHA.

## Связанные темы
[[Современные возможности — Records и Sealed]], [[Records]], [[Pattern Matching]], [[Интерфейсы]], [[Полиморфизм]]

---

## Синтаксис

```java
public sealed class Shape permits Circle, Rectangle, Triangle {}

// Каждый permits-класс должен быть одним из:
public final class Circle extends Shape {       // final — нельзя наследовать дальше
    double radius;
}
public sealed class Rectangle extends Shape    // sealed — можно, но только указанным
    permits Square {
    double width, height;
}
public non-sealed class Triangle extends Shape { // non-sealed — открыт для всех
    double base, height;
}
```

**Требования:**
- Все `permits`-классы должны быть в том же пакете (без JPMS) или модуле
- `permits`-классы должны напрямую расширять/реализовывать sealed-тип
- Каждый permits-класс обязан быть `final`, `sealed` или `non-sealed`

## Sealed интерфейс

```java
public sealed interface Expr permits Num, Add, Mul {}

public record Num(int value)        implements Expr {}
public record Add(Expr l, Expr r)   implements Expr {}
public record Mul(Expr l, Expr r)   implements Expr {}
```

Records идеально сочетаются с sealed: `final` по умолчанию → не нужно указывать.

## Exhaustive switch

Компилятор знает все `permits` → не требует `default` (и предупреждает, если он лишний):

```java
int eval(Expr e) {
    return switch (e) {
        case Num(int v)        -> v;
        case Add(var l, var r) -> eval(l) + eval(r);
        case Mul(var l, var r) -> eval(l) * eval(r);
        // default НЕ нужен — sealed иерархия исчерпана
    };
}
// Добавишь новый permits-класс Div → compile error здесь → принудительное обновление
```

## Sealed + JIT (Class Hierarchy Analysis)

JIT использует `PermittedSubclasses` метаданные класса для **Class Hierarchy Analysis (CHA)**:

```java
// JIT знает: Shape имеет ровно 3 реализации
// → может девиртуализировать вызовы через Shape-ссылку
// → агрессивный инлайнинг

// Без sealed: JIT видит "открытую" иерархию → консервативные предположения
// С sealed: JIT видит "закрытую" → более смелые оптимизации
```

## Sealed vs Enum

| | Enum | Sealed |
|---|---|---|
| Экземпляры | Фиксированный набор singleton-ов | Неограниченное число экземпляров |
| Состояние | Одинаковое для всех экз. константы | Каждый подтип — свой класс с полями |
| Наследование | Нет | Полное (методы, поля, конструкторы) |
| Когда | Константы (статусы, дни недели) | ADT (Abstract Data Type), AST, Result |

```java
// Enum: ограниченный набор значений одного типа
enum Status { PENDING, ACTIVE, CLOSED }

// Sealed: ограниченная иерархия разных типов
sealed interface Result<T> permits Success, Failure {}
record Success<T>(T value)          implements Result<T> {}
record Failure<T>(String error)     implements Result<T> {}
```

## Вопросы на интервью

- Что означают `final`, `sealed`, `non-sealed` для permits-классов?
- Почему `sealed` позволяет убрать `default` из switch?
- Чем `sealed` помогает JIT-компилятору? Что такое CHA?
- В чём разница между sealed class и enum?
- Как sealed-иерархия взаимодействует с добавлением нового подтипа?
- Могут ли permits-классы находиться в разных пакетах?

## Подводные камни

- **Добавление нового permits-класса** — все switch без `default` перестанут компилироваться. Это намеренная фича — нельзя забыть обновить обработку.
- **`non-sealed` открывает иерархию снова** — треугольник `non-sealed` → любой может его наследовать → JIT теряет гарантии о полноте иерархии.
- **Permits-классы в том же пакете** — без модульной системы нельзя разнести permits по пакетам. С JPMS — можно внутри одного модуля.
- **Sealed интерфейс + `default`-метод** — добавление нового permits-класса к интерфейсу с `default`-методами: новый класс наследует `default`, но switch всё равно сломается — нужно добавить case.
- **Record автоматически `final`** — records уже `final`, поэтому в `permits` для sealed-интерфейса record не нужно дополнительно `final`.
