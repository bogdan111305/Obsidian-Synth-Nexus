Tags: #scala

## Неявные параметры

Неявные параметры — это параметры, которые могут быть автоматически переданы в функцию из контекста ее вызова. Для этого в нем должны быть однозначно определены и помечены ключевым словом implicit переменные соответствующих типов.

``` Scala
def printContext(implicit ctx: Context) = println(ctx.name)

implicit val ctx = Context("Hello world")

printContext
```

Выведет:

``` Scala
Hello world
```

В методе printContext мы неявно получаем переменную типа Context и печатаем содержимое ее поля name. Пока не страшно.

Механизм разрешения неявных параметров поддерживает обобщенные типы.  

``` Scala
case class Context[T](message: String)

def printContextAwared[T](x: T)(implicit ctx: Context[T]) = println(s"${ctx.message}: $x")

implicit val ctxInt = Context[Int]("This is Integer")
implicit val ctxStr = Context[String]("This is String")

printContextAwared(1)
printContextAwared("string")
```

Выведет:

``` Scala
This is Integer: 1
This is String: string
```

Этот код эквивалентен тому, как если бы мы явно передавали в метод printContextAwared параметры ctxInt в первом случае и ctxString во втором.

``` Scala
printContextAwared(1)(ctxInt)
printContextAwared("string")(ctxStr)
```

Что интересно, неявные параметры не обязательно должны быть полями, они могут быть методами.

``` Scala
implicit def dateTime: LocalDateTime = LocalDateTime.now()

def printCurrentDateTime(implicit dt: LocalDateTime) = println(dt.toString)

printCurrentDateTime
Thread.sleep(1000)
printCurrentDateTime
```

Выведет:

``` Scala
2017-05-27T16:30:49.332
2017-05-27T16:30:50.476
```

Более того, неявные параметры-функции могут, в свою очередь, принимать неявные параметры.

``` Scala
implicit def dateTime(implicit zone: ZoneId): ZonedDateTime = ZonedDateTime.now(zone)

def printCurrentDateTime(implicit dt: ZonedDateTime) = println(dt.toString)

implicit val utc = ZoneOffset.UTC

printCurrentDateTime
```

Выведет:

``` Scala
2017-05-28T07:07:27.322Z
```
## Неявные преобразования

Неявные преобразования позволяют автоматически преобразовывать значения одного типа к другому.  
Чтобы задать неявное преобразование вам нужно определить функцию от одного явного аргумента и пометить ее ключевым словом implicit.

``` Scala
case class A(i: Int)
case class B(i: Int)

implicit def aToB(a: A): B = B(a.i)

val a = A(1)
val b: B = a
println(b)
```

Выведет:

``` Scala
B(1)
```

Все что справедливо для неявных параметров-функций, справедливо также и для неявных преобразований: поддерживаются обобщенные типы, должен быть только один явный, но может быть сколько угодно неявных параметров и т. п.

``` Scala
case class A(i: Int)
case class B(i: Int)
case class PrintContext[T](t: String)

implicit def aToB(a: A): B = B(a.i)
implicit val cContext: PrintContext[B] = PrintContext("The value of type B is")

def printContextAwared[T](t: T)(implicit ctx: PrintContext[T]): Unit = println(s"${ctx.t}: $t")

val a = A(1)
printContextAwared[B](a)
```

_Ограничения_  
Scala не допускает применение нескольких неявных преобразований подряд, таким образом код:

``` Scala
case class A(i: Int)
case class B(i: Int)
case class C(i: Int)

implicit def aToB(a: A): B = B(a.i)
implicit def bToC(b: B): C = C(b.i)

val a = A(1)
val c: C = a
```

Не скомпилируется.  
Тем не менее, как мы уже убедились, Scala не запрещает искать неявные _параметры_ по цепочке, так что мы можем исправить этот код следующим образом:

``` Scala
case class A(i: Int)
case class B(i: Int)
case class C(i: Int)

implicit def aToB(a: A): B = B(a.i)
implicit def bToC[T](t: T)(implicit tToB: T => B): C = C(t.i)

val a = A(1)
val c: C = a
```

Стоит заметить, что если функция принимает значение неявно, то в ее теле оно будет видимо как неявное значение или преобразование. В предыдущем примере для объявления метода bToC, tToB является неявным параметром и при этом внутри метода работает уже как неявное преобразование.
## Неявные классы

Ключевое слово implicit перед объявлением класса — это более компактная форма записи неявного преобразования значения аргумента конструктора к данному классу.

``` Scala
implicit class ReachInt(self: Int) {
  def fib: Int =
    self match {
      case 0 | 1 => 1
      case i => (i - 1).fib + (i - 2).fib
    }
}

println(5.fib)
```

Выведет:

``` Scala
5
```

Может показаться, что неявные классы это всего лишь способ примешивания функционала к классу, но на самом это понятие несколько шире.  

``` Scala
sealed trait Animal
case object Dog extends Animal
case object Bear extends Animal
case object Cow extends Animal

case class Habitat[A <: Animal](name: String)

implicit val dogHabitat = Habitat[Dog.type]("House")
implicit val bearHabitat = Habitat[Bear.type]("Forest")

implicit class AnimalOps[A <: Animal](animal: A) {
  def getHabitat(implicit habitat: Habitat[A]): Habitat[A] = habitat
}

println(Dog.getHabitat)
println(Bear.getHabitat)
//Не скомпилируется:
//println(Cow.getHabitat)
```

Выведет:

``` Scala
Habitat(House)
Habitat(Forest)
```

Здесь в неявном классе AnimalOps мы объявляем что тип значения, к которому он будет применен, будет виден нам как A, затем в методе getHabitat мы требуем неявный параметр Habitat[A]. При его отсутствии, как в строчке с Cow, мы получим ошибку компиляции.

Не прибегая к помощи неявных классов, достичь такого же эффекта нам бы мог помочь [F-bounded polymorphism](http://www.alessandrolacava.com/blog/scala-self-recursive-types/):

``` Scala
sealed trait Animal[A <: Animal[A]] { self: A =>
  def getHabitat(implicit habitat: Habitat[A]): Habitat[A] = habitat
}

trait Dog extends Animal[Dog]
trait Bear extends Animal[Bear]
trait Cow extends Animal[Cow]

case object Dog extends Dog
case object Bear extends Bear
case object Cow extends Cow

case class Habitat[A <: Animal[A]](name: String)

implicit val dogHabitat = Habitat[Dog]("House")
implicit val bearHabitat = Habitat[Bear]("Forest")

println(Dog.getHabitat)
println(Bear.getHabitat)
```

Как видно, в этом случае у типа Animal существенно усложнилось объявление, появился дополнительный рекурсивный параметр A, который играет исключительно служебную роль. Это сбивает с толку.