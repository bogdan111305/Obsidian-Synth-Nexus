Tags: #java #jvm #cmd #commandline #javac #java-tools

# Java в командной строке: Полное руководство

Работа с Java через командную строку — это мощный способ компиляции, запуска и отладки приложений без использования IDE. Она позволяет глубже понять процесс работы JVM и взаимодействие с различными утилитами, такими как `javac`, `java`, `jdb`, `jar`, `javap` и `javadoc`. В этой статье мы подробно разберем этапы компиляции и запуска, настройку `classpath`, отладку, создание JAR-архивов, анализ байт-кода и генерацию документации. Статья ориентирована как на начинающих, так и на опытных разработчиков, желающих углубить свои знания.

## 1. Этапы работы с Java-программами

Запуск Java-приложения включает два основных этапа:

1. **Компиляция**: Преобразование исходного кода (`.java`) в байт-код (`.class`) с помощью утилиты `javac`.
2. **Запуск**: Исполнение байт-кода в JVM с помощью утилиты `java`.

Для правильной работы обеих утилит JVM должна знать, где искать классы, что задается через переменную окружения или параметр `classpath`.

## 2. Переменная `classpath`

**`classpath`** определяет пути, по которым JVM ищет скомпилированные классы (`.class`) и ресурсы (например, JAR-файлы). Это может быть:

- Папка со скомпилированными файлами.
- JAR-архивы.
- Несколько путей, разделенных `;` (Windows) или `:` (Linux/macOS).

**Пример задания `classpath`**:

```bash
java -classpath bin;lib/my-lib.jar com.example.Main
```

**Заметка**: Если `classpath` не указан, JVM использует текущую директорию (`.`). Для сложных проектов рекомендуется использовать инструменты сборки (`Maven`, `Gradle`), но понимание `classpath` важно для работы в командной строке.

## 3. Утилита `javac` (Компиляция)

`javac` — это компилятор Java, который преобразует исходные файлы (`.java`) в байт-код (`.class`), понятный JVM. Компилятор создает **AST (Abstract Syntax Tree)** для анализа кода и проверки типов, после чего генерирует байт-код.

### Синтаксис команды

```bash
javac [options] [sourcefiles] [classes] [@argfiles]
```

- **options**: Параметры компиляции (например, `-d`, `-g`).
- **sourcefiles**: Исходные файлы (`.java`), которые нужно скомпилировать.
- **classes**: Классы для обработки аннотаций (используется с процессорами аннотаций).
- **@argfiles**: Файлы с аргументами для компилятора (удобно для больших проектов).

### Основные параметры `javac`

|Параметр|Описание|Пример|
|---|---|---|
|`-d <dir>`|Директория для сохранения скомпилированных `.class` файлов.|`javac -d bin src/com/example/Main.java`|
|`-classpath <path>`|Путь к классам или JAR-файлам.|`javac -classpath lib/my-lib.jar src/Main.java`|
|`-g`|Генерирует отладочную информацию (переменные, строки кода).|`javac -g src/Main.java`|
|`-sourcepath <path>`|Путь к исходным файлам.|`javac -sourcepath src src/com/example/Main.java`|
|`-processor <class>`|Указывает процессор аннотаций.|`javac -processor MyProcessor src/Main.java`|
|`-Xlint`|Включает предупреждения компилятора.|`javac -Xlint src/Main.java`|

### Пример компиляции

```bash
# Компилируем файл Main.java в папку bin
javac -d bin src/com/example/Main.java
```

**Заметка**: Если класс находится в пакете (например, `com.example.Main`), структура директорий в `bin` будет соответствовать структуре пакета (`bin/com/example/Main.class`).

### Процессоры аннотаций

Если вы используете процессоры аннотаций (например, для генерации кода с помощью Lombok), их нужно указать на этапе компиляции. Зависимости для процессоров добавляются через `classpath`:

```bash
javac -classpath lib/lombok.jar -processor lombok.launch.AnnotationProcessor src/Main.java
```

Для проектов с `Maven` или `Gradle` процессоры аннотаций настраиваются в конфигурации сборки.

