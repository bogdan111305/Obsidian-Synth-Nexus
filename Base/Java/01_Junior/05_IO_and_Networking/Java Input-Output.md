---
title: "Java Input/Output — потоки, NIO.2, Files"
tags: [java, io, nio, streams, files, inputstream, outputstream]
updated: 2026-08-16
---

# Java Input/Output (I/O и NIO.2)

> [!QUOTE] Суть
> **java.io** — блокирующий, поток байтов/символов. **java.nio** — неблокирующий, буферы + каналы (Channel + Selector). **java.nio.file** (NIO.2, Java 7+) — Files, Path, WatchService. Для файловых операций используй `Files.readAllBytes()`, `Files.lines()` вместо старых `FileInputStream`.

> [!WARNING] Ловушка: не закрытые потоки
> Открытые `InputStream`/`OutputStream` без закрытия — утечка файловых дескрипторов. Всегда используй **try-with-resources**: `try (var is = new FileInputStream(file)) { ... }`. Автоматически вызывает `close()` при выходе из блока.

## 1. Основы ввода-вывода в Java

Система I/O в Java построена на **потоках** (streams) — последовательностях данных.

- **Байтовые потоки** — данные как байты (8 бит); для бинарных данных (файлы, изображения, сеть).
- **Символьные потоки** — данные как символы Unicode (16 бит); для текстовых данных.
- **Фильтрующие потоки** — оборачивают другой поток, добавляя функциональность (буферизация, преобразование типов), не меняя интерфейс. Это классическая реализация паттерна Decorator: каждый обёрточный класс (`Buffered*`, `Data*`, мосты `InputStreamReader`/`OutputStreamWriter`) хранит ссылку на оборачиваемый поток и добавляет свою логику поверх его вызовов. Благодаря этому потоки свободно комбинируются в цепочки произвольной длины: `new BufferedReader(new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8))` — файл → байты → декодирование в символы → буферизация построчного чтения.

### Сравнение пакетов

| | `java.io` | `java.nio` | `java.nio.file` (NIO.2) |
|--|--|--|--|
| Модель | Блокирующая | Неблокирующая | — (утилиты) |
| Единица данных | Байт/символ (Stream) | Буфер (Channel + Buffer) | Path/файл целиком |
| Подходит для | Простые файловые операции | High-throughput, серверы | Современная работа с файлами (Java 7+) |

## 2. Иерархия классов I/O

### 2.1. Байтовые потоки

Базовые классы: `InputStream` и `OutputStream` (абстрактные).

|Класс|Описание|Пример использования|
|---|---|---|
|`FileInputStream`|Читает байты из файла.|Чтение бинарных файлов.|
|`FileOutputStream`|Записывает байты в файл.|Запись в бинарные файлы.|
|`BufferedInputStream`|Буферизирует входные данные.|Уменьшение обращений к источнику.|
|`BufferedOutputStream`|Буферизирует выходные данные.|Уменьшение обращений к приёмнику.|
|`DataInputStream`|Читает примитивные типы данных.|Чтение чисел из потока.|
|`DataOutputStream`|Записывает примитивные типы данных.|Запись чисел в поток.|
|`ObjectInputStream`|Десериализует объекты.|Восстановление объектов из потока.|
|`ObjectOutputStream`|Сериализует объекты.|Сохранение объектов в поток.|

### 2.2. Символьные потоки

Базовые классы: `Reader` и `Writer` (абстрактные).

|Класс|Описание|Пример использования|
|---|---|---|
|`FileReader`|Читает символы из файла.|Чтение текстовых файлов.|
|`FileWriter`|Записывает символы в файл.|Запись в текстовые файлы.|
|`BufferedReader`|Буферизирует чтение символов.|Эффективное чтение текста.|
|`BufferedWriter`|Буферизирует запись символов.|Эффективная запись текста.|
|`InputStreamReader`|Преобразует байты в символы.|Чтение текста из байтового потока.|
|`OutputStreamWriter`|Преобразует символы в байты.|Запись текста в байтовый поток.|
|`PrintWriter`|Удобная запись форматированного текста.|Логирование, вывод в консоль.|

### 2.3. Классы `java.nio.file`

|Класс/Интерфейс|Описание|Пример использования|
|---|---|---|
|`Path`|Представляет путь к файлу или директории.|Работа с путями.|
|`Files`|Утилитные методы для работы с файлами.|Чтение, запись, копирование файлов.|
|`FileSystem`|Представляет файловую систему.|Доступ к файловым системам.|

## 3. Работа с байтовыми потоками

**Базовое чтение/запись** — `FileInputStream`/`FileOutputStream` читают и пишут по одному байту или в массив:

