---
title: "Java Input/Output — потоки, NIO.2, Files"
tags: [java, io, nio, streams, files, inputstream, outputstream]
updated: 2026-03-04
---

# Java Input/Output (I/O и NIO.2)

> [!QUOTE] Суть
> **java.io** — блокирующий, поток байтов/символов. **java.nio** — неблокирующий, буферы + каналы (Channel + Selector). **java.nio.file** (NIO.2, Java 7+) — Files, Path, WatchService. Для файловых операций используй `Files.readAllBytes()`, `Files.lines()` вместо старых `FileInputStream`.

> [!WARNING] Ловушка: не закрытые потоки
> Открытые `InputStream`/`OutputStream` без закрытия — утечка файловых дескрипторов. Всегда используй **try-with-resources**: `try (var is = new FileInputStream(file)) { ... }`. Автоматически вызывает `close()` при выходе из блока.

## 1. Основы ввода-вывода в Java

Система I/O в Java построена на концепции **потоков** (streams), которые представляют последовательность данных. Потоки делятся на два типа:

- **Байтовые потоки**: Работают с данными в виде байтов (8 бит). Используются для бинарных данных (например, файлы, изображения, сеть).
- **Символьные потоки**: Работают с данными в виде символов Unicode (16 бит). Используются для текстовых данных.

### Основные пакеты

- **`java.io`**: Содержит основные классы для работы с потоками, файлами и сериализацией.
- **`java.nio`**: Предоставляет неблокирующий I/O, буферы и каналы для высокопроизводительных операций.
- **`java.nio.file`**: Улучшенный API для работы с файлами и директориями (Java 7+).

### Категории потоков

1. **Входные потоки** (input streams): Читают данные из источника (например, `InputStream`, `Reader`).
2. **Выходные потоки** (output streams): Записывают данные в приёмник (например, `OutputStream`, `Writer`).
3. **Фильтрующие потоки**: Оборачивают другие потоки, добавляя функциональность (например, буферизация, преобразование).

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

### 3.1. Чтение и запись файлов

Пример чтения и записи с использованием `FileInputStream` и `FileOutputStream`:

