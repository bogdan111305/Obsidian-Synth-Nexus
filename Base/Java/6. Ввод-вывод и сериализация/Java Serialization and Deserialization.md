---
title: "Java — Сериализация и Десериализация"
tags: [java, serialization, deserialization, serializable, externalizable, transient]
updated: 2026-03-04
---

# Java Serialization and Deserialization

## 1. Что такое сериализация и десериализация?

- **Сериализация**: Процесс преобразования объекта Java в последовательность байтов, которую можно сохранить в файл, передать по сети или сохранить в базе данных.
- **Десериализация**: Процесс восстановления объекта Java из потока байтов.

### Основные применения

- Сохранение состояния объектов (например, в файл или базу данных).
- Передача объектов между клиентом и сервером (например, в RMI или сетевых приложениях).
- Кэширование объектов.
- Поддержка распределенных систем.

### Основные классы и интерфейсы

- **`java.io.Serializable`**: Маркерный интерфейс для включения сериализации.
- **`java.io.Externalizable`**: Интерфейс для пользовательской сериализации.
- **`ObjectOutputStream`**: Класс для записи объектов в поток (сериализация).
- **`ObjectInputStream`**: Класс для чтения объектов из потока (десериализация).

## 2. Интерфейс `Serializable`

`Serializable` — это маркерный интерфейс без методов, который указывает JVM, что объект можно сериализовать. JVM автоматически обрабатывает сериализацию и десериализацию для классов, реализующих `Serializable`.

### Основные особенности

- Все поля объекта, не помеченные как `transient`, сериализуются.
- Если поле имеет тип, не реализующий `Serializable`, будет выброшено `NotSerializableException`.
- Наследуемые поля сериализуются, если родительский класс также реализует `Serializable`.

### Пример простой сериализации

```java
import java.io.*;

public class Person implements Serializable {
    private String name;
    private int age;
    private transient String password; // Не сериализуется

    public Person(String name, int age, String password) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + ", password='" + password + "'}";
    }

    public static void main(String[] args) {
        // Сериализация
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("person.ser"))) {
            Person person = new Person("Алексей", 30, "secret");
            oos.writeObject(person);
            System.out.println("Объект сериализован: " + person);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Десериализация
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("person.ser"))) {
            Person person = (Person) ois.readObject();
            System.out.println("Объект десериализован: " + person);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

**Вывод**:

```
Объект сериализован: Person{name='Алексей', age=30, password='secret'}
Объект десериализован: Person{name='Алексей', age=30, password='null'}
```

**Заметка**: Поле `password`, помеченное как `transient`, не сериализуется и становится `null` после десериализации.

### Поле `serialVersionUID`

`serialVersionUID` — это уникальный идентификатор версии класса, используемый для проверки совместимости при десериализации. Если класс изменился (например, добавлены или удалены поля), JVM может выбросить `InvalidClassException`.

**Рекомендация**: Всегда явно задавайте `serialVersionUID`:

```java
private static final long serialVersionUID = 1L;
```

Генерация `serialVersionUID`:

- Используйте IDE (например, IntelliJ IDEA, Eclipse).
- Или утилиту `serialver`:
    
    ```bash
    serialver com.example.Person
    ```
    

**Пример**:

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    // ...
}
```

## 3. Интерфейс `Externalizable`

`Externalizable` расширяет `Serializable` и позволяет полностью контролировать процесс сериализации и десериализации, реализуя методы `writeExternal` и `readExternal`.

### Особенности

- Требует реализации двух методов:
    - `writeExternal(ObjectOutput out)`: Определяет, как сериализовать объект.
    - `readExternal(ObjectInput in)`: Определяет, как десериализовать объект.
- Нет автоматической сериализации — разработчик отвечает за все поля.
- Более производителен, но требует больше кода.

### Пример `Externalizable`

```java
import java.io.*;

public class Employee implements Externalizable {
    private String name;
    private int id;

    public Employee() {} // Требуется конструктор без параметров

    public Employee(String name, int id) {
        this.name = name;
        this.id = id;
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(name);
        out.writeInt(id);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        name = in.readUTF();
        id = in.readInt();
    }

    @Override
    public String toString() {
        return "Employee{name='" + name + "', id=" + id + "}";
    }

    public static void main(String[] args) {
        // Сериализация
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("employee.ser"))) {
            Employee emp = new Employee("Мария", 1001);
            oos.writeObject(emp);
            System.out.println("Сериализовано: " + emp);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // Десериализация
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("employee.ser"))) {
            Employee emp = (Employee) ois.readObject();
            System.out.println("Десериализовано: " + emp);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

**Вывод**:

```
Сериализовано: Employee{name='Мария', id=1001}
Десериализовано: Employee{name='Мария', id=1001}
```

**Заметка**: `Externalizable` требует конструктор без параметров, так как JVM вызывает его перед `readExternal`.

## 4. Настройка сериализации

### 4.1. Поля `transient`

Поля, помеченные как `transient`, исключаются из сериализации. Они получают значение по умолчанию (`null` для объектов, `0` для чисел) при десериализации.

### 4.2. Методы `writeObject` и `readObject`

Для пользовательской сериализации в классе, реализующем `Serializable`, можно определить методы:

- `private void writeObject(ObjectOutputStream out) throws IOException`
- `private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException`

**Пример**:

```java
import java.io.*;

