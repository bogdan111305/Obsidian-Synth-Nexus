## Роль класса Object

- **Корень иерархии**: Каждый класс в Java неявно наследует `Object`, если не указан другой родитель.
- **Универсальный тип**: Позволяет ссылаться на любой объект как на `Object`, обеспечивая полиморфизм.
- **Базовое поведение**: Предоставляет методы, доступные всем объектам (`toString`, `equals`, `hashCode`, и др.).
- **Интеграция с JVM**: Используется в коллекциях, синхронизации и рефлексии.

**Пример**:

```java
Object obj = new String("Привет");
System.out.println(obj.getClass()); // Выведет: class java.lang.String
```

**Особенности**:

- Даже массивы и перечисления (`enum`) наследуют `Object`:
    
    ```java
    int[] arr = {1, 2, 3};
    System.out.println(arr instanceof Object); // true
    ```
    
- Используется в коллекциях (`List<Object>`, `Map<Object, Object>`) и потоках.

## Ключевые методы класса Object

Класс `Object` предоставляет следующие методы, доступные всем объектам:

|Метод|Назначение|
|---|---|
|`toString()`|Возвращает строковое представление объекта|
|`equals(Object obj)`|Сравнивает объекты по содержимому|
|`hashCode()`|Возвращает хэш-код объекта для коллекций (`HashMap`, `HashSet`)|
|`getClass()`|Возвращает класс объекта во время выполнения|
|`clone()`|Создаёт копию объекта (требуется `Cloneable`)|
|`finalize()` (устарел в Java 9)|Вызывается перед сборкой мусора (не рекомендуется)|
|`wait()` / `notify()` / `notifyAll()`|Механизмы синхронизации для многопоточных приложений|

### 2.1. `toString()`

- **Назначение**: Возвращает строковое представление объекта.
- **По умолчанию**: Возвращает `ClassName@HashCode` (например, `MyClass@1b6d3586`).
- **Рекомендация**: Переопределяйте для читаемого вывода.

**Пример**:

```java
public class Person {
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

**Использование**:

```java
Person person = new Person("Алиса", 25);
System.out.println(person); // Выведет: Person{name='Алиса', age=25}
```

**Подводные камни**:

- Непереопределённый `toString` даёт неинформативный результат.
- Используйте утилиты, такие как `Objects.toString`, для упрощения:
    
    ```java
    import java.util.Objects;
    
    @Override
    public String toString() {
        return Objects.toString(name) + ", " + age;
    }
    ```
    

### 2.2. `equals(Object obj)`

- **Назначение**: Сравнивает объекты по содержимому.
- **По умолчанию**: Сравнивает ссылки (`==`).
- **Контракт**:
    - Рефлексивность: `x.equals(x)` → `true`.
    - Симметричность: `x.equals(y)` → `y.equals(x)`.
    - Транзитивность: Если `x.equals(y)` и `y.equals(z)`, то `x.equals(z)`.
    - Консистентность: Многократные вызовы `x.equals(y)` дают одинаковый результат.
    - Сравнение с `null`: `x.equals(null)` → `false`.

**Пример**:

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Person other = (Person) obj;
        return age == other.age && Objects.equals(name, other.name);
    }
}
```

**Использование**:

```java
Person p1 = new Person("Алиса", 25);
Person p2 = new Person("Алиса", 25);
System.out.println(p1.equals(p2)); // true
```

**Подводные камни**:

- Неправильное переопределение `equals` может нарушить контракт, особенно в коллекциях.
- Используйте `Objects.equals` для безопасного сравнения полей.

### 2.3. `hashCode()`

- **Назначение**: Возвращает хэш-код объекта для использования в коллекциях (`HashMap`, `HashSet`).
- **Контракт**:
    - Если `x.equals(y)`, то `x.hashCode() == y.hashCode()`.
    - Хэш-код должен быть консистентным при многократных вызовах.
- **По умолчанию**: Основан на адресе объекта в памяти.

**Пример**:

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Person other = (Person) obj;
        return age == other.age && Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

**Использование**:

```java
Person p1 = new Person("Алиса", 25);
Person p2 = new Person("Алиса", 25);
System.out.println(p1.hashCode() == p2.hashCode()); // true
```

**Подводные камни**:

- Непереопределённый `hashCode` при переопределённом `equals` ломает коллекции.
- Используйте `Objects.hash` для генерации хэш-кода.

### 2.4. `getClass()`

