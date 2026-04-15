# Сборка и запуск Java-приложений

> `.java` → `javac` → `.class` (байт-код) → JVM (`java`) → JIT → машинный код. `classpath` — список мест где JVM ищет `.class` файлы. `jar` — ZIP-архив с байт-кодом и `MANIFEST.MF`. `javap` — декомпилятор байт-кода.
> На интервью: что делает `javac` (AST → type check → bytecode), чем classpath отличается от modulepath (Java 9+), ограничения `redefineClasses` при hot reload.

## Связанные темы
[[Java Memory Structure]], [[ClassLoaders]], [[Java Reflection API]]

---

## Компиляция: javac

Компилирует `.java` → `.class`. Процесс: парсинг → AST → проверка типов → генерация байт-кода.

```bash
javac -d bin src/com/example/Main.java
javac -d bin -classpath lib/my-lib.jar src/com/example/Main.java

# Ключевые флаги:
# -d <dir>        куда складывать .class
# -classpath <p>  где искать зависимости
# -g              отладочная информация (номера строк, имена переменных)
# -Xlint:all      все предупреждения
# --release 21    целевая версия JVM (cross-compilation)
```

**Процессоры аннотаций (Lombok, MapStruct):**
```bash
javac -classpath lib/lombok.jar -processor lombok.launch.AnnotationProcessor src/Main.java
# Или через Maven/Gradle annotationProcessor configuration
```

---

## Запуск: java

```bash
java -classpath bin com.example.Main
java -jar app.jar
java -Dfile.encoding=UTF-8 -Xms512m -Xmx2g com.example.Main

# -cp / -classpath  где искать .class и .jar
# -D<name>=<value>  системные свойства (System.getProperty)
# -Xms / -Xmx      начальный/максимальный heap
# -jar              запуск исполняемого JAR (Main-Class из MANIFEST.MF)
```

**classpath:** список через `:` (Linux/macOS) или `;` (Windows). Если не указан — текущая директория.

---

## Архивы: jar

```bash
jar cvf app.jar -C bin .          # создать JAR из содержимого bin/
jar cvmf manifest.mf app.jar -C bin .  # с кастомным манифестом
jar tf app.jar                    # содержимое архива
jar xf app.jar                    # распаковать

# MANIFEST.MF для исполняемого JAR:
# Main-Class: com.example.Main
# Class-Path: lib/dependency.jar
```

---

## Анализ байт-кода: javap

```bash
javap -c com.example.Calculator             # дизассемблировать (байт-код)
javap -p -verbose com.example.MyClass       # приватные методы + детали
javap -classpath calculator.jar com.example.Calculator

# Полезно для:
# - Проверить что javac сгенерировал (bridge methods, try-with-resources)
# - Увидеть invokedynamic для лямбд
# - Отладить generics type erasure
```

---

## Прочие JVM утилиты

```bash
jps                        # список JVM-процессов (аналог ps для Java)
jstat -gcutil <pid> 1s     # метрики GC в реальном времени
jstack <pid>               # thread dump (стеки всех потоков)
jmap -dump:live,format=b,file=heap.hprof <pid>  # heap dump
jcmd <pid> help            # все доступные команды для процесса
```

---

## JPMS (Java 9+): Модульная система

```bash
# Скомпилировать модуль:
javac --module-source-path src -d mods --module com.example.app

# Запустить модульное приложение:
java --module-path mods -m com.example.app/com.example.Main

# module-info.java:
# module com.example.app {
#     requires java.sql;
#     exports com.example.api;
# }
```

**classpath vs modulepath:** modulepath применяет модульные границы (strong encapsulation), classpath — нет. `--add-opens` для обхода инкапсуляции (рефлексия в сторонних библиотеках).

---

## Вопросы на интервью

- Что делает `javac` внутри? (парсинг → AST → type checking → bytecode generation)
- Чем `-classpath` отличается от `--module-path`?
- Что такое `MANIFEST.MF`? Какие ключевые поля он содержит?
- Зачем нужен `-g` флаг при компиляции?
- Как `javap` помогает при отладке лямбд и generic-кода?

---

## Подводные камни

- **`-Xlint` по умолчанию выключен** — компилятор молчит о deprecated API, unchecked casts, unused imports. Добавляй `-Xlint:all -Werror` в CI для обнаружения проблем на стадии компиляции.
- **classpath порядок важен** — если одноимённый класс есть в двух JAR, побеждает первый в classpath. Причина странных `NoSuchMethodError` при конфликте версий.
- **`--release` vs `-source/-target`** — `-source 8 -target 8` не гарантирует, что код не использует API из Java 11. `--release 8` проверяет как сигнатуры, так и API.
- **Fat JAR и Class-Path манифест** — при `java -jar` classpath из командной строки игнорируется, используется только `Class-Path` из манифеста. Spring Boot Executable JAR обходит это через кастомный ClassLoader.
- **Анонимные классы в hot reload** — `redefineClasses` не может добавлять новые классы. Если лямбда компилируется в новый синтетический метод → горячая замена не работает. JRebel/DCEVM используют другой подход (новый ClassLoader).
