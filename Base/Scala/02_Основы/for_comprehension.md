# For-comprehension в Scala

## Обзор

For-comprehension - это мощная конструкция в Scala, которая позволяет писать декларативный код для работы с коллекциями, Option, Try и другими монадическими типами. Это синтаксический сахар для комбинации методов `map`, `flatMap`, `filter` и `foreach`.

## Базовый синтаксис

### Простой for-comprehension

```scala
val numbers = List(1, 2, 3, 4, 5)

// Базовый for-comprehension
val doubled = for {
  n <- numbers
} yield n * 2
// doubled = List(2, 4, 6, 8, 10)

// Эквивалентно
val doubledAlt = numbers.map(_ * 2)
```

### For-comprehension с условиями

```scala
val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// С фильтрацией
val evenSquares = for {
  n <- numbers
  if n % 2 == 0
} yield n * n
// evenSquares = List(4, 16, 36, 64, 100)

// Эквивалентно
val evenSquaresAlt = numbers.filter(_ % 2 == 0).map(n => n * n)
```

## Множественные генераторы

### Вложенные циклы

```scala
val numbers = List(1, 2, 3)
val letters = List('a', 'b', 'c')

// Множественные генераторы
val combinations = for {
  n <- numbers
  l <- letters
} yield s"$n$l"
// combinations = List("1a", "1b", "1c", "2a", "2b", "2c", "3a", "3b", "3c")

// Эквивалентно
val combinationsAlt = numbers.flatMap(n => letters.map(l => s"$n$l"))
```

### Работа с вложенными структурами

```scala
case class User(name: String, emails: List[String])

val users = List(
  User("Alice", List("alice@example.com", "alice@work.com")),
  User("Bob", List("bob@example.com"))
)

// Извлечение всех email
val allEmails = for {
  user <- users
  email <- user.emails
} yield email
// allEmails = List("alice@example.com", "alice@work.com", "bob@example.com")
```

## Работа с Option

### Извлечение значений из Option

```scala
val maybeName: Option[String] = Some("Alice")
val maybeAge: Option[Int] = Some(25)

// Безопасное извлечение
val userInfo = for {
  name <- maybeName
  age <- maybeAge
} yield s"$name is $age years old"
// userInfo = Some("Alice is 25 years old")

// Если один из Option равен None
val maybeName2: Option[String] = Some("Bob")
val maybeAge2: Option[Int] = None

val userInfo2 = for {
  name <- maybeName2
  age <- maybeAge2
} yield s"$name is $age years old"
// userInfo2 = None
```

### Обработка вложенных Option

```scala
case class Address(city: String)
case class User(name: String, address: Option[Address])

val users = List(
  User("Alice", Some(Address("New York"))),
  User("Bob", None),
  User("Charlie", Some(Address("London")))
)

// Извлечение городов
val cities = for {
  user <- users
  address <- user.address
} yield s"${user.name} lives in ${address.city}"
// cities = List("Alice lives in New York", "Charlie lives in London")
```

## Работа с Try

```scala
import scala.util.{Try, Success, Failure}

def divide(a: Int, b: Int): Try[Int] = Try(a / b)

val results = for {
  result1 <- divide(10, 2)
  result2 <- divide(20, 4)
  result3 <- divide(30, 6)
} yield result1 + result2 + result3
// results = Success(20)

// При ошибке
val errorResults = for {
  result1 <- divide(10, 2)
  result2 <- divide(20, 0) // Ошибка деления на ноль
  result3 <- divide(30, 6)
} yield result1 + result2 + result3
// errorResults = Failure(java.lang.ArithmeticException: / by zero)
```

## Работа с коллекциями

### Фильтрация и трансформация

```scala
val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// Сложная фильтрация и трансформация
val result = for {
  n <- numbers
  if n % 2 == 0
  if n > 5
  square = n * n
  if square < 100
} yield square
// result = List(36, 64)
```

### Работа с Map

```scala
val userAges = Map("Alice" -> 25, "Bob" -> 30, "Charlie" -> 35)

// Извлечение пользователей старше определенного возраста
val olderUsers = for {
  (name, age) <- userAges
  if age > 28
} yield s"$name is $age years old"
// olderUsers = List("Bob is 30 years old", "Charlie is 35 years old")
```

## Паттерн-матчинг в for-comprehension

### Извлечение из case classes

```scala
case class Person(name: String, age: Int, city: String)

val people = List(
  Person("Alice", 25, "New York"),
  Person("Bob", 30, "London"),
  Person("Charlie", 35, "Paris")
)

// Извлечение с паттерн-матчингом
val adultNames = for {
  Person(name, age, _) <- people
  if age >= 18
} yield name
// adultNames = List("Alice", "Bob", "Charlie")
```

### Работа с кортежами