## 4. Утилита `java` (Запуск)

`java` — это утилита для запуска скомпилированного байт-кода в JVM. Она загружает классы, инициализирует их и вызывает метод `main`.

### Синтаксис команды

```bash
java [options] <mainclass> [args]
```

- **options**: Параметры JVM (например, `-classpath`, `-D`).
- **mainclass**: Полное имя класса с методом `main` (например, `com.example.Main`).
- **args**: Аргументы командной строки, передаваемые в `main`.

### Основные параметры `java`

|Параметр|Описание|Пример|
|---|---|---|
|`-classpath <path>`|Путь к классам или JAR-файлам.|`java -classpath bin com.example.Main`|
|`-cp <path>`|Сокращение для `-classpath`.|`java -cp bin com.example.Main`|
|`-D<name>=<value>`|Устанавливает системное свойство.|`java -Dapp.config=prod com.example.Main`|
|`-jar <jarfile>`|Запускает JAR-файл с указанным `Main-Class`.|`java -jar app.jar`|
|`-Xms<size>`, `-Xmx<size>`|Устанавливает начальный и максимальный размер кучи.|`java -Xms512m -Xmx2g com.example.Main`|
|`-verbose:class`|Выводит информацию о загрузке классов.|`java -verbose:class com.example.Main`|

### Пример запуска

```bash
# Запуск класса Main из пакета com.example
java -classpath bin com.example.Main
```

**Важно**: Имя класса должно включать полный путь пакета (например, `com.example.Main`), а `classpath` указывает на корневую папку, содержащую структуру пакетов.

### Системные свойства

Системные свойства (`-D`) задают конфигурацию для приложения:

```bash
java -Dfile.encoding=UTF-8 -Dlog.level=DEBUG com.example.Main
```

Эти свойства доступны в коде через `System.getProperty("name")`.

### Пример с JAR

```bash
java -jar calculator.jar
```

## 5. Отладка с `jdb`

`jdb` — это отладчик командной строки для Java, позволяющий пошагово выполнять код, устанавливать точки останова и инспектировать переменные.

### Подготовка к отладке

1. **Компиляция с отладочной информацией**:  
    Используйте флаг `-g` для включения информации о переменных и строках кода:
    
    ```bash
    javac -g -d bin src/com/example/Main.java
    ```
    
2. **Запуск `jdb`**:  
    Укажите `classpath` для скомпилированных классов и `sourcepath` для исходных файлов:
    
    ```bash
    jdb -classpath bin -sourcepath src com.example.Main
    ```
    

### Основные команды `jdb`

|Команда|Описание|Пример|
|---|---|---|
|`stop at <class>:<line>`|Устанавливает точку останова.|`stop at com.example.Main:10`|
|`run`|Запускает выполнение программы.|`run`|
|`list`|Показывает текущую строку кода.|`list`|
|`step`|Выполняет следующий шаг (входит в методы).|`step`|
|`next`|Выполняет следующий шаг (не входит в методы).|`next`|
|`print <variable>`|Выводит значение переменной.|`print x`|
|`locals`|Показывает все локальные переменные.|`locals`|
|`cont`|Продолжает выполнение до следующей точки останова.|`cont`|

### Пример отладки

```bash
# Компиляция с отладочной информацией
javac -g -d bin src/com/example/Calculator.java

# Запуск jdb
jdb -classpath bin -sourcepath src com.example.Calculator

# Установка точки останова
> stop at com.example.Calculator:9

# Запуск программы
> run

# Показать текущую строку
main[1] list

# Выполнить шаг
main[1] step

# Вывести значение переменной
main[1] print result
```

**Заметка**: Для сложных приложений рекомендуется использовать отладчики в IDE (например, IntelliJ IDEA, Eclipse), но `jdb` полезен для минималистичных сценариев или серверов без GUI.

## 6. Утилита `jar` (Создание архивов)

`jar` — это утилита для создания и управления JAR-архивами, которые содержат скомпилированные классы и ресурсы.

### Синтаксис команды

```bash
jar [options] [jar-file] [manifest] [-C dir] files
```

### Основные параметры `jar`

