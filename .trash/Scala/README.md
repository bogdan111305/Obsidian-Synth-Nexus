# Scala: Полное руководство и справочник

## О языке Scala

**Scala** (Scalable Language) — это современный мультипарадигменный язык программирования, который элегантно объединяет объектно-ориентированное и функциональное программирование. Созданный Мартином Одерски в 2003 году, Scala работает на виртуальной машине Java (JVM) и обеспечивает полную совместимость с Java.

### Ключевые особенности Scala

- **Мультипарадигменность**: Поддержка ООП и ФП в одном языке
- **Статическая типизация**: Мощная система типов с выводом типов
- **JVM совместимость**: Полная интеграция с Java экосистемой
- **Краткость**: Выразительный синтаксис, требующий меньше кода
- **Масштабируемость**: От скриптов до крупных распределенных систем
- **Безопасность**: Неизменяемость и безопасность типов по умолчанию

## Структура обучения

### 📚 Фундаментальные основы

#### [01. Введение в Scala](./01_Введение/)
- История и философия языка
- Установка и настройка среды разработки
- Первые шаги: Hello World
- Scala 2 vs Scala 3: ключевые различия
- Экосистема и инструменты

#### [02. Основы языка](./02_Основы/)
- Переменные: val, var, lazy val
- Система типов и литералы
- Выражения и блоки кода
- Условия и циклы
- Функции и методы
- Pattern matching
- For-comprehension

#### [03. Коллекции](./03_Коллекции/)
- Обзор коллекций Scala
- List, Vector, Set, Map
- Иммутабельные vs мутабельные коллекции
- Функциональные операции (map, filter, fold)
- Option, Either, Try
- Работа с потоками данных

### 🏗️ Объектно-ориентированное программирование

#### [04. ООП в Scala](./04_ООП/)
- Классы и объекты
- Case классы и companion objects
- Наследование и полиморфизм
- Трейты и миксины
- Линеаризация трейтов
- Модули и пакеты

### 🔧 Функциональное программирование

#### [05. Функциональное программирование](./05_Функциональное_программирование/)
- Функции как значения первого класса
- Функции высшего порядка
- Каррирование и частичное применение
- Частичные функции
- Монады и функторы
- Композиция функций
- Чистые функции и побочные эффекты

### 🎯 Продвинутые темы

#### [06. Система типов](./06_Типы/)
- Параметрические типы (Generic)
- Вариантность (Variance)
- Higher-kinded types
- Path-dependent types
- Structural types
- Opaque types (Scala 3)
- Union и intersection types (Scala 3)
- Enum и sealed traits

#### [07. Implicit система и расширения](./07_Implicit_и_расширения/)
- Implicit parameters и conversions (Scala 2)
- Given/using система (Scala 3)
- Extension methods
- Context functions
- Type class pattern
- Derivation
- Inline и макросы

#### [08. Scala 3 - Новые возможности](./08_Scala3/)
- Enum классы
- Opaque types
- Match types
- Inline функции и макросы
- Extension methods
- Context functions
- Top-level definitions
- Indentation-based syntax

### ⚡ Специализированные темы

#### [09. Конкурентность и параллелизм](./09_Конкурентность/)
- Future и Promise
- ExecutionContext
- Akka и Actor model
- Cats Effect и ZIO
- Параллельные коллекции
- Лучшие практики многопоточности

#### [10. Ввод-вывод и работа с данными](./10_IO/)
- Работа с файлами
- JSON обработка (circe, play-json)
- XML и CSV
- Сериализация и десериализация
- Работа с базами данных

#### [11. Тестирование](./11_Тестирование/)
- ScalaTest
- Specs2
- Property-based testing (ScalaCheck)
- Моки и интеграционное тестирование
- Лучшие практики тестирования

#### [12. Интеграция с Java](./12_Java_интеграция/)
- Вызов Java из Scala и наоборот
- Использование Java библиотек
- SAM (Single Abstract Method)
- Interoperability лучшие практики

### 🛠️ Инструменты и экосистема

#### [13. Инструменты разработки](./13_Инструменты/)
- SBT (Scala Build Tool)
- Scala CLI
- Ammonite REPL
- IDE: IntelliJ IDEA, Metals, VSCode
- Scaladoc
- Профилирование и отладка