- **Назначение**: Возвращает объект `Class`, представляющий тип объекта во время выполнения.
- **Особенности**:
    - Не переопределяется (метод `final`).
    - Используется в рефлексии и сравнении типов.

**Пример**:

```java
Object obj = new String("Привет");
System.out.println(obj.getClass()); // class java.lang.String
```

**Использование в `equals`**:

```java
if (obj == null || getClass() != obj.getClass()) return false;
```

### 2.5. `clone()`

- **Назначение**: Создаёт копию объекта (поверхностное копирование).
- **Требование**: Класс должен реализовать интерфейс `Cloneable`.
- **Особенности**:
    - Поверхностное копирование копирует ссылки на объекты.
    - Для глубокого копирования требуется дополнительная логика.

**Пример**:

```java
public class Person implements Cloneable {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public Person clone() throws CloneNotSupportedException {
        return (Person) super.clone();
    }
}
```

**Использование**:

```java
Person p1 = new Person("Алиса", 25);
Person p2 = p1.clone();
System.out.println(p1 != p2); // true (разные объекты)
System.out.println(p1.equals(p2)); // true (одинаковое содержимое)
```

**Подводные камни**:

- Без `Cloneable` вызов `clone()` бросает `CloneNotSupportedException`.
- Поверхностное копирование не копирует вложенные объекты.

**Глубокое копирование**:

```java
public class Person implements Cloneable {
    private String name;
    private Address address;

    @Override
    public Person clone() throws CloneNotSupportedException {
        Person cloned = (Person) super.clone();
        cloned.address = new Address(address.city); // Глубокое копирование
        return cloned;
    }
}
```

### 2.6. `finalize()` (устарел в Java 9)

- **Назначение**: Вызывается перед сборкой мусора.
- **Статус**: Помечен как `@Deprecated(since="9")`, не рекомендуется использовать.
- **Замена**: Используйте `try-with-resources` или `Cleaner` для управления ресурсами.

**Пример (не рекомендуется)**:

```java
@Override
protected void finalize() throws Throwable {
    System.out.println("Объект уничтожен");
}
```

**Подводные камни**:

- Нет гарантии вызова `finalize`.
- Замедляет сборку мусора.

### 2.7. `wait()`, `notify()`, `notifyAll()`

- **Назначение**: Используются для синхронизации потоков.
- **Требование**: Вызываются только в `synchronized` блоке.
- **Методы**:
    - `wait()`: Приостанавливает поток до вызова `notify` или `notifyAll`.
    - `notify()`: Пробуждает один ждущий поток.
    - `notifyAll()`: Пробуждает все ждущие потоки.

**Пример**:

```java
public class Message {
    private String message;

    public synchronized void setMessage(String message) throws InterruptedException {
        this.message = message;
        notify();
    }

    public synchronized String getMessage() throws InterruptedException {
        while (message == null) {
            wait();
        }
        return message;
    }
}
```

**Использование**:

```java
Message msg = new Message();
new Thread(() -> {
    try {
        System.out.println(msg.getMessage());
    } catch (InterruptedException e) {}
}).start();
new Thread(() -> {
    try {
        msg.setMessage("Привет");
    } catch (InterruptedException e) {}
}).start();
```

**Подводные камни**:

- Вызов вне `synchronized` блока бросает `IllegalMonitorStateException`.
- Используйте `Lock` и `Condition` для более гибкой синхронизации.

## Внутренняя реализация в JVM

- **Класс `Object`**:
    - Определён в `java.lang.Object`, загружается в Method Area.
    - Содержит нативные методы (например, `hashCode`, `clone`), реализованные в JVM.
- **Наследование**:
    - Каждый класс содержит ссылку на `Object` в метаданных.
    - Методы `Object` включаются в таблицу виртуальных методов (vtable).
- **Методы**:
    - `toString`, `equals`, `hashCode`: Реализованы в Java, могут быть переопределены.
    - `getClass`, `clone`: Нативные, вызываются через JNI.
    - `wait`, `notify`, `notifyAll`: Нативные, интегрированы с монитором объекта.
- **Память**:
    - Объекты содержат заголовок с ссылкой на класс и монитор для синхронизации.

**Пример байт-кода (упрощённо)**:

```java
public class Person {
    @Override
    public String toString() { return "Person"; }
}
```

Байт-код:

```
.class public Person
  .super java/lang/Object
  .method public toString()Ljava/lang/String;
    ldc "Person"
    areturn
  .end method
```