|Параметр|Описание|Пример|
|---|---|---|
|`c`|Создает новый JAR-архив.|`jar cvf app.jar -C bin .`|
|`v`|Выводит подробную информацию.|`jar cvf app.jar ...`|
|`f`|Указывает имя JAR-файла.|`jar cvf app.jar ...`|
|`m`|Указывает файл манифеста.|`jar cvmf manifest.mf app.jar ...`|
|`-C <dir>`|Изменяет текущую директорию для архивирования.|`jar cvf app.jar -C bin .`|

### Пример создания JAR

```bash
# Создание JAR из папки bin
jar cvf calculator.jar -C bin .
```

### Манифест

JAR-архив может содержать файл манифеста (`META-INF/MANIFEST.MF`), указывающий основной класс:

```plaintext
Main-Class: com.example.Main
```

**Создание JAR с манифестом**:

```bash
jar cvmf manifest.mf app.jar -C bin .
```

**Запуск JAR**:

```bash
java -jar app.jar
```

## 7. Утилита `javap` (Дизассемблер)

`javap` позволяет анализировать байт-код скомпилированных классов, показывая их структуру, методы и инструкции.

### Основные параметры `javap`

|Параметр|Описание|Пример|
|---|---|---|
|`-c`|Дизассемблирует байт-код.|`javap -c com.example.Main`|
|`-p`|Показывает приватные члены.|`javap -p com.example.Main`|
|`-classpath <path>`|Путь к классам или JAR.|`javap -classpath app.jar com.example.Main`|
|`-verbose`|Выводит подробную информацию.|`javap -verbose com.example.Main`|

### Пример

```bash
# Дизассемблирование класса из JAR
javap -c -classpath calculator.jar com.example.Calculator
```

**Вывод** (пример):

```plaintext
public class com.example.Calculator {
  public com.example.Calculator();
    Code:
       0: aload_0
       1: invokespecial #1  // Method java/lang/Object."<init>":()V
       4: return

  public int add(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: iadd
       3: ireturn
}
```

**Когда использовать**: Для анализа байт-кода, оптимизации или изучения работы JVM.

## 8. Утилита `javadoc` (Генерация документации)

`javadoc` создает HTML-документацию на основе JavaDoc-комментариев в исходном коде.

### Синтаксис команды

```bash
javadoc [options] [packagenames] [sourcefiles] [@files]
```

### Основные параметры `javadoc`

|Параметр|Описание|Пример|
|---|---|---|
|`-d <dir>`|Директория для сохранения документации.|`javadoc -d doc ...`|
|`-sourcepath <path>`|Путь к исходным файлам.|`javadoc -sourcepath src ...`|
|`-charset <charset>`|Кодировка документации.|`javadoc -charset utf-8 ...`|
|`-author`|Включает информацию об авторе.|`javadoc -author ...`|
|`-subpackages <pkg>`|Обрабатывает подпакеты.|`javadoc -subpackages com.example ...`|

### Пример

```java
package com.example;

/**
 * Класс для выполнения математических операций.
 * @author Ivan Motorin
 */
public class Calculator {
    /**
     * Складывает два числа.
     * @param a Первое число
     * @param b Второе число
     * @return Сумма чисел
     */
    public int add(int a, int b) {
        return a + b;
    }
}
```

```bash
# Генерация документации
javadoc -d doc -charset utf-8 -sourcepath src -author -subpackages com.example
```

**Результат**: HTML-файлы с документацией в папке `doc`.

## 9. Прочие утилиты JVM

### `jps` (Список процессов JVM)

Показывает запущенные JVM-процессы и их идентификаторы (PID).

```bash
jps
# Вывод:
# 1234 com.example.Main
# 5678 Jps
```

### `jstat` (Статистика JVM)

Отображает статистику по памяти и сборке мусора.

```bash
jstat -gc 1234
```

### `jstack` (Снимок стека)

Выводит стек вызовов для указанного процесса.

```bash
jstack 1234
```

### `jmap` (Снимок памяти)

Создает дамп кучи или статистику по объектам.

```bash
jmap -dump:file=heap.hprof 1234
```

