# Паттерн-матчинг в Scala

## Обзор

Паттерн-матчинг (pattern matching) - это мощная конструкция в Scala, которая позволяет декомпозировать данные и выполнять различные действия в зависимости от структуры и значений данных. Это более выразительная альтернатива условным операторам.

## Базовый синтаксис

### Простой match

```scala
def describeNumber(x: Int): String = x match {
  case 0 => "zero"
  case 1 => "one"
  case 2 => "two"
  case _ => "many"
}

// Использование
val result1 = describeNumber(0) // "zero"
val result2 = describeNumber(5) // "many"
```

### Match как выражение

```scala
val x = 10
val result = x match {
  case n if n > 0 => "positive"
  case n if n < 0 => "negative"
  case _ => "zero"
}
```

## Паттерны для различных типов

### Константные паттерны

```scala
def getDayName(day: Int): String = day match {
  case 1 => "Monday"
  case 2 => "Tuesday"
  case 3 => "Wednesday"
  case 4 => "Thursday"
  case 5 => "Friday"
  case 6 => "Saturday"
  case 7 => "Sunday"
  case _ => "Invalid day"
}
```

### Паттерны с переменными

```scala
def analyzeNumber(x: Any): String = x match {
  case 0 => "zero"
  case n: Int if n > 0 => s"positive integer: $n"
  case n: Int if n < 0 => s"negative integer: $n"
  case s: String => s"string: $s"
  case _ => "unknown"
}
```

### Типизированные паттерны

```scala
def processValue(x: Any): String = x match {
  case s: String => s"String: $s"
  case i: Int => s"Integer: $i"
  case d: Double => s"Double: $d"
  case b: Boolean => s"Boolean: $b"
  case _ => "Unknown type"
}
```

## Паттерны для коллекций

### Списки

```scala
def analyzeList(list: List[Int]): String = list match {
  case Nil => "empty list"
  case head :: Nil => s"single element: $head"
  case head :: tail => s"head: $head, tail: $tail"
}

// Использование
val result1 = analyzeList(List()) // "empty list"
val result2 = analyzeList(List(1)) // "single element: 1"
val result3 = analyzeList(List(1, 2, 3)) // "head: 1, tail: List(2, 3)"
```

### Массивы

```scala
def analyzeArray(arr: Array[Int]): String = arr match {
  case Array() => "empty array"
  case Array(x) => s"single element: $x"
  case Array(x, y) => s"two elements: $x, $y"
  case Array(x, y, z, _*) => s"three or more: $x, $y, $z"
}
```

### Кортежи

```scala
def analyzeTuple(tuple: (String, Int)): String = tuple match {
  case (name, age) if age < 18 => s"$name is a minor"
  case (name, age) if age >= 18 => s"$name is an adult"
}

// Использование
val result1 = analyzeTuple(("Alice", 16)) // "Alice is a minor"
val result2 = analyzeTuple(("Bob", 25)) // "Bob is an adult"
```

## Паттерны для case classes

### Простые case classes

```scala
case class Person(name: String, age: Int)
case class Book(title: String, author: String, year: Int)

def analyzeItem(item: Any): String = item match {
  case Person(name, age) if age < 18 => s"$name is a minor"
  case Person(name, age) => s"$name is an adult"
  case Book(title, author, year) => s"Book: $title by $author ($year)"
  case _ => "Unknown item"
}

// Использование
val person = Person("Alice", 20)
val book = Book("Scala Programming", "Martin Odersky", 2008)
val result1 = analyzeItem(person) // "Alice is an adult"
val result2 = analyzeItem(book) // "Book: Scala Programming by Martin Odersky (2008)"
```

### Вложенные case classes

```scala
case class Address(street: String, city: String)
case class Employee(name: String, address: Address, salary: Double)

def analyzeEmployee(emp: Employee): String = emp match {
  case Employee(name, Address(street, city), salary) if salary > 50000 =>
    s"$name lives in $city and earns well"
  case Employee(name, Address(street, city), salary) =>
    s"$name lives in $city"
}
```

## Паттерны с условиями (guards)

```scala
def analyzeNumber(x: Int): String = x match {
  case n if n > 0 && n < 10 => "single digit positive"
  case n if n >= 10 && n < 100 => "two digit number"
  case n if n >= 100 => "large number"
  case 0 => "zero"
  case n if n < 0 => "negative"
}

// Использование
val result1 = analyzeNumber(5) // "single digit positive"
val result2 = analyzeNumber(25) // "two digit number"
val result3 = analyzeNumber(150) // "large number"
```

## Паттерны для Option

```scala
def processOption(opt: Option[String]): String = opt match {
  case Some(value) => s"Found: $value"
  case None => "Nothing found"
}

// Использование
val result1 = processOption(Some("Hello")) // "Found: Hello"
val result2 = processOption(None) // "Nothing found"
```

## Паттерны для Try

