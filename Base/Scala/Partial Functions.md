Tags: #scala #scala-base

### **Что такое частичные функции?**

Частичная функция — это функция, которая определена не для всех возможных входных значений. В Scala частичные функции представлены трейтом `PartialFunction[A, B]`, где:

- `A` — тип входного значения.
- `B` — тип выходного значения.

Частичные функции полезны, когда функция должна работать только с подмножеством возможных входных данных.

---
### **Синтаксический сахар `case`**

В Scala синтаксис `case` — это удобный способ определения частичных функций. Под капотом этот синтаксис преобразуется в методы, которые реализуют трейт `PartialFunction`. Давайте разберем, как это работает.

---
### **Трейт `PartialFunction`**

Трейт `PartialFunction` требует реализации двух методов:

1. **`isDefinedAt(x: A): Boolean`** — проверяет, определена ли функция для значения `x`.
2. **`apply(x: A): B`** — применяет функцию к значению `x`.

Когда вы используете синтаксис `case`, компилятор Scala автоматически генерирует эти методы.

---
### **Пример: создание частичной функции с `case`**

Рассмотрим пример частичной функции, которая работает только с четными числами:

``` Scala
val isEven: PartialFunction[Int, String] = {
  case x if x % 2 == 0 => s"$x is even"
}
```

Этот код эквивалентен следующему:

``` Scala
val isEven = new PartialFunction[Int, String] {
  // Метод, проверяющий, определена ли функция для x
  def isDefinedAt(x: Int): Boolean = x % 2 == 0

  // Метод, применяющий функцию к x
  def apply(x: Int): String = s"$x is even"
}
```

Здесь:

- `isDefinedAt` проверяет, является ли число четным.
- `apply` возвращает строку с результатом.
---
### **Сравнение с Java**

В Java нет встроенной поддержки частичных функций, но можно создать аналогичный функционал с помощью интерфейсов и классов. Например, в Java это могло бы выглядеть так:

``` java 
interface PartialFunction<A, B> {
    boolean isDefinedAt(A x);
    B apply(A x);
}

class IsEven implements PartialFunction<Integer, String> {
    @Override
    public boolean isDefinedAt(Integer x) {
        return x % 2 == 0;
    }

    @Override
    public String apply(Integer x) {
        return x + " is even";
    }
}

public class Main {
    public static void main(String[] args) {
        PartialFunction<Integer, String> isEven = new IsEven();
        if (isEven.isDefinedAt(4)) {
            System.out.println(isEven.apply(4)); // 4 is even
        }
    }
}
```
В Scala синтаксис `case` позволяет сделать то же самое более лаконично и выразительно.

---
### **Методы частичных функций**

#### 1. `isDefinedAt`

Проверяет, определена ли функция для конкретного значения.

``` Scala
println(isEven.isDefinedAt(2)) // true
println(isEven.isDefinedAt(3)) // false
```
#### 2. `apply`

Применяет функцию к значению. Если функция не определена для этого значения, выбрасывается исключение `MatchError`.

``` Scala
println(isEven(2)) // "2 is even"
// println(isEven(3)) // MatchError
```
#### 3. `orElse`

Комбинирует две частичные функции. Если первая функция не определена для входного значения, используется вторая.

``` Scala
val isOdd: PartialFunction[Int, String] = {
  case x if x % 2 != 0 => s"$x is odd"
}

val checkNumber = isEven orElse isOdd

println(checkNumber(2)) // "2 is even"
println(checkNumber(3)) // "3 is odd"
```
#### 4. `andThen`

Комбинирует частичную функцию с другой функцией. Результат частичной функции передается в следующую функцию.

``` Scala
val double: Int => Int = x => x * 2

val evenAndDouble = isEven andThen double

println(evenAndDouble(2)) // 4
```
---
### **Использование частичных функций**

#### 1. **С коллекциями**

Частичные функции часто используются с методами коллекций, такими как `collect`, который применяет функцию только к тем элементам, для которых она определена.

``` Scala
val numbers = List(1, 2, 3, 4, 5)

val evenNumbers = numbers.collect(isEven)

println(evenNumbers) // List("2 is even", "4 is even")
```
#### 2. **Обработка ошибок**

Частичные функции полезны для обработки ошибок или исключительных ситуаций, когда функция может быть не определена для некоторых входных данных.

``` Scala
val safeDivide: PartialFunction[(Int, Int), Int] = {
  case (x, y) if y != 0 => x / y
}

println(safeDivide.isDefinedAt((10, 0))) // false
println(safeDivide((10, 2))) // 5
```
#### 3. **Комбинирование функций**

Частичные функции можно комбинировать для создания более сложных функций.

``` Scala
val checkNumber = isEven orElse isOdd

println(checkNumber(2)) // "2 is even"
println(checkNumber(3)) // "3 is odd"
```