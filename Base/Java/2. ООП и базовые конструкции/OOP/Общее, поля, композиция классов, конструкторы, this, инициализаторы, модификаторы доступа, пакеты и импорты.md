## Основы классов и объектов

### Концепция классов и объектов

- **Класс** — это шаблон или описание, определяющее поля (состояние) и методы (поведение) для объектов. Он задаёт структуру и функциональность.
- **Объект** — это экземпляр класса, созданный с помощью оператора `new`, обладающий собственными значениями полей и способностью вызывать методы.

**Пример**:

```java
public class Person {
    String name; // Поле (состояние)
    int age;     // Поле (состояние)

    void sayHello() { // Метод (поведение)
        System.out.println("Привет, меня зовут " + name);
    }
}

// Создание объекта
Person p = new Person();
p.name = "Алиса";
p.age = 25;
p.sayHello(); // Выведет: Привет, меня зовут Алиса
```

**Ключевые особенности**:

- Класс — это чертеж, объект — его реализация.
- Каждый объект имеет собственные копии полей экземпляра.
- Методы определяют поведение, доступное для всех объектов класса.

---

## Поля и переменные

### Поля экземпляра

Поля — это переменные, объявленные внутри класса, но вне методов. Они описывают **состояние** объекта.

**Пример**:

```java
public class Person {
    String name; // Поле экземпляра
    int age;     // Поле экземпляра
}
```

#### Особенности полей экземпляра

- **Каждому объекту — своя копия**: Каждый объект класса имеет собственные значения полей.
- **Инициализация**:
    - Поля можно инициализировать при объявлении, в конструкторе или инициализаторе.
    - Без явной инициализации примитивы получают значения по умолчанию:
        - Числовые (`int`, `double`, и т.д.): `0`, `0.0`.
        - `boolean`: `false`.
        - Ссылочные типы (объекты, массивы): `null`.
- **Модификаторы доступа**: Управляют видимостью полей (`private`, `protected`, `public`, пакетный доступ).

**Пример с инициализацией**:

```java
public class Person {
    String name = "Неизвестно"; // Инициализация при объявлении
    int age;
}
```

### Статические поля

- **Статические поля** принадлежат классу, а не объектам. Они общие для всех экземпляров.
- Объявляются с ключевым словом `static`.

**Пример**:

```java
public class Person {
    static int populationCount; // Статическое поле
    String name;

    public Person(String name) {
        this.name = name;
        populationCount++; // Увеличиваем счетчик
    }
}
```

**Использование**:

```java
Person p1 = new Person("Алиса");
Person p2 = new Person("Боб");
System.out.println(Person.populationCount); // Выведет: 2
```

**Особенности статических полей**:

- Хранятся в области памяти класса (Method Area).
- Инициализируются при загрузке класса.
- Используются для хранения общего состояния (например, счетчиков, констант).

---

## Конструкторы и инициализация

### Конструкторы

**Конструктор** — это специальный метод, вызываемый при создании объекта с помощью `new`. Он инициализирует поля объекта.

**Пример**:

```java
public class Person {
    String name;
    int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

**Использование**:

```java
Person p = new Person("Алиса", 25);
```

#### Особенности конструкторов

- **Название**: Совпадает с именем класса.
- **Отсутствие возвращаемого типа**: Даже `void` не указывается.
- **Конструктор по умолчанию**: Если конструктор не определён, компилятор создаёт пустой конструктор:
    
    ```java
    public Person() {}
    ```
    
- **Перегрузка**: Можно определить несколько конструкторов с разными параметрами.

**Пример перегрузки**:

```java
public class Person {
    String name;
    int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public Person(String name) {
        this(name, 0); // Вызов другого конструктора
    }

    public Person() {
        this("Неизвестно", 0); // Вызов другого конструктора
    }
}
```

**Использование**:

```java
Person p1 = new Person("Алиса", 25);
Person p2 = new Person("Боб");
Person p3 = new Person();
System.out.println(p2.age); // 0
System.out.println(p3.name); // Неизвестно
```

### Ключевое слово `this`

- **Разрешение конфликтов имён**: Отличает поля от параметров:
    
    ```java
    public Person(String name) {
        this.name = name; // this.name — поле, name — параметр
    }
    ```
    
- **Вызов другого конструктора**: Используется в начале конструктора:
    
    ```java
    public Person() {
        this("Неизвестно", 0);
    }
    ```
    

**Пример с перегрузкой**:

```java
public class Book {
    String title;
    int pages;

