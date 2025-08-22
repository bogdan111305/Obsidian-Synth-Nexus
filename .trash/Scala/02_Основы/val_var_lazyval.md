# Переменные в Scala: val, var и lazy val

## Обзор

В Scala есть три основных способа объявления переменных:
- `val` - неизменяемая переменная (immutable)
- `var` - изменяемая переменная (mutable) 
- `lazy val` - ленивая неизменяемая переменная

## val - Неизменяемые переменные

`val` объявляет неизменяемую переменную. После присвоения значения его нельзя изменить.

```scala
val name = "Scala"
val age: Int = 25
val isActive = true

// Попытка изменить val приведет к ошибке компиляции
// name = "Java" // Ошибка: reassignment to val
```

### Преимущества val

1. **Безопасность потоков** - val безопасны в многопоточных приложениях
2. **Предсказуемость** - значение не может измениться неожиданно
3. **Функциональный стиль** - поощряет иммутабельность

```scala
val numbers = List(1, 2, 3, 4, 5)
val doubled = numbers.map(_ * 2) // Создается новый список
// numbers остается неизменным
```

## var - Изменяемые переменные

`var` объявляет изменяемую переменную. Значение можно изменять после объявления.

```scala
var counter = 0
var message = "Hello"
var temperature: Double = 20.5

// Можно изменять значение
counter += 1
message = "Hello, Scala!"
temperature = 25.0
```

### Когда использовать var

1. **Локальные переменные в циклах**
```scala
var sum = 0
for (i <- 1 to 10) {
  sum += i
}
```

2. **Состояние в императивном коде**
```scala
var currentUser: Option[User] = None
def login(user: User): Unit = {
  currentUser = Some(user)
}
```

3. **Взаимодействие с Java API**
```scala
var javaList = new java.util.ArrayList[String]()
javaList.add("item")
```

### Рекомендации по использованию var

- Избегайте var в функциональном коде
- Используйте только когда действительно необходимо
- Ограничивайте область видимости var

## lazy val - Ленивые переменные

`lazy val` вычисляется только при первом обращении к ней.

```scala
lazy val expensiveComputation = {
  println("Выполняется дорогостоящее вычисление...")
  Thread.sleep(1000)
  42
}

// expensiveComputation еще не вычислено
println("Начинаем работу...")

// Только сейчас выполнится вычисление
val result = expensiveComputation
```

### Преимущества lazy val

1. **Отложенное вычисление** - вычисляется только при необходимости
2. **Кэширование** - результат сохраняется после первого вычисления
3. **Избежание циклических зависимостей**

```scala
class DatabaseConnection {
  lazy val connection = {
    // Создание соединения с базой данных
    createConnection()
  }
}
```

### Примеры использования lazy val

```scala
// Ленивая загрузка конфигурации
lazy val config = {
  val file = scala.io.Source.fromFile("config.properties")
  try {
    parseConfig(file.getLines().toMap)
  } finally {
    file.close()
  }
}

// Ленивое вычисление в коллекциях
lazy val filteredData = largeDataset.filter(_.isValid).map(_.transform)
```

## Сравнение с Java

| Scala | Java | Описание |
|-------|------|----------|
| `val` | `final` | Неизменяемая переменная |
| `var` | обычная переменная | Изменяемая переменная |
| `lazy val` | нет аналога | Ленивая неизменяемая переменная |

```java
// Java
final String name = "Java";
String message = "Hello";
// Нет прямого аналога lazy val
```

## Типичные ошибки

### 1. Попытка изменить val
```scala
val x = 10
x = 20 // Ошибка компиляции
```

### 2. Использование var в функциональном коде
```scala
// Плохо
var sum = 0
list.foreach(sum += _)

// Хорошо
val sum = list.sum
```

### 3. Неправильное использование lazy val
```scala
// Плохо - вычисление может не произойти
lazy val result = expensiveOperation()
if (condition) {
  println(result) // Может не выполниться
}

// Хорошо - явное вычисление
val result = if (condition) expensiveOperation() else 0
```

## Лучшие практики

1. **Предпочитайте val** - используйте неизменяемые переменные по умолчанию
2. **Ограничивайте var** - используйте только когда необходимо
3. **Используйте lazy val для дорогостоящих операций**
4. **Явно указывайте типы** для сложных выражений

```scala
// Хорошо
val users: List[User] = loadUsers()
val activeUsers = users.filter(_.isActive)

// Избегайте
var userList = List[User]()
for (user <- allUsers) {
  if (user.isActive) {
    userList = user :: userList
  }
}
```

## Вопросы для собеседования

1. **В чем разница между val, var и lazy val?**
2. **Когда следует использовать var?**
3. **Какие преимущества дает lazy val?**
4. **Как val связан с функциональным программированием?**
5. **Можно ли использовать var в многопоточных приложениях?**

## Дополнительные ресурсы

- [Scala Language Specification - Variables](https://scala-lang.org/files/archive/spec/2.13/04-basic-declarations-and-definitions.html)
- [Scala 3 Book - Variables](https://docs.scala-lang.org/scala3/book/first-steps.html)
- Martin Odersky - "Programming in Scala" (Chapter 2) 