public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password;

    public User(String name, String password) {
        this.name = name;
        this.password = password;
    }

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject(); // Сериализует не-transient поля
        out.writeUTF(encrypt(password)); // Пользовательская сериализация
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject(); // Десериализует не-transient поля
        this.password = decrypt(in.readUTF()); // Пользовательская десериализация
    }

    private String encrypt(String data) {
        return new StringBuilder(data).reverse().toString(); // Пример шифрования
    }

    private String decrypt(String data) {
        return new StringBuilder(data).reverse().toString(); // Пример дешифрования
    }

    @Override
    public String toString() {
        return "User{name='" + name + "', password='" + password + "'}";
    }
}
```

**Заметка**: Вызов `defaultWriteObject` и `defaultReadObject` обязателен для сериализации/десериализации стандартных полей.

### 4.3. `writeReplace` и `readResolve`

- **`writeReplace()`**: Позволяет заменить объект перед сериализацией.
- **`readResolve()`**: Позволяет заменить объект после десериализации (например, для реализации синглтона).

**Пример (синглтон)**:

```java
import java.io.*;

public class Singleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }

    private Object readResolve() {
        return INSTANCE; // Гарантирует синглтон при десериализации
    }
}
```

## 5. Внутренняя реализация

### 5.1. Сериализация

- JVM использует рефлексию для анализа структуры класса.
- Сериализуются все не-`transient` поля, включая приватные, если класс реализует `Serializable`.
- Для `Externalizable` JVM вызывает `writeExternal` и `readExternal`.
- Метаданные (имя класса, `serialVersionUID`) включаются в поток.

### 5.2. Десериализация

- JVM читает метаданные и создает объект, используя конструктор без параметров (для `Externalizable`) или рефлексию.
- Проверяется `serialVersionUID` для совместимости.
- Поля восстанавливаются в порядке, определенном классом.

### 5.3. Формат потока

- Поток начинается с магического числа (`AC ED`) и версии (`00 05`).
- Содержит описание класса, поля и их значения.
- Для массивов и коллекций сериализуются элементы рекурсивно.

## 6. Производительность

- **Сериализация**: Затратна из-за рефлексии, записи метаданных и обработки сложных графов объектов.
- **Десериализация**: Может быть медленнее из-за проверки совместимости и создания объектов.
- **`Externalizable`**: Быстрее `Serializable`, так как минимизирует рефлексию.
- **Оптимизация**:
    - Используйте `transient` для ненужных полей.
    - Применяйте `Externalizable` для критических по производительности сценариев.
    - Минимизируйте размер графа объектов.

**Пример оптимизации**:

```java
public class Data implements Serializable {
    private static final long serialVersionUID = 1L;
    private transient List<String> cache; // Не сериализуем кэш
    private String id;