    public Book(String title, int pages) {
        this.title = title;
        this.pages = pages;
    }

    public Book(String title) {
        this(title, 100); // По умолчанию 100 страниц
    }

    public Book() {
        this("Без названия"); // Вызов конструктора с одним параметром
    }
}
```

### Инициализаторы

Инициализаторы — это блоки кода, выполняемые при создании объекта или загрузке класса.

#### Нестатический инициализатор (Instance Initializer)

- Выполняется **при создании объекта**, до конструктора.
- Полезен для общей инициализации, используемой в нескольких конструкторах.

**Пример**:

```java
public class Demo {
    int x;

    { // Нестатический инициализатор
        x = 42;
        System.out.println("Инициализатор: x = " + x);
    }

    public Demo() {
        System.out.println("Конструктор: x = " + x);
    }
}
```

**Использование**:

```java
Demo demo = new Demo();
// Выведет:
// Инициализатор: x = 42
// Конструктор: x = 42
```

#### Статический инициализатор (Static Initializer)

- Выполняется **один раз при загрузке класса**.
- Используется для инициализации `static` полей или загрузки ресурсов.

**Пример**:

```java
public class StaticDemo {
    static int count;

    static {
        count = 10;
        System.out.println("Статический блок: count = " + count);
    }
}
```

**Использование**:

```java
System.out.println(StaticDemo.count); // Статический блок выполнится перед доступом
// Выведет:
// Статический блок: count = 10
// 10
```

**Подводные камни**:

- Статические инициализаторы не должны выбрасывать непроверяемые исключения, так как это может привести к `ExceptionInInitializerError`.
- Нестатические инициализаторы увеличивают сложность кода, используйте их осторожно.

---

## Композиция классов

### Основы композиции

**Композиция** — это отношение "имеет" (has-a), при котором один класс содержит объект другого класса как поле. Это мощный инструмент для повторного использования кода и модульной архитектуры.

**Пример**:

```java
public class Engine {
    int horsepower;

    public Engine(int horsepower) {
        this.horsepower = horsepower;
    }
}

public class Car {
    Engine engine; // Композиция: Car "имеет" Engine
    String model;

    public Car(String model, Engine engine) {
        this.model = model;
        this.engine = engine;
    }

    public void start() {
        System.out.println(model + " с двигателем " + engine.horsepower + " л.с. завелся");
    }
}
```

**Использование**:

```java
Engine engine = new Engine(200);
Car car = new Car("Toyota", engine);
car.start(); // Выведет: Toyota с двигателем 200 л.с. завелся
```

#### Особенности композиции

- **Модульность**: Позволяет разделять ответственность между классами.
- **Гибкость**: Легче изменять компоненты, чем наследуемые классы.
- **Предпочтение перед наследованием**: Используйте композицию, если нет логического "is-a" отношения (например, `Car` не является `Engine`).
- **Повторное использование**: Компоненты могут использоваться в разных классах.

**Пример с композицией**:

```java
public class Address {
    String city;
    String street;

    public Address(String city, String street) {
        this.city = city;
        this.street = street;
    }
}

public class User {
    String name;
    Address address; // Композиция

    public User(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    public void printInfo() {
        System.out.println(name + " живет в " + address.city + ", " + address.street);
    }
}
```

**Использование**:

```java
Address address = new Address("Москва", "Ленинский проспект");
User user = new User("Алиса", address);
user.printInfo(); // Выведет: Алиса живет в Москва, Ленинский проспект
```

### Композиция vs Наследование

- **Композиция**:
    - Отношение "имеет" (has-a).
    - Гибкость в изменении поведения.
    - Пример: `Car` имеет `Engine`.
- **Наследование**:
    - Отношение "является" (is-a).
    - Жесткая связь между классами.
    - Пример: `Dog` является `Animal`.

**Рекомендация**: Используйте композицию, если наследование создаёт избыточную зависимость или нарушает принцип единственной ответственности.

---

## Модификаторы доступа

### Таблица модификаторов доступа

Модификаторы доступа управляют видимостью классов, полей, методов и конструкторов.

|Модификатор|В классе|В пакете|В подклассе|Вне пакета|
|---|---|---|---|---|
|`private`|✅|❌|❌|❌|
|(default)|✅|✅|❌|❌|
|`protected`|✅|✅|✅|❌|
|`public`|✅|✅|✅|✅|

**Пример**:

```java
public class Example {
    private int a;        // Только внутри класса
    int b;                // Доступен в пакете (package-private)
    protected int c;      // Пакет + подклассы
    public int d;         // Доступен везде
}
```

**Рекомендация**:

- Используйте `private` для полей, чтобы обеспечить инкапсуляцию.
- Предоставляйте доступ через геттеры/сеттеры:
    
    ```java
    public class Person {
        private String name;
    