## 10. Подводные камни и лучшие практики

### Подводные камни

1. **Ошибки в `classpath`**:
    - Неправильный путь или отсутствие классов приводит к `ClassNotFoundException`.
    - **Решение**: Проверьте структуру директорий и пакетов. Используйте `-cp .` для текущей папки.
2. **Кодировка**:
    - Неправильная кодировка исходных файлов может привести к ошибкам компиляции.
    - **Решение**: Указывайте `-encoding UTF-8` в `javac`.
3. **Отладка без `-g`**:
    - Без флага `-g` `jdb` не сможет отображать локальные переменные.
    - **Решение**: Всегда компилируйте с `-g` для отладки.
4. **JAR без манифеста**:
    - Если манифест отсутствует или `Main-Class` указан неверно, `java -jar` не сработает.
    - **Решение**: Проверяйте `META-INF/MANIFEST.MF`.
5. **Переменные окружения**:
    - Неправильная настройка `JAVA_HOME` или `PATH` может привести к ошибкам.
    - **Решение**: Убедитесь, что JDK установлен и пути настроены.

### Лучшие практики

1. **Используйте `-d` для организации классов**:
    - Сохраняйте `.class` файлы в отдельной папке (например, `bin`).
2. **Кэшируйте `classpath`**:
    - Для сложных проектов используйте переменную окружения `CLASSPATH` или скрипты.
3. **Добавляйте отладочную информацию**:
    - Компилируйте с `-g` для поддержки `jdb` или IDE.
4. **Автоматизируйте с инструментами сборки**:
    - Для больших проектов используйте `Maven` или `Gradle`, но знание командной строки полезно для понимания.
5. **Пишите JavaDoc-комментарии**:
    - Используйте `@param`, `@return`, `@author` для генерации качественной документации.
6. **Проверяйте байт-код с `javap`**:
    - Используйте `javap -c` для анализа оптимизаций или ошибок.

## 11. Пример полного рабочего процесса

### Исходный код (`src/com/example/Calculator.java`)

```java
package com.example;

/**
 * Простой калькулятор.
 * @author Ivan Motorin
 */
public class Calculator {
    public static void main(String[] args) {
        Calculator calc = new Calculator();
        System.out.println(calc.add(5, 3));
    }

    /**
     * Складывает два числа.
     * @param a Первое число
     * @param b Второе число
     * @return Сумма
     */
    public int add(int a, int b) {
        return a + b;
    }
}
```

### Шаги

1. **Компиляция**:
    
    ```bash
    javac -g -d bin src/com/example/Calculator.java
    ```
    
2. **Запуск**:
    
    ```bash
    java -classpath bin com.example.Calculator
    # Вывод: 8
    ```
    
3. **Создание JAR**:
    
    ```bash
    echo "Main-Class: com.example.Calculator" > manifest.mf
    jar cvmf manifest.mf calculator.jar -C bin .
    ```
    
4. **Запуск JAR**:
    
    ```bash
    java -jar calculator.jar
    # Вывод: 8
    ```
    
5. **Отладка**:
    
    ```bash
    jdb -classpath bin -sourcepath src com.example.Calculator
    > stop at com.example.Calculator:9
    > run
    ```
    
6. **Дизассемблирование**:
    
    ```bash
    javap -c -classpath calculator.jar com.example.Calculator
    ```
    
7. **Генерация документации**:
    
    ```bash
    javadoc -d doc -charset utf-8 -sourcepath src -author -subpackages com.example
    ```
    

## 12. Заключение

Работа с Java через командную строку — это фундаментальный навык, который помогает понять внутренние процессы компиляции, запуска и отладки приложений. Утилиты `javac`, `java`, `jdb`, `jar`, `javap` и `javadoc` предоставляют полный контроль над жизненным циклом Java-программы. Понимание `classpath`, параметров JVM и особенностей каждой утилиты позволяет эффективно разрабатывать и отлаживать код даже в сложных проектах. Для больших приложений рекомендуется использовать инструменты сборки (`Maven`, `Gradle`), но знание командной строки остается незаменимым для низкоуровневой работы и отладки.