```java
try (FileOutputStream fos = new FileOutputStream("output.bin")) {
    fos.write(new byte[] {65, 66, 67}); // 'A', 'B', 'C'
}

try (FileInputStream fis = new FileInputStream("output.bin")) {
    int byteData;
    while ((byteData = fis.read()) != -1) {
        System.out.print((char) byteData); // ABC
    }
}
```

**Буферизация** — оборачивание в `BufferedInputStream`/`BufferedOutputStream` снижает число обращений к источнику/приёмнику (посимвольное чтение без буфера — дорого из-за системных вызовов на каждый байт):

```java
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("input.bin"));
     BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("output.bin"))) {
    int byteData;
    while ((byteData = bis.read()) != -1) bos.write(byteData);
}
```

**Примитивы** — `DataInputStream`/`DataOutputStream` читают/пишут `int`, `double`, UTF-строки напрямую, без ручной сериализации байтов:

```java
try (DataOutputStream dos = new DataOutputStream(new FileOutputStream("data.bin"))) {
    dos.writeInt(42);
    dos.writeDouble(3.14);
    dos.writeUTF("Привет");
}
try (DataInputStream dis = new DataInputStream(new FileInputStream("data.bin"))) {
    System.out.println(dis.readInt());    // 42
    System.out.println(dis.readDouble()); // 3.14
    System.out.println(dis.readUTF());    // Привет
}
```

## 4. Работа с символьными потоками

- **`FileReader`/`FileWriter`** — базовое чтение/запись текстовых файлов (посимвольно).
- **`BufferedReader`/`BufferedWriter`** — буферизация; `readLine()` даёт удобное построчное чтение.
- **`InputStreamReader`/`OutputStreamWriter`** — мост байты↔символы с явным указанием кодировки; без явной кодировки поведение платформозависимо и непредсказуемо.
- **`PrintWriter`** — форматированный вывод (`printf`, `println`) поверх любого потока.

```java
// Запись + построчное чтение с буферизацией и явной кодировкой:
try (BufferedWriter writer = new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream("output.txt"), StandardCharsets.UTF_8))) {
    writer.write("Привет, мир!");
    writer.newLine();
    writer.write("Java I/O");
}

try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream("output.txt"), StandardCharsets.UTF_8))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line.toUpperCase());
    }
}

// PrintWriter — форматированный вывод:
try (PrintWriter writer = new PrintWriter("log.txt", "UTF-8")) {
    writer.printf("Имя: %s, Возраст: %d%n", "Алексей", 30);
}
```

## 5. Сериализация и десериализация

Сериализация — преобразование объекта в поток байтов; десериализация — восстановление объекта из потока. Классы: `ObjectOutputStream`/`ObjectInputStream`, объект должен реализовывать `Serializable`. Поле `transient` в сериализацию не попадает (при десериализации получает значение по умолчанию).

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient int age; // не сериализуется

    public Person(String name, int age) { this.name = name; this.age = age; }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("person.ser"))) {
            oos.writeObject(new Person("Мария", 25));
        }
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("person.ser"))) {
            Person person = (Person) ois.readObject();
            System.out.println(person); // age восстановится как 0 (transient)
        }
    }
}
```

Десериализация недоверенных данных может выполнить произвольный код через `readObject()` — ограничивай допустимые классы через `ObjectInputFilter` (Java 9+) или, где возможно, избегай Java-сериализации в пользу JSON/Protocol Buffers:

```java
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter("com.example.*;!*");
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("data.ser"))) {
    ois.setObjectInputFilter(filter);
    Object obj = ois.readObject();
}
```

## 6. Работа с файлами (`java.nio.file`, Java 7+)

Современный API для файлов и директорий: `Path` — путь, `Files` — утилитные операции над ним.

```java
Path path = Path.of("example.txt");

Files.writeString(path, "Привет, NIO!\n", StandardCharsets.UTF_8);
String content = Files.readString(path, StandardCharsets.UTF_8);