    public Data(String id) {
        this.id = id;
        this.cache = new ArrayList<>();
    }
}
```

## 7. Безопасность

Сериализация и десериализация могут быть уязвимы, особенно при десериализации данных из недоверенных источников.

### Уязвимости

- **Инъекция объектов**: Злоумышленник может передать данные, создающие вредоносные объекты.
- **DoS-атаки**: Сложные графы объектов могут перегрузить JVM.
- **Утечка данных**: Приватные поля могут быть прочитаны через десериализацию.

### Меры безопасности

1. **Используйте `ObjectInputFilter`** (Java 9+):
    
    ```java
    ObjectInputFilter filter = ObjectInputFilter.Config.createFilter("com.example.*;!*");
    ObjectInputStream ois = new ObjectInputStream(new FileInputStream("data.ser"));
    ois.setObjectInputFilter(filter);
    ```
    
2. **Ограничьте десериализуемые классы**:
    
    - Реализуйте `readObject` с проверкой типов.
    - Используйте белый список классов.
3. **Избегайте десериализации из недоверенных источников**:
    
    - Предпочитайте JSON, XML или другие форматы (например, Jackson, Gson).
4. **Используйте `transient` для чувствительных данных**:
    
    - Например, пароли или ключи.

## 8. Подводные камни

1. **Несовместимость версий**:
    
    - Изменение структуры класса (добавление/удаление полей) может вызвать `InvalidClassException`.
    - **Решение**: Задавайте `serialVersionUID` и тестируйте совместимость.
2. **Утечки памяти**:
    
    - Сериализация сложных графов объектов может удерживать ссылки, препятствуя сборке мусора.
    - **Решение**: Используйте `transient` для временных данных.
3. **Безопасность десериализации**:
    
    - Десериализация недоверенных данных может привести к выполнению вредоносного кода.
    - **Решение**: Используйте фильтры или альтернативные форматы.
4. **Производительность**:
    
    - Сериализация больших объектов или коллекций может быть медленной.
    - **Решение**: Оптимизируйте с помощью `Externalizable` или минимизируйте данные.
5. **Не все объекты сериализуемы**:
    
    - Объекты, содержащие несериализуемые поля (например, `Thread`), вызовут `NotSerializableException`.
    - **Решение**: Помечайте такие поля как `transient`.

## 9. Лучшие практики

1. **Явно задавайте `serialVersionUID`**:
    - Предотвращает ошибки совместимости.
2. **Используйте `transient` для ненужных данных**:
    - Снижает размер сериализованного потока и улучшает производительность.
3. **Предпочитайте `Externalizable` для производительности**:
    - Когда требуется полный контроль и оптимизация.
4. **Ограничивайте десериализацию**:
    - Используйте `ObjectInputFilter` или белый список классов.
5. **Рассмотрите альтернативы сериализации**:
    - JSON (Jackson, Gson), XML или Protocol Buffers для передачи данных.
6. **Тестируйте сериализацию**:
    - Проверяйте совместимость версий и корректность восстановления объектов.
7. **Логируйте ошибки**:
    
    ```java
    try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("data.ser"))) {
        Object obj = ois.readObject();
    } catch (IOException | ClassNotFoundException e) {
        Logger.getLogger("Serialization").log(Level.SEVERE, "Ошибка десериализации", e);
    }
    ```
    
8. **Избегайте сериализации сложных графов**:
    - Минимизируйте ссылки между объектами.

## 10. Альтернативы сериализации

- **JSON**: Используйте библиотеки Jackson или Gson для легковесной сериализации.
    
    ```java
    ObjectMapper mapper = new ObjectMapper();
    String json = mapper.writeValueAsString(new Person("Алексей", 30, "secret"));
    Person person = mapper.readValue(json, Person.class);
    ```
    
- **XML**: Используйте JAXB или XStream.
- **Protocol Buffers/Kryo**: Высокопроизводительные форматы для больших данных.
- **Кэширование**: Используйте базы данных (Redis, Ehcache) вместо сериализации для хранения состояния.

## 11. Пример: Комплексная сериализация

```java
import java.io.*;
import java.util.ArrayList;
import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

public class Department implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private List<Person> employees;
    private transient Logger logger = Logger.getLogger(Department.class.getName());

    public Department(String name) {
        this.name = name;
        this.employees = new ArrayList<>();
    }

    public void addEmployee(Person person) {
        employees.add(person);
    }

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        logger.info("Сериализация отдела: " + name);
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        logger = Logger.getLogger(Department.class.getName()); // Восстанавливаем transient поле
        logger.info("Десериализация отдела: " + name);
    }

    @Override
    public String toString() {
        return "Department{name='" + name + "', employees=" + employees + "}";
    }

    public static void main(String[] args) {
        Department dept = new Department("IT");
        dept.addEmployee(new Person("Алексей", 30, "secret"));
        dept.addEmployee(new Person("Мария", 25, "pass123"));

        // Сериализация
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("dept.ser"))) {
            oos.writeObject(dept);
            System.out.println("Сериализовано: " + dept);
        } catch (IOException e) {
            Logger.getLogger("Main").log(Level.SEVERE, "Ошибка сериализации", e);
        }

        // Десериализация
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("dept.ser"))) {
            Department restored = (Department) ois.readObject();
            System.out.println("Десериализовано: " + restored);
        } catch (IOException | ClassNotFoundException e) {
            Logger.getLogger("Main").log(Level.SEVERE, "Ошибка десериализации", e);
        }
    }
}
```

**Вывод**:

```
Сериализовано: Department{name='IT', employees=[Person{name='Алексей', age=30, password='null'}, Person{name='Мария', age=25, password='null'}]}
Десериализовано: Department{name='IT', employees=[Person{name='Алексей', age=30, password='null'}, Person{name='Мария', age=25, password='null'}]}
```