- **Производительность**:
    - Вызов методов `Object`: O(1) с JIT-оптимизациями.
    - `hashCode` и `equals`: Зависят от переопределения.
    - `clone`: Поверхностное копирование быстрое, глубокое требует дополнительной работы.

## Производительность

- **Методы**:
    - `toString`, `equals`, `hashCode`: Зависят от сложности переопределения.
    - `getClass`: O(1), доступ к метаданным объекта.
    - `clone`: O(1) для поверхностного копирования, O(n) для глубокого.
    - `wait`/`notify`: Накладные расходы на управление монитором.
- **Память**:
    - Все объекты содержат заголовок (8–16 байт), включающий ссылку на класс.
    - Переопределённые методы увеличивают размер vtable незначительно.
- **Рекомендация**:
    - Оптимизируйте `equals` и `hashCode` для коллекций.
    - Избегайте сложной логики в `toString`.

## Многопоточность

- **Синхронизация**:
    - Методы `wait`, `notify`, `notifyAll` используют монитор объекта.
    - Требуют `synchronized` блока:
        
        ```java
        synchronized (obj) {
            obj.wait();
        }
        ```
        
- **Потокобезопасность `equals` и `hashCode`**:
    - Переопределённые методы должны быть безопасными, если объект используется в многопоточной среде:
        
        ```java
        public class Person {
            private final String name; // Неизменяемое поле
            private final int age;
        
            @Override
            public boolean equals(Object obj) {
                if (this == obj) return true;
                if (obj == null || getClass() != obj.getClass()) return false;
                Person other = (Person) obj;
                return age == other.age && Objects.equals(name, other.name);
            }
        
            @Override
            public int hashCode() {
                return Objects.hash(name, age);
            }
        }
        ```
        
- **Подводные камни**:
    - Изменяемые поля в `equals`/`hashCode` могут нарушить контракт в коллекциях.
    - **Решение**: Используйте неизменяемые поля или синхронизацию.

## Современные возможности (Java 14+)

> **Подробнее о современных возможностях Java (Pattern Matching, Sealed-классы, Records) см. в статье [Современные возможности Java](../Современные%20возможности%20Java.md).**
> **Подробнее о стримах см. в разделе [Stream API и коллекции](../3.%20Коллекции%20и%20Stream%20API/Stream%20API.md).**

## Сравнение с альтернативами

- **Record-классы**:
    - Подходят для неизменяемых данных, автоматически реализуют методы `Object`.
    - Ограничение: `final`, не поддерживают наследование.
- **Утилиты `Objects`**:
    - Упрощают реализацию `equals`, `hashCode`, `toString`.
- **Интерфейсы**:
    - Используйте для определения поведения вместо наследования от `Object`.

**Пример с `Objects`**:

```java
import java.util.Objects;

public class Person {
    private String name;
    private int age;

    @Override
    public boolean equals(Object obj) {
        return obj instanceof Person p && Objects.equals(name, p.name) && age == p.age;
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }

    @Override
    public String toString() {
        return Objects.toString(name) + ", " + age;
    }
}
```

## Подводные камни

1. **Непереопределённые `equals`/`hashCode`**:
    
    - Нарушение контракта ломает коллекции (`HashMap`, `HashSet`).
    - **Решение**: Всегда переопределяйте оба метода вместе.
2. **Изменяемые поля в `equals`/`hashCode`**:
    
    - Изменение полей после добавления в коллекцию приводит к ошибкам.
    - **Решение**: Используйте неизменяемые поля или `record`.
3. **Неправильная синхронизация**:
    
    - Вызов `wait`/`notify` вне `synchronized` блока вызывает исключение.
    - **Решение**: Всегда используйте `synchronized`.
4. **Устаревший `finalize`**:
    
    - Не используйте, так как он ненадёжен и замедляет сборку мусора.
5. **Поверхностное копирование в `clone`**:
    
    - Копирует ссылки, а не объекты.
    - **Решение**: Реализуйте глубокое копирование вручную.

## Лучшие практики

1. **Переопределяйте `toString`, `equals`, `hashCode`**:
    
    - Используйте `Objects` для упрощения:
        
        ```java
        import java.util.Objects;
        
        @Override
        public int hashCode() {
            return Objects.hash(name, age);
        }
        ```
        
2. **Используйте `record` для данных**:
    
    - Автоматическая реализация методов `Object`:
        
        ```java
        public record Person(String name, int age) {}
        ```
        
3. **Обеспечивайте потокобезопасность**:
    
    - Используйте неизменяемые поля или синхронизацию:
        
        ```java
        public class Person {
            private final String name;
            private final int age;
        }
        ```
        
