---
title: "Java — Сборка и запуск (javac, jar, JVM)"
tags: [java, build, jvm, classpath, compilation, basics]
updated: 2026-03-04
---

# Сборка и запуск Java-приложений

Работа с Java через командную строку — фундаментальный навык для понимания процессов компиляции, запуска, отладки и упаковки приложений. Это полезно для:

- Быстрой диагностики проблем
- Автоматизации сборки
- Глубокого понимания JVM
- Работы на серверах без IDE
## Основные этапы работы с Java-программой

1. **Компиляция**: `.java` → `.class` (байт-код) с помощью `javac`
2. **Запуск**: выполнение байт-кода в JVM с помощью `java`

```bash
javac -d bin src/com/example/Main.java
java -classpath bin com.example.Main
```
## Переменная `classpath`

- Определяет, где JVM ищет классы и ресурсы
- Может включать папки и JAR-файлы
- Разделитель: `;` (Windows), `:` (Linux/macOS)

```bash
java -classpath bin;lib/my-lib.jar com.example.Main
```
> Если не указан, используется текущая директория (`.`)
## Компиляция: `javac`

- Преобразует `.java` в `.class`
- Создаёт AST, проверяет типы, генерирует байт-код

**Часто используемые параметры:**
- `-d <dir>` — куда складывать `.class`
- `-classpath <path>` — где искать классы/JAR
- `-g` — добавить отладочную информацию
- `-Xlint` — включить предупреждения

```bash
javac -d bin -classpath lib/my-lib.jar src/com/example/Main.java
```

**Процессоры аннотаций:**

```bash
javac -classpath lib/lombok.jar -processor lombok.launch.AnnotationProcessor src/Main.java
```
## Запуск: `java`

- Загружает классы, инициализирует, вызывает `main`

**Часто используемые параметры:**
- `-classpath` или `-cp` — где искать классы/JAR
- `-D<name>=<value>` — системные свойства
- `-jar <jarfile>` — запуск JAR
- `-Xms`, `-Xmx` — размер кучи

```bash
java -classpath bin com.example.Main
java -jar app.jar
java -Dfile.encoding=UTF-8 com.example.Main
```
## Отладка: `jdb`

- Пошаговое выполнение, точки останова, просмотр переменных

**Базовые команды:**
- `stop at <class>:<line>` — точка останова
- `run` — запуск
- `list` — показать строку
- `step`/`next` — шаги
- `print <var>` — вывести переменную
- `locals` — все локальные
- `cont` — продолжить

```bash
javac -g -d bin src/com/example/Main.java
jdb -classpath bin -sourcepath src com.example.Main
```
## Архивы: `jar`

- Создание и управление JAR-архивами

**Часто используемые параметры:**
- `c` — создать архив
- `v` — подробный вывод
- `f` — имя файла
- `m` — файл манифеста
- `-C <dir>` — сменить директорию

```bash
jar cvf app.jar -C bin .
# С манифестом:
jar cvmf manifest.mf app.jar -C bin .
```
**Манифест:**
```
Main-Class: com.example.Main
```
## Анализ байт-кода: `javap`

- Показывает структуру класса, байт-код, методы

**Часто используемые параметры:**
- `-c` — дизассемблировать
- `-p` — приватные члены
- `-classpath <path>`
- `-verbose` — подробности

```bash
javap -c -classpath calculator.jar com.example.Calculator
```
## Документация: `javadoc`

- Генерирует HTML-документацию из JavaDoc-комментариев

**Часто используемые параметры:**
- `-d <dir>` — куда сохранять
- `-sourcepath <path>`
- `-charset <charset>`
- `-author`
- `-subpackages <pkg>`

```bash
javadoc -d doc -charset utf-8 -sourcepath src -author -subpackages com.example
```
## Прочие утилиты JVM (1-2 предложения и пример)

- `jps` — список JVM-процессов: `jps`
- `jstat` — статистика памяти/GC: `jstat -gc <pid>`
- `jstack` — стек вызовов: `jstack <pid>`
- `jmap` — дамп памяти: `jmap -dump:file=heap.hprof <pid>`
## Связанные темы

- [[Java Memory Structure]] — как JVM использует память при выполнении
- [[Java Reflection API]] — динамическая загрузка классов
- [[Корпоративная социальная сеть - микросервисная архитектура, выбор языка и идеи для вовлечённости|Корпоративная социальная сеть]] — сборка Spring Boot сервисов

# Вопросы для собеседования по компиляции и сборке Java

1. Как работает компиляция Java кода? Какие этапы включает?
2. Что такое JIT компиляция? Как она работает?
3. Чем отличается javac от JIT компилятора?
4. Как работает байткод в Java? Что такое .class файлы?
5. Что такое JVM? Как она выполняет байткод?
6. Как работает интерпретация vs компиляция в Java?
7. Что такое HotSpot JVM? Какие оптимизации она выполняет?
8. Как работает AOT (Ahead-of-Time) компиляция?
9. Что такое GraalVM? Чем отличается от стандартной JVM?
10. Как работает инкрементальная компиляция?
11. Что такое bytecode verification? Зачем она нужна?
12. Как работает class loading в Java?
13. Что такое Bootstrap, Extension, Application ClassLoaders?
14. Как работает dynamic linking в Java?
15. Что такое reflection? Как оно связано с компиляцией?
16. Как работает annotation processing?
17. Что такое build tools (Maven, Gradle)? Как они работают?
18. Как работает dependency resolution?
19. Что такое incremental compilation в IDE?
20. Как работает debugging в Java? Что такое debug symbols?
21. Что такое profiling? Как профилировать Java приложения?
22. Как работает optimization в JVM?
23. Что такое escape analysis? Как оно влияет на производительность?
24. Как работает inlining в JVM?
25. Какие проблемы могут возникнуть при компиляции больших проектов?