#### [14. Популярные библиотеки](./14_Библиотеки/)
- **Функциональное программирование**: Cats, Cats Effect, Scalaz
- **Веб-фреймворки**: Play, Akka HTTP, http4s
- **Работа с данными**: Spark, Slick, Doobie
- **JSON**: circe, play-json, upickle
- **Тестирование**: ScalaTest, Specs2, ScalaCheck
- **Утилиты**: Shapeless, Monocle, Refined

### 🚀 Практика и проекты

#### [15. Практические примеры](./15_Практика/)
- Консольные приложения
- Веб-приложения
- RESTful API
- Микросервисы
- Обработка данных
- Функциональные архитектуры

#### [16. Архитектурные паттерны](./16_Архитектура/)
- Layered architecture
- Hexagonal architecture
- Event sourcing
- CQRS
- Reactive patterns
- Функциональные архитектуры

### 📖 Справочные материалы

#### [17. Best Practices](./17_Best_Practices/)
- Стиль кода и соглашения
- Функциональное программирование
- Производительность
- Безопасность
- Антипаттерны и типичные ошибки

#### [18. Вопросы для собеседования](./18_Собеседование/)
- Junior уровень
- Middle уровень
- Senior уровень
- Практические задачи
- Системное проектирование

#### [19. Ресурсы для изучения](./19_Ресурсы/)
- Официальная документация
- Книги
- Онлайн курсы
- Сообщества и форумы
- Блоги и статьи

## Траектории обучения

### 🟢 Начинающий разработчик
1. **Введение** → **Основы языка** → **Коллекции** → **ООП**
2. Практика: простые консольные приложения
3. **Функциональное программирование** (базовые концепции)

### 🟡 Средний уровень
1. **Система типов** → **Implicit/Given** → **Тестирование**
2. **Scala 3** новые возможности
3. Практика: веб-приложения, работа с данными

### 🔴 Продвинутый уровень
1. **Конкурентность** → **Архитектурные паттерны**
2. **Популярные библиотеки** → **Продвинутые техники ФП**
3. Практика: микросервисы, высоконагруженные системы

## Полезные ссылки

### Официальная документация
- [Scala Language](https://scala-lang.org/)
- [Scala 3 Book](https://docs.scala-lang.org/scala3/book/introduction.html)
- [API Documentation](https://www.scala-lang.org/api/)
- [Language Specification](https://scala-lang.org/files/archive/spec/2.13/)

### Инструменты
- [SBT](https://www.scala-sbt.org/)
- [Scala CLI](https://scala-cli.virtuslab.org/)
- [Ammonite](https://ammonite.io/)
- [Metals](https://scalameta.org/metals/)

### Сообщество
- [Scala Users Forum](https://users.scala-lang.org/)
- [Discord Scala](https://discord.com/invite/scala)
- [Reddit r/scala](https://www.reddit.com/r/scala/)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/scala)

### Книги (топ-10)
1. **"Programming in Scala"** - Martin Odersky, Lex Spoon, Bill Venners
2. **"Functional Programming in Scala"** - Paul Chiusano, Runar Bjarnason
3. **"Scala for the Impatient"** - Cay Horstmann
4. **"Essential Scala"** - Noel Welsh, Dave Gurnell
5. **"Scala with Cats"** - Noel Welsh, Dave Gurnell
6. **"Advanced Scala with Cats"** - Noel Welsh, Dave Gurnell
7. **"Hands-on Scala Programming"** - Li Haoyi
8. **"Akka in Action"** - Raymond Roestenburg, Rob Bakker, Rob Williams
9. **"Scala Cookbook"** - Alvin Alexander
10. **"Scala Puzzlers"** - Andrew Phillips, Nermin Serifovic

---

> **Примечание**: Этот справочник покрывает как Scala 2, так и Scala 3, с акцентом на современные практики и подходы. Каждый раздел содержит подробные объяснения, примеры кода, лучшие практики и вопросы для самопроверки.

**Последнее обновление**: 2024
**Версии Scala**: 2.13.x, 3.x