```scala
val pairs = List(("Alice", 25), ("Bob", 30), ("Charlie", 35))

// Извлечение с паттерн-матчингом
val adultNames = for {
  (name, age) <- pairs
  if age >= 18
} yield name
// adultNames = List("Alice", "Bob", "Charlie")
```

## For-comprehension без yield

### Выполнение действий (foreach)

```scala
val numbers = List(1, 2, 3, 4, 5)

// Выполнение действий без возврата значения
for {
  n <- numbers
  if n % 2 == 0
} {
  println(s"Even number: $n")
}
// Выведет:
// Even number: 2
// Even number: 4

// Эквивалентно
numbers.filter(_ % 2 == 0).foreach(n => println(s"Even number: $n"))
```

## Сложные примеры

### Обработка вложенных структур

```scala
case class Order(items: List[Item])
case class Item(name: String, price: Double)

val orders = List(
  Order(List(Item("Book", 20.0), Item("Pen", 5.0))),
  Order(List(Item("Laptop", 1000.0))),
  Order(List(Item("Coffee", 3.0), Item("Tea", 2.0)))
)

// Вычисление общей стоимости всех заказов
val totalCost = for {
  order <- orders
  item <- order.items
} yield item.price
// totalCost = List(20.0, 5.0, 1000.0, 3.0, 2.0)

val sum = totalCost.sum // 1030.0
```

### Работа с несколькими коллекциями

```scala
val names = List("Alice", "Bob", "Charlie")
val ages = List(25, 30, 35)
val cities = List("New York", "London", "Paris")

// Создание пользователей
val users = for {
  name <- names
  age <- ages
  city <- cities
  if age > 25
} yield s"$name ($age) from $city"
// users = List("Alice (30) from New York", "Alice (30) from London", ...)
```

## Типичные ошибки

### 1. Неправильное использование yield

```scala
// Ошибка - yield в неправильном месте
val result = for {
  n <- List(1, 2, 3)
  yield n * 2 // Ошибка компиляции
}

// Правильно
val result = for {
  n <- List(1, 2, 3)
} yield n * 2
```

### 2. Забывание фильтров

```scala
// Ошибка - может вызвать исключение
val maybeName: Option[String] = None
val result = for {
  name <- maybeName
} yield name.toUpperCase // Не выполнится, но код выглядит небезопасно

// Правильно - с проверкой
val result = for {
  name <- maybeName
  if name.nonEmpty
} yield name.toUpperCase
```

### 3. Неправильный порядок генераторов

```scala
// Неэффективно - создает все комбинации
val result = for {
  n <- List(1, 2, 3, 4, 5)
  if n % 2 == 0
  m <- List(1, 2, 3)
} yield n * m

// Более эффективно
val result = for {
  n <- List(1, 2, 3, 4, 5) if n % 2 == 0
  m <- List(1, 2, 3)
} yield n * m
```

## Лучшие практики

### 1. Используйте for-comprehension для сложных операций

```scala
// Хорошо - читаемо
val result = for {
  user <- findUser(id)
  permissions <- getUserPermissions(user)
  if permissions.contains("admin")
  settings <- getUserSettings(user)
} yield UserInfo(user, permissions, settings)

// Избегайте - менее читаемо
val result = findUser(id)
  .flatMap(user => getUserPermissions(user))
  .filter(_.contains("admin"))
  .flatMap(permissions => getUserSettings(user).map(settings => UserInfo(user, permissions, settings)))
```

### 2. Используйте фильтры для раннего отсеивания

```scala
// Хорошо - ранняя фильтрация
val result = for {
  user <- users
  if user.isActive
  if user.age >= 18
  permission <- user.permissions
  if permission.isValid
} yield permission

// Избегайте - поздняя фильтрация
val result = for {
  user <- users
  permission <- user.permissions
} yield permission
```

### 3. Разбивайте сложные for-comprehension

```scala
// Хорошо - разбито на части
val activeUsers = for {
  user <- users
  if user.isActive
} yield user

val userPermissions = for {
  user <- activeUsers
  permission <- user.permissions
} yield permission

// Избегайте - слишком сложно
val result = for {
  user <- users
  if user.isActive
  permission <- user.permissions
  if permission.isValid
  setting <- user.settings
  if setting.isEnabled
} yield (user, permission, setting)
```

## Вопросы для собеседования

1. **В чем разница между for-comprehension и обычными циклами?**
2. **Как for-comprehension связан с монадами?**
3. **Когда следует использовать for-comprehension вместо map/flatMap?**
4. **Как работает for-comprehension с Option и Try?**
5. **В чем разница между for-comprehension с yield и без yield?**

## Дополнительные ресурсы

- [Scala Language Specification - For Comprehensions](https://scala-lang.org/files/archive/spec/2.13/06-expressions.html#for-comprehensions-and-for-loops)
- [Scala 3 Book - For Expressions](https://docs.scala-lang.org/scala3/book/control-structures.html)
- Martin Odersky - "Programming in Scala" (Chapter 23) 