        public String getName() {
            return name;
        }
    
        public void setName(String name) {
            this.name = name;
        }
    }
    ```
    

---

## Пакеты и импорты

### Основы пакетов

**Пакет** — это пространство имён для организации классов, предотвращающее конфликты имён и упрощающее структуру проекта.

**Пример**:

```java
package com.example.utils;

public class StringHelper {
    public static String capitalize(String input) {
        return input.substring(0, 1).toUpperCase() + input.substring(1);
    }
}
```

**Правила**:

- Файл должен находиться в директории, соответствующей пакету (например, `com/example/utils/StringHelper.java`).
- Пакет указывается в начале файла с помощью `package`.

### Импорт

- **Обычный импорт**: Доступ к классам из других пакетов:
    
    ```java
    import com.example.utils.StringHelper;
    
    public class Main {
        public static void main(String[] args) {
            System.out.println(StringHelper.capitalize("привет")); // Выведет: Привет
        }
    }
    ```
    
- **Импорт с подстановкой**: Использует `*` для всех классов пакета:
    
    ```java
    import com.example.utils.*;
    ```
    
    **Рекомендация**: Избегайте `*`, используйте явный импорт для читаемости.
    
- **Статический импорт**: Доступ к статическим членам без указания класса:
    
    ```java
    import static java.lang.Math.*;
    
    public class Demo {
        public static void main(String[] args) {
            double r = sqrt(25); // Вместо Math.sqrt
            System.out.println(r); // Выведет: 5.0
        }
    }
    ```
    

### Пакет по умолчанию

- Если `package` не указан, класс попадает в **default package**.
- **Проблемы**:
    - Ограничивает масштабируемость.
    - Не рекомендуется для продакшен-кода.
- **Рекомендация**: Всегда указывайте пакет.

---

## Внутренняя реализация в JVM

### Структура классов и объектов

- **Класс**:
    - Хранится в байт-коде (`.class`), загружается в Method Area.
    - Содержит метаданные (поля, методы, конструкторы) и пул констант.
- **Объект**:
    - Создаётся в куче (Heap) через `new`.
    - Содержит заголовок (метаданные, ссылка на класс) и поля экземпляра.
- **Поля**:
    - Экземплярные поля хранятся в объекте (Heap).
    - Статические поля — в Method Area (область класса).
- **Конструкторы**:
    - Компилируются в метод `<init>`, вызываемый через `invokespecial`.
    - Выполняют инициализацию полей и вызов родительского конструктора (`super()`).
- **Инициализаторы**:
    - Нестатические: Встраиваются в `<init>` перед кодом конструктора.
    - Статические: Выполняются в методе `<clinit>` при загрузке класса.
- **Композиция**:
    - Поля-объекты хранят ссылки на другие объекты в куче.
    - Не влияет на производительность, так как это просто ссылки.

**Пример байт-кода (упрощённо)**:

```java
public class Person {
    String name;
    public Person(String name) {
        this.name = name;
    }
}
```

Байт-код:

```
.class public Person
  .field name Ljava/lang/String;
  .method public <init>(Ljava/lang/String;)V
    aload_0
    invokespecial java/lang/Object/<init>()V
    aload_0
    aload_1
    putfield Person/name Ljava/lang/String;
    return
  .end method
```

---

## Производительность и оптимизация

### Анализ производительности

- **Создание объектов**:
    - Зависит от количества полей и сложности конструктора.
    - JIT-компилятор оптимизирует вызовы конструкторов.
- **Поля**:
    - Доступ к полям экземпляра: O(1).
    - Статические поля: O(1), но могут требовать синхронизации в многопоточных приложениях.
- **Композиция**:
    - Не добавляет значительных накладных расходов, так как хранит только ссылки.
    - Может увеличить использование памяти при создании множества объектов.
- **Инициализаторы**:
    - Нестатические: Минимальные накладные расходы, встраиваются в конструктор.
    - Статические: Выполняются один раз, но сложные инициализаторы могут замедлить загрузку класса.

**Рекомендация**:

- Минимизируйте количество полей для уменьшения размера объектов.
- Избегайте сложных статических инициализаторов.

---

## Многопоточность

### Потокобезопасность классов

Классы и объекты не являются потокобезопасными по умолчанию. Особенности:

- **Поля экземпляра**:
    - Требуют синхронизации при доступе из разных потоков:
        
        ```java
        public class Person {
            private String name;
            private final Object lock = new Object();
        