Files.copy(path, Path.of("example_copy.txt"), StandardCopyOption.REPLACE_EXISTING);
```

### Полезные методы `Files`

|Метод|Описание|Пример|
|---|---|---|
|`readString(Path, Charset)`|Читает весь файл в строку.|`Files.readString(path, StandardCharsets.UTF_8)`|
|`writeString(Path, CharSequence, Charset)`|Записывает строку в файл.|`Files.writeString(path, "Текст")`|
|`readAllBytes(Path)`|Читает файл в массив байтов.|`Files.readAllBytes(path)`|
|`write(Path, byte[])`|Записывает байты в файл.|`Files.write(path, bytes)`|
|`copy(Path, Path, CopyOption...)`|Копирует файл.|`Files.copy(src, dst)`|
|`delete(Path)`|Удаляет файл.|`Files.delete(path)`|
|`walk(Path)`|Рекурсивно обходит директорию.|`Files.walk(path).forEach(System.out::println)`|

`readString`/`readAllBytes`/`writeString` загружают весь файл в память целиком — для больших файлов используй потоковое чтение (`Files.lines()`, `Files.newBufferedReader()`) или `FileChannel` (§8), а не эти методы.

## 7. Работа с консолью

`System.in`, `System.out`, `System.err` — стандартные потоки консоли.

```java
try (BufferedReader reader = new BufferedReader(new InputStreamReader(System.in))) {
    System.out.println("Введите текст:");
    String line = reader.readLine();
    System.out.println("Вы ввели: " + line);
}
```

## 8. NIO: каналы и буферы

`java.nio` — `Channel` + `Buffer` для высокопроизводительного I/O. `ByteBuffer` требует явного переключения между режимами записи (`put`) и чтения (`flip`); `FileChannel` — канал для чтения/записи файлов, эффективен на больших файлах.

```java
Path path = Path.of("output.bin");

try (FileChannel channel = FileChannel.open(path, StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    buffer.put("Привет, NIO!".getBytes(StandardCharsets.UTF_8));
    buffer.flip();       // переключение put → get
    channel.write(buffer);
}

try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    channel.read(buffer);
    buffer.flip();
    System.out.println(new String(buffer.array(), 0, buffer.limit(), StandardCharsets.UTF_8));
}
```

## 9. Производительность, безопасность и типичные ошибки

- **Буферизация обязательна** для посимвольного/побайтового чтения — оборачивай в `Buffered*` (или используй `Files`/`FileChannel`, которые буферизуют сами). Разница на больших данных — на порядки:

```java
// Плохо: системный вызов на каждый байт
try (FileInputStream fis = new FileInputStream("input.bin")) {
    int b;
    while ((b = fis.read()) != -1) { /* ... */ }
}
// Хорошо: чтение блоками
try (FileInputStream fis = new FileInputStream("input.bin")) {
    byte[] buffer = new byte[8192];
    int n;
    while ((n = fis.read(buffer)) != -1) { /* обработка buffer[0..n) */ }
}
```

- **try-with-resources всегда** — гарантирует `close()` даже при исключении; ручной `finally` — источник утечек дескрипторов.
- **Кодировка — всегда явно** (`StandardCharsets.UTF_8`) при преобразовании байты↔символы; неявная платформенная кодировка ломается между окружениями.
- **Большие файлы не читай целиком в память** (`readAllBytes`/`readString`) — риск `OutOfMemoryError`; используй потоковое чтение или `FileChannel`.
- **Десериализация недоверенных данных опасна** — ограничивай классы через `ObjectInputFilter` или замени Java-сериализацию на JSON/Protobuf (см. §5).
- **`IOException` — checked и вездесущ** — обрабатывай осмысленно (лог с контекстом), не глотай молча.
- **Предпочитай `java.nio.file`** (`Files`/`Path`) старому `File` для новых файловых операций — короче, безопаснее, лучше диагностика ошибок.

## 10. Пример: комплексная обработка I/O

```java
import java.io.*;
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.util.logging.Logger;

public class FileProcessor {
    private static final Logger LOGGER = Logger.getLogger(FileProcessor.class.getName());

    public static void processFile(Path inputPath, Path outputPath) {
        try {
            String content = Files.readString(inputPath, StandardCharsets.UTF_8);
            String transformed = content.toUpperCase();
            try (BufferedWriter writer = Files.newBufferedWriter(outputPath, StandardCharsets.UTF_8)) {
                writer.write(transformed);
            }
            LOGGER.info("Обработано: " + inputPath + " -> " + outputPath);
        } catch (IOException e) {
            LOGGER.severe("Ошибка обработки файла: " + e.getMessage());
        }

        try (ObjectOutputStream oos = new ObjectOutputStream(Files.newOutputStream(Path.of("data.ser")))) {
            oos.writeObject(new Person("Алексей", 30));
        } catch (IOException e) {
            LOGGER.severe("Ошибка сериализации: " + e.getMessage());
        }

        ObjectInputFilter filter = ObjectInputFilter.Config.createFilter("Person;!*");
        try (ObjectInputStream ois = new ObjectInputStream(Files.newInputStream(Path.of("data.ser")))) {
            ois.setObjectInputFilter(filter);
            LOGGER.info("Десериализовано: " + ois.readObject());
        } catch (IOException | ClassNotFoundException e) {
            LOGGER.severe("Ошибка десериализации: " + e.getMessage());
        }
    }
}

class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private int age;
    public Person(String name, int age) { this.name = name; this.age = age; }
    @Override public String toString() { return "Person{name='" + name + "', age=" + age + "}"; }
}
```