4. **Избегайте `finalize`**:
    
    - Используйте `try-with-resources` или `Cleaner`.
5. **Реализуйте `Cloneable` осторожно**:
    
    - Предпочитайте глубокое копирование для сложных объектов.
6. **Документируйте методы**:
    
    ```java
    /**
     * Возвращает строковое представление объекта Person.
     */
    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }
    ```
    

## Пример: Комплексное использование

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.logging.Logger;

// Класс с полной реализацией методов Object
public class Person implements Cloneable {
    private final String name;
    private final int age;
    private final Address address; // Композиция
    private final Object lock = new Object();
    private static final Logger LOGGER = Logger.getLogger(Person.class.getName());

    public Person(String name, int age, Address address) {
        this.name = name;
        this.age = age;
        this.address = address;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Person other = (Person) obj;
        return age == other.age && Objects.equals(name, other.name) && Objects.equals(address, other.address);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age, address);
    }

    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + ", address=" + address + "}";
    }

    @Override
    public Person clone() throws CloneNotSupportedException {
        Person cloned = (Person) super.clone();
        cloned.address = new Address(address.city); // Глубокое копирование
        return cloned;
    }

    public synchronized void waitForUpdate() throws InterruptedException {
        synchronized (lock) {
            lock.wait();
        }
    }

    public synchronized void notifyUpdate() {
        synchronized (lock) {
            lock.notify();
        }
    }
}

// Компонент для композиции
public class Address {
    String city;

    public Address(String city) {
        this.city = city;
    }

    @Override
    public boolean equals(Object obj) {
        return obj instanceof Address a && Objects.equals(city, a.city);
    }

    @Override
    public int hashCode() {
        return Objects.hash(city);
    }

    @Override
    public String toString() {
        return "Address{city='" + city + "'}";
    }
}

// Демонстрация
public class ObjectExample {
    private static final Logger LOGGER = Logger.getLogger(ObjectExample.class.getName());

    public static void main(String[] args) throws CloneNotSupportedException {
        // Создание объекта
        Address address = new Address("Москва");
        Person person = new Person("Алиса", 25, address);

        // toString
        System.out.println(person); // Person{name='Алиса', age=25, address=Address{city='Москва'}}

        // equals и hashCode
        Person person2 = new Person("Алиса", 25, new Address("Москва"));
        System.out.println(person.equals(person2)); // true
        System.out.println(person.hashCode() == person2.hashCode()); // true

        // getClass
        System.out.println(person.getClass()); // class Person

        // clone
        Person cloned = person.clone();
        System.out.println(person != cloned); // true
        System.out.println(person.equals(cloned)); // true

        // Многопоточность
        ConcurrentHashMap<Person, String> map = new ConcurrentHashMap<>();
        map.put(person, "Персона 1");
        new Thread(() -> {
            try {
                person.waitForUpdate();
                LOGGER.info("Обновление получено: " + person);
            } catch (InterruptedException e) {}
        }).start();
        new Thread(() -> {
            person.notifyUpdate();
        }).start();

        // Pattern matching
        Object obj = person;
        if (obj instanceof Person p) {
            LOGGER.info("Pattern matching: " + p);
        }
    }
}
```

**Предполагаемый вывод**:

```
Person{name='Алиса', age=25, address=Address{city='Москва'}}
true
true
class Person
true
true
INFO: Pattern matching: Person{name='Алиса', age=25, address=Address{city='Москва'}}
INFO: Обновление получено: Person{name='Алиса', age=25, address=Address{city='Москва'}}
```

## Вопросы для собеседования

1. Какую роль играет класс Object в иерархии Java?
2. Какие методы определяет Object и зачем их переопределять?
3. Как работает контракт equals/hashCode?
4. Как реализован метод clone и когда его использовать?
5. Почему finalize считается устаревшим?
6. Как работают wait/notify/notifyAll и для чего они нужны?
7. Как реализован Object в JVM (vtable, заголовок объекта)?
8. Как обеспечить потокобезопасность equals/hashCode?
9. Чем record-классы отличаются от обычных классов с точки зрения Object?
10. Какие подводные камни связаны с изменяемыми полями в equals/hashCode?
11. Как работает getClass и зачем он нужен?
12. Как реализовать глубокое копирование объекта?
13. Как использовать Object в коллекциях?
14. Как тестировать корректность equals/hashCode?
15. Какие современные возможности Java упрощают работу с Object?