            public void setName(String name) {
                synchronized (lock) {
                    this.name = name;
                }
            }
        }
        ```
        
- **Статические поля**:
    - Общие для всех объектов, требуют синхронизации:
        
        ```java
        public class Person {
            private static int populationCount;
            public static synchronized void incrementPopulation() {
                populationCount++;
            }
        }
        ```
        
- **Композиция**:
    - Компоненты (например, `Engine` в `Car`) должны быть потокобезопасными, если используются в многопоточной среде.
- **Конструкторы**:
    - Безопасны, если не публикуют объект до завершения инициализации:
        
        ```java
        public class SafeObject {
            private static SafeObject instance;
        
            private SafeObject() {} // Приватный конструктор
        
            public static synchronized SafeObject getInstance() {
                if (instance == null) {
                    instance = new SafeObject();
                }
                return instance;
            }
        }
        ```
        

**Подводные камни**:

- Неправильная синхронизация полей может привести к состоянию гонки.
- Публикация объекта до завершения конструктора (`this` в конструкторе) может вызвать проблемы.

**Рекомендация**: Используйте `synchronized`, `volatile`, `AtomicReference` или `ConcurrentHashMap` для потокобезопасности.

---

## Современные возможности Java

> **Подробнее о современных возможностях Java (Pattern Matching, Sealed-классы, Records) см. в статье [Современные возможности Java](../Современные%20возможности%20Java.md).**
> **Подробнее о стримах см. в разделе [Stream API и коллекции](../3.%20Коллекции%20и%20Stream%20API/Stream%20API.md).**

---

## Лучшие практики

### Инкапсуляция

- Используйте `private` поля и публичные геттеры/сеттеры:
    
    ```java
    public class Person {
        private String name;
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
    }
    ```
    

### Минимизация статических полей

- Используйте `static` только для констант или общего состояния.

### Предпочтение композиции

- Используйте композицию вместо наследования для гибкости:
    
    ```java
    public class Car {
        private Engine engine;
        public Car(Engine engine) { this.engine = engine; }
    }
    ```
    

### Использование record-классов

- Для неизменяемых данных:
    
    ```java
    public record Person(String name, int age) {}
    ```
    

### Потокобезопасность

- Синхронизируйте доступ к полям:
    
    ```java
    public class Counter {
        private int count;
        public synchronized void increment() { count++; }
    }
    ```
    

### Организация пакетов

- Используйте осмысленные имена пакетов (например, `com.company.module`).
- Избегайте пакета по умолчанию.

### Документирование

```java
/**
 * Представляет пользователя с именем и адресом.
 */
public class User {
    private String name;
    private Address address;
}
```

---

## Типичные проблемы и решения

### Неправильная инициализация

- **Проблема**: Поля без инициализации могут привести к `NullPointerException`.
- **Решение**: Инициализируйте поля в конструкторе или при объявлении.

### Злоупотребление статическими полями

- **Проблема**: Статические поля усложняют тестирование и многопоточность.
- **Решение**: Предпочитайте поля экземпляра или константы.

### Сложные инициализаторы

- **Проблема**: Нестатические инициализаторы усложняют читаемость.
- **Решение**: Используйте конструкторы для инициализации.

### Публикация объектов в конструкторе

- **Проблема**: Передача `this` в конструкторе может привести к доступу к неинициализированному объекту.
- **Решение**: Завершайте инициализацию перед публикацией.

### Неправильный выбор композиции или наследования

- **Проблема**: Наследование вместо композиции создаёт жесткие зависимости.
- **Решение**: Используйте композицию, если нет явного "is-a" отношения.

---

## Комплексный пример

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Logger;

// Компонент
public class Address {
    private final String city;
    private final String street;

    public Address(String city, String street) {
        this.city = city;
        this.street = street;
    }

    public String getCity() {
        return city;
    }

    public String getStreet() {
        return street;
    }
}

// Record-класс
public record Person(String name, int age, Address address) {}

// Sealed-класс
public sealed class Vehicle permits Car {
    private final String model;
    private static final Logger LOGGER = Logger.getLogger(Vehicle.class.getName());
    private static int vehicleCount;

    // Статический инициализатор
    static {
        vehicleCount = 0;
        LOGGER.info("Инициализация Vehicle: vehicleCount = 0");
    }

    public Vehicle(String model) {
        this.model = model;
        vehicleCount++;
    }

    public String getModel() {
        return model;
    }

    public static int getVehicleCount() {
        return vehicleCount;
    }
}

// Наследник
public final class Car extends Vehicle {
    private final Engine engine;
    private final Object lock = new Object();

    // Нестатический инициализатор
    {
        LOGGER.info("Создание Car");
    }

    public Car(String model, Engine engine) {
        super(model);
        this.engine = engine;
    }

    public void start() {
        synchronized (lock) {
            System.out.println(getModel() + " с двигателем " + engine.horsepower + " л.с. завелся");
        }
    }
}

// Компонент
public class Engine {
    int horsepower;

    public Engine(int horsepower) {
        this.horsepower = horsepower;
    }
}

// Демонстрация
public class ClassExample {
    private static final Logger LOGGER = Logger.getLogger(ClassExample.class.getName());

    public static void main(String[] args) {
        // Композиция
        Address address = new Address("Москва", "Ленинский проспект");
        Person person = new Person("Алиса", 25, address);
        LOGGER.info(person.name() + " живет в " + person.address().getCity());

        // Sealed-класс и композиция
        Engine engine = new Engine(200);
        Vehicle car = new Car("Toyota", engine);
        car.start();

        // Многопоточность
        ConcurrentHashMap<String, Vehicle> vehicles = new ConcurrentHashMap<>();
        vehicles.put("car1", new Car("Honda", new Engine(150)));
        new Thread(() -> vehicles.put("car2", new Car("BMW", new Engine(300)))).start();

        // Потокобезопасный перебор
        new Thread(() -> {
            vehicles.forEach((key, vehicle) -> {
                if (vehicle instanceof Car c) {
                    LOGGER.info("Потокобезопасный запуск: " + key);
                    c.start();
                }
            });
        }).start();

        // Статическое поле
        LOGGER.info("Всего машин: " + Vehicle.getVehicleCount());
    }
}
```

**Предполагаемый вывод**:

```
INFO: Инициализация Vehicle: vehicleCount = 0
INFO: Создание Car
INFO: Алиса живет в Москва
Toyota с двигателем 200 л.с. завелся
INFO: Создание Car
INFO: Создание Car
INFO: Потокобезопасный запуск: car1
Honda с двигателем 150 л.с. завелся
INFO: Потокобезопасный запуск: car2
BMW с двигателем 300 л.с. завелся
INFO: Всего машин: 3
```

---

## Вопросы для собеседования

1. Что такое класс и объект в Java? В чём разница между ними?
2. Как объявить класс? Какие модификаторы доступа можно использовать?
3. Что такое поля класса? Чем отличаются статические поля от полей экземпляра?
4. Как работает инициализация полей? Какие значения по умолчанию?
5. Что такое конструктор? Как создать перегруженные конструкторы?
6. Как работает ключевое слово `this`? Для чего оно используется?
7. Что такое инициализаторы? Чем отличаются статические от нестатических?
8. Что такое композиция классов? Когда её использовать вместо наследования?
9. Как работают модификаторы доступа? Опишите все уровни доступа.
10. Что такое пакеты в Java? Как организовать структуру пакетов?
11. Как работает импорт? Что такое статический импорт?
12. Как JVM представляет классы и объекты в памяти?
13. Как оптимизировать производительность при работе с классами?
14. Как обеспечить потокобезопасность классов?
15. Что такое record-классы? Когда их использовать?
16. Что такое sealed-классы? Как они работают?
17. Что такое pattern matching? Как его использовать?
18. Как правильно реализовать инкапсуляцию?
19. Какие проблемы могут возникнуть при работе с конструкторами?
20. Как избежать утечек памяти при работе с объектами?
21. Что такое escape analysis? Как оно влияет на производительность?
22. Как работает garbage collection для объектов?
23. Что такое reflection? Как оно связано с классами?
24. Как тестировать классы? Какие подходы использовать?
25. Какие современные возможности Java упрощают работу с классами?