```scala
import scala.util.{Try, Success, Failure}

def processTry(result: Try[Int]): String = result match {
  case Success(value) => s"Success: $value"
  case Failure(exception) => s"Failure: ${exception.getMessage}"
}

// Использование
val success = Try(10 / 2)
val failure = Try(10 / 0)
val result1 = processTry(success) // "Success: 5"
val result2 = processTry(failure) // "Failure: / by zero"
```

## Паттерны для sealed traits

```scala
sealed trait Shape
case class Circle(radius: Double) extends Shape
case class Rectangle(width: Double, height: Double) extends Shape
case class Square(side: Double) extends Shape

def calculateArea(shape: Shape): Double = shape match {
  case Circle(radius) => math.Pi * radius * radius
  case Rectangle(width, height) => width * height
  case Square(side) => side * side
}

// Использование
val circle = Circle(5.0)
val rectangle = Rectangle(3.0, 4.0)
val square = Square(6.0)

val area1 = calculateArea(circle) // 78.54...
val area2 = calculateArea(rectangle) // 12.0
val area3 = calculateArea(square) // 36.0
```

## Паттерны в for-выражениях

```scala
val list = List(Some(1), None, Some(2), Some(3), None)

// Извлечение значений из Some
val values = for {
  Some(value) <- list
} yield value
// values = List(1, 2, 3)

// Паттерны в генераторах
val pairs = List(("Alice", 25), ("Bob", 30), ("Charlie", 35))
val names = for {
  (name, age) <- pairs if age > 25
} yield name
// names = List("Bob", "Charlie")
```

## Паттерны в частичных функциях

```scala
val analyzeNumber: PartialFunction[Any, String] = {
  case n: Int if n > 0 => "positive"
  case n: Int if n < 0 => "negative"
  case 0 => "zero"
  case s: String => "string"
}

// Использование
val result1 = analyzeNumber(5) // "positive"
val result2 = analyzeNumber(-3) // "negative"
val result3 = analyzeNumber("hello") // "string"

// Проверка определенности
analyzeNumber.isDefinedAt(5) // true
analyzeNumber.isDefinedAt(3.14) // false
```

## Типичные ошибки

### 1. Неполный паттерн-матчинг

```scala
// Ошибка - не все случаи покрыты
def process(x: Int): String = x match {
  case 1 => "one"
  case 2 => "two"
  // Отсутствует case для других значений
}

// Правильно
def process(x: Int): String = x match {
  case 1 => "one"
  case 2 => "two"
  case _ => "other"
}
```

### 2. Неправильный порядок паттернов

```scala
// Ошибка - более общий паттерн перехватывает все
def process(x: Any): String = x match {
  case _ => "anything"
  case s: String => "string" // Этот case никогда не выполнится
}

// Правильно
def process(x: Any): String = x match {
  case s: String => "string"
  case _ => "anything"
}
```

### 3. Забывание скобок в паттернах

```scala
// Ошибка - неправильный синтаксис
def process(x: Int): String = x match {
  case n if n > 0 => "positive"
  case n if n < 0 => "negative"
  case 0 => "zero"
}

// Правильно
def process(x: Int): String = x match {
  case n if n > 0 => "positive"
  case n if n < 0 => "negative"
  case 0 => "zero"
}
```

## Лучшие практики

### 1. Используйте sealed traits для полного покрытия

```scala
sealed trait Result
case class Success(value: String) extends Result
case class Error(message: String) extends Result

def processResult(result: Result): String = result match {
  case Success(value) => s"Success: $value"
  case Error(message) => s"Error: $message"
  // Компилятор проверит полноту покрытия
}
```

### 2. Используйте паттерн-матчинг вместо if-else

```scala
// Хорошо
def analyze(x: Any): String = x match {
  case s: String => s"String: $s"
  case i: Int => s"Int: $i"
  case _ => "Unknown"
}

// Избегайте
def analyze(x: Any): String = {
  if (x.isInstanceOf[String]) s"String: $x"
  else if (x.isInstanceOf[Int]) s"Int: $x"
  else "Unknown"
}
```

### 3. Используйте деструктуризацию

```scala
// Хорошо
def processPerson(person: Person): String = person match {
  case Person(name, age) => s"$name is $age years old"
}

// Избегайте
def processPerson(person: Person): String = {
  s"${person.name} is ${person.age} years old"
}
```

## Вопросы для собеседования

1. **В чем разница между паттерн-матчингом и if-else?**
2. **Что такое sealed trait и зачем его использовать?**
3. **Как работает деструктуризация в паттерн-матчинге?**
4. **Что такое частичные функции?**
5. **Как использовать паттерн-матчинг с коллекциями?**

## Дополнительные ресурсы

- [Scala Language Specification - Pattern Matching](https://scala-lang.org/files/archive/spec/2.13/08-pattern-matching.html)
- [Scala 3 Book - Pattern Matching](https://docs.scala-lang.org/scala3/book/pattern-matching.html)
- Martin Odersky - "Programming in Scala" (Chapter 15) 