```java
import java.io.*;

public class ByteStreamExample {
    public static void main(String[] args) {
        // Запись в файл
        try (FileOutputStream fos = new FileOutputStream("output.bin")) {
            byte[] data = {65, 66, 67}; // 'A', 'B', 'C'
            fos.write(data);
            System.out.println("Данные записаны");
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Чтение из файла
        try (FileInputStream fis = new FileInputStream("output.bin")) {
            int byteData;
            while ((byteData = fis.read()) != -1) {
                System.out.print((char) byteData);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**Вывод**:

```
Данные записаны
ABC
```

### 3.2. Буферизация

Буферизация снижает количество обращений к источнику/приёмнику, улучшая производительность.

```java
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("input.bin"));
     BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("output.bin"))) {
    int byteData;
    while ((byteData = bis.read()) != -1) {
        bos.write(byteData);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

### 3.3. Работа с примитивами

`DataInputStream` и `DataOutputStream` позволяют читать и записывать примитивные типы.

```java
try (DataOutputStream dos = new DataOutputStream(new FileOutputStream("data.bin"))) {
    dos.writeInt(42);
    dos.writeDouble(3.14);
    dos.writeUTF("Привет");
} catch (IOException e) {
    e.printStackTrace();
}

try (DataInputStream dis = new DataInputStream(new FileInputStream("data.bin"))) {
    System.out.println("Int: " + dis.readInt());
    System.out.println("Double: " + dis.readDouble());
    System.out.println("String: " + dis.readUTF());
} catch (IOException e) {
    e.printStackTrace();
}
```

**Вывод**:

```
Int: 42
Double: 3.14
String: Привет
```

## 4. Работа с символьными потоками

### 4.1. Чтение и запись текстовых файлов

`FileReader` и `FileWriter` предназначены для текстовых данных.

```java
try (FileWriter writer = new FileWriter("output.txt")) {
    writer.write("Привет, мир!\n");
    writer.write("Java I/O");
} catch (IOException e) {
    e.printStackTrace();
}

try (FileReader reader = new FileReader("output.txt")) {
    int charData;
    while ((charData = reader.read()) != -1) {
        System.out.print((char) charData);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

**Вывод**:

```
Привет, мир!
Java I/O
```

### 4.2. Буферизация текстовых потоков

`BufferedReader` и `BufferedWriter` ускоряют операции за счет чтения/записи большими блоками.

```java
try (BufferedReader reader = new BufferedReader(new FileReader("input.txt"));
     BufferedWriter writer = new BufferedWriter(new FileWriter("output.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        writer.write(line.toUpperCase());
        writer.newLine();
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

### 4.3. Преобразование байтов в символы

`InputStreamReader` и `OutputStreamWriter` используются для преобразования между байтовыми и символьными потоками с указанием кодировки.

```java
try (InputStreamReader isr = new InputStreamReader(new FileInputStream("input.txt"), "UTF-8");
     OutputStreamWriter osw = new OutputStreamWriter(new FileOutputStream("output.txt"), "UTF-8")) {
    int charData;
    while ((charData = isr.read()) != -1) {
        osw.write(charData);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

### 4.4. `PrintWriter` для форматированного вывода

`PrintWriter` удобен для записи текста с поддержкой форматирования.

```java
try (PrintWriter writer = new PrintWriter("output.txt", "UTF-8")) {
    writer.printf("Имя: %s, Возраст: %d%n", "Алексей", 30);
    writer.println("Java I/O");
} catch (IOException e) {
    e.printStackTrace();
}
```

## 5. Сериализация и десериализация

Сериализация — это процесс преобразования объекта в поток байтов, а десериализация — восстановление объекта из потока. Используются `ObjectInputStream` и `ObjectOutputStream`.

### Пример

```java
import java.io.*;

public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient int age; // Не сериализуется

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }

    public static void main(String[] args) {
        // Сериализация
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("person.ser"))) {
            Person person = new Person("Мария", 25);
            oos.writeObject(person);
            System.out.println("Сериализовано: " + person);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Десериализация
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("person.ser"))) {
            Person person = (Person) ois.readObject();
            System.out.println("Десериализовано: " + person);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

**Вывод**:

```
Сериализовано: Person{name='Мария', age=25}
Десериализовано: Person{name='Мария', age=0}
```

**Заметка**: Поле `age` (`transient`) становится `0` после десериализации.

## 6. Работа с файлами (`java.nio.file`)

Пакет `java.nio.file` (Java 7+) предоставляет современный API для работы с файлами и директориями.

### Основные классы

- **`Path`**: Представляет путь к файлу или директории.
- **`Files`**: Утилитные методы для операций с файлами.

### Пример

```java
import java.nio.file.*;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

public class NioFileExample {
    public static void main(String[] args) {
        Path path = Paths.get("example.txt");

        // Запись в файл
        try {
            Files.writeString(path, "Привет, NIO!\n", StandardCharsets.UTF_8);
            System.out.println("Файл записан");
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Чтение файла
        try {
            String content = Files.readString(path, StandardCharsets.UTF_8);
            System.out.println("Содержимое: " + content);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Копирование файла
        Path copyPath = Paths.get("example_copy.txt");
        try {
            Files.copy(path, copyPath, StandardCopyOption.REPLACE_EXISTING);
            System.out.println("Файл скопирован");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**Вывод**:

```
Файл записан
Содержимое: Привет, NIO!

Файл скопирован
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

## 7. Работа с консолью

`System.in`, `System.out` и `System.err` — это стандартные потоки для ввода-вывода в консоли.

### Пример

```java
import java.io.*;

public class ConsoleExample {
    public static void main(String[] args) {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(System.in))) {
            System.out.println("Введите текст:");
            String line = reader.readLine();
            System.out.println("Вы ввели: " + line);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**Заметка**: `PrintWriter` или `System.out.printf` удобны для форматированного вывода в консоль.

## 8. NIO.2: Каналы и буферы

Пакет `java.nio` предоставляет каналы (`Channel`) и буферы (`Buffer`) для высокопроизводительного I/O.

### Основные классы

- **`ByteBuffer`**: Буфер для работы с байтами.
- **`FileChannel`**: Канал для чтения/записи файлов.

### Пример

```java
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.*;

public class NioChannelExample {
    public static void main(String[] args) {
        Path path = Paths.get("output.bin");

        // Запись
        try (FileChannel channel = FileChannel.open(path, StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            buffer.put("Привет, NIO!".getBytes(StandardCharsets.UTF_8));
            buffer.flip();
            channel.write(buffer);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Чтение
        try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            channel.read(buffer);
            buffer.flip();
            System.out.println(new String(buffer.array(), 0, buffer.limit(), StandardCharsets.UTF_8));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**Вывод**:

```
Привет, NIO!
```

**Заметка**: `ByteBuffer` требует переключения между режимами записи (`put`) и чтения (`flip`).

## 9. Производительность

- **Буферизация**: Используйте `BufferedInputStream`, `BufferedOutputStream`, `BufferedReader`, `BufferedWriter` для уменьшения системных вызовов.
- **NIO**: `FileChannel` и `ByteBuffer` обеспечивают лучшую производительность для больших файлов.
- **Try-with-Resources**: Автоматическое закрытие ресурсов снижает вероятность утечек.
- **Кодировка**: Указывайте кодировку (например, `UTF-8`) для предсказуемого поведения.

**Пример оптимизации**:

```java
// Плохо: чтение по одному байту
try (FileInputStream fis = new FileInputStream("input.bin")) {
    int byteData;
    while ((byteData = fis.read()) != -1) {
        // Обработка
    }
}

// Хорошо: использование буфера
try (FileInputStream fis = new FileInputStream("input.bin")) {
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = fis.read(buffer)) != -1) {
        // Обработка buffer[0..bytesRead]
    }
}
```

## 10. Безопасность

- **Десериализация**: Десериализация объектов из недоверенных источников может привести к инъекции вредоносного кода.
    - **Решение**: Используйте `ObjectInputFilter` (Java 9+):
        
        ```java
        ObjectInputFilter filter = ObjectInputFilter.Config.createFilter("com.example.*;!*");
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("data.ser"))) {
            ois.setObjectInputFilter(filter);
            Object obj = ois.readObject();
        }
        ```
        
- **Кодировка**: Неправильная кодировка может исказить данные.
    - **Решение**: Всегда указывайте кодировку (например, `StandardCharsets.UTF_8`).
- **Утечки ресурсов**: Незакрытые потоки могут исчерпать файловые дескрипторы.
    - **Решение**: Используйте `try-with-resources`.

## 11. Подводные камни

1. **Необработанные исключения**:
    
    - Все операции I/O могут выбросить `IOException`.
    - **Решение**: Всегда обрабатывайте исключения или объявляйте их через `throws`.
2. **Неправильная кодировка**:
    
    - Чтение текста без указания кодировки может привести к искажению.
    - **Решение**: Используйте `InputStreamReader` с явной кодировкой.
3. **Утечки ресурсов**:
    
    - Незакрытые потоки приводят к проблемам.
    - **Решение**: Используйте `try-with-resources` (Java 7+).
4. **Десериализация недоверенных данных**:
    
    - Может вызвать выполнение вредоносного кода.
    - **Решение**: Ограничивайте классы с помощью `ObjectInputFilter`.
5. **Большие файлы**:
    
    - Чтение больших файлов в память может вызвать `OutOfMemoryError`.
    - **Решение**: Читайте файлы по частям или используйте `FileChannel`.

## 12. Лучшие практики

1. **Используйте `try-with-resources`**:
    
    - Гарантирует закрытие ресурсов.
    
    ```java
    try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
        // Код
    }
    ```
    
2. **Применяйте буферизацию**:
    
    - Используйте `Buffered*` классы для уменьшения системных вызовов.
3. **Указывайте кодировку**:
    
    - Всегда задавайте `StandardCharsets.UTF_8` для текстовых операций.
4. **Предпочитайте `java.nio.file`**:
    
    - Используйте `Files` и `Path` для работы с файлами вместо `File`.
5. **Обрабатывайте исключения**:
    
    - Логируйте или обрабатывайте `IOException` осмысленно.
    
    ```java
    import java.util.logging.Logger;
    
    Logger logger = Logger.getLogger("IO");
    try (FileInputStream fis = new FileInputStream("file.txt")) {
        // Код
    } catch (IOException e) {
        logger.severe("Ошибка чтения файла: " + e.getMessage());
    }
    ```
    
6. **Используйте NIO для больших данных**:
    
    - `FileChannel` и `ByteBuffer` для высокопроизводительных операций.
7. **Ограничивайте десериализацию**:
    
    - Используйте фильтры или альтернативы (JSON, Protocol Buffers).
8. **Тестируйте производительность**:
    
    - Используйте профилировщики (например, VisualVM) для анализа I/O.

## 13. Пример: Комплексная обработка I/O

```java
import java.io.*;
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.util.logging.Logger;

public class FileProcessor {
    private static final Logger LOGGER = Logger.getLogger(FileProcessor.class.getName());

    public static void processFile(Path inputPath, Path outputPath) {
        // Чтение с помощью java.nio.file
        try {
            String content = Files.readString(inputPath, StandardCharsets.UTF_8);
            LOGGER.info("Прочитано: " + inputPath);

            // Преобразование в верхний регистр
            String transformed = content.toUpperCase();

            // Запись с помощью BufferedWriter
            try (BufferedWriter writer = Files.newBufferedWriter(
		            outputPath, StandardCharsets.UTF_8
	            )
            ) {
                writer.write(transformed);
                LOGGER.info("Записано: " + outputPath);
            }
        } catch (IOException e) {
            LOGGER.severe("Ошибка обработки файла: " + e.getMessage());
        }

        // Сериализация объекта
        try (ObjectOutputStream oos = new ObjectOutputStream(
		        Files.newOutputStream(Paths.get("data.ser"))
	        )
        ) {
            oos.writeObject(new Person("Алексей", 30));
            LOGGER.info("Объект сериализован");
        } catch (IOException e) {
            LOGGER.severe("Ошибка сериализации: " + e.getMessage());
        }

        // Десериализация с фильтром
        ObjectInputFilter filter = ObjectInputFilter.Config
	        .createFilter("Person;!*");
        try (ObjectInputStream ois = new ObjectInputStream(
		        Files.newInputStream(Paths.get("data.ser"))
	        )
		) {
            ois.setObjectInputFilter(filter);
            Person person = (Person) ois.readObject();
            LOGGER.info("Десериализовано: " + person);
        } catch (IOException | ClassNotFoundException e) {
            LOGGER.severe("Ошибка десериализации: " + e.getMessage());
        }
    }

    public static void main(String[] args) {
        Path input = Paths.get("input.txt");
        Path output = Paths.get("output.txt");
        processFile(input, output);
    }
}

class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }
}
```

**Предполагаемый вывод** (при наличии `input.txt`):

```
INFO: Прочитано: input.txt
INFO: Записано: output.txt
INFO: Объект сериализован
INFO: Десериализовано: Person{name='Алексей', age=30}
```