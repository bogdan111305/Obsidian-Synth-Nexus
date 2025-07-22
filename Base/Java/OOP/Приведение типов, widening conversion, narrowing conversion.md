## 1. Приведение примитивных типов

Примитивные типы (`byte`, `short`, `int`, `long`, `float`, `double`, `char`, `boolean`) поддерживают два вида приведения: **расширение** (widening) и **сужение** (narrowing).

### 1.1. Иерархия ширины типов (Widening Conversion)

Расширение типа — преобразование меньшего типа в больший, всегда безопасное и выполняемое автоматически (неявно).

**Иерархия**:

```
byte → short → int → long → float → double
char → int → long → float → double
```

**Пример**:

```java
byte b = 10;
int i = b; // Неявное расширение: byte (8 бит) → int (32 бита)
System.out.println(i); // 10

char c = 'A';
int i2 = c; // Неявное расширение: char (16 бит) → int (32 бита)
System.out.println(i2); // 65 (код Unicode для 'A')
```

**Байт-код (упрощённый, для `int i = b`)**:

```java
iload_1     // Загрузка byte
i2i         // Расширение до int (неявное)
istore_2    // Сохранение в i
```

**Особенности**:

- Расширение не теряет данные, так как больший тип имеет достаточную ёмкость.
- JVM использует инструкции вроде `i2l` (int → long), `l2f` (long → float).
- `boolean` не участвует в приведениях.

### 1.2. Сужение типа (Narrowing Conversion)

Сужение типа — преобразование большего типа в меньший, требующее явного кастинга. Может привести к потере данных или переполнению.

**Пример**:

```java
int i = 1000;
byte b = (byte) i; // Явное сужение: int (32 бита) → byte (8 бит)
System.out.println(b); // -24 (обрезка до младших 8 бит: 1000 = 0x3E8, byte = 0xE8 = -24)

double d = 3.14159;
int i2 = (int) d; // Явное сужение: double → int, дробная часть отбрасывается
System.out.println(i2); // 3
```

**Байт-код (упрощённый, для `byte b = (byte) i`)**:

```java
iload_1     // Загрузка int
i2b         // Сужение до byte
istore_2    // Сохранение в b
```

**Особенности**:

- **Обрезка битов**: Берутся младшие биты (например, `int` 1000 → `byte` обрезает до 8 бит).
- **Отбрасывание дробной части**: При приведении `float`/`double` к целым типам дробная часть отбрасывается.
- **Переполнение**: Если значение выходит за пределы диапазона (например, `byte` [-128, 127]), происходит обрезка.
- **Крайние случаи**:
    
    ```java
    double d = Double.POSITIVE_INFINITY;
    int i = (int) d; // Integer.MAX_VALUE
    System.out.println(i); // 2147483647
    ```
    

## 2. Приведение ссылочных типов

Ссылочные типы (объекты, массивы, интерфейсы) поддерживают приведение вверх (upcasting) и вниз (downcasting) по иерархии наследования.

### 2.1. Upcasting (Приведение вверх)

Upcasting — преобразование объекта подкласса к типу суперкласса или интерфейса. Безопасно и не требует явного кастинга.

**Пример**:

```java
class Animal {}
class Dog extends Animal {}
class Cat extends Animal {}

Dog dog = new Dog();
Animal animal = dog; // Upcasting: Dog → Animal
```

**Байт-код (упрощённый)**:

```java
aload_1     // Загрузка Dog
astore_2    // Сохранение как Animal (без дополнительных инструкций)
```

**Особенности**:

- Безопасно, так как подкласс совместим с суперклассом.
- Доступны только методы/поля суперкласса.
- Используется для полиморфизма.

**Пример с интерфейсом**:

```java
import java.util.*;

List<String> list = new ArrayList<>();
Collection<String> coll = list; // Upcasting: ArrayList → Collection
```

### 2.2. Downcasting (Приведение вниз)

Downcasting — преобразование объекта суперкласса к типу подкласса. Требует явного кастинга и может вызвать `ClassCastException`.

**Пример**:

```java
Animal animal = new Dog();
Dog dog = (Dog) animal; // Downcasting: Animal → Dog (безопасно)

Animal animal2 = new Animal();
Dog dog2 = (Dog) animal2; // ClassCastException: animal2 не Dog
```

**Байт-код (упрощённый, для `(Dog) animal`)**:

```java
aload_1             // Загрузка объекта
checkcast Dog       // Проверка и приведение к Dog
astore_2            // Сохранение
```

**Зачем нужен downcasting?**

- Для доступа к методам/полям подкласса:
    
    ```java
    class Dog extends Animal {
        void bark() { System.out.println("Woof!"); }
    }
    Animal animal = new Dog();
    ((Dog) animal).bark(); // Woof!
    ```
    

### 2.3. Проверка с `instanceof`

Оператор `instanceof` проверяет, является ли объект экземпляром класса, интерфейса или их подтипов, предотвращая `ClassCastException`.

**Пример**:

```java
Animal animal = new Dog();
if (animal instanceof Dog) {
    Dog dog = (Dog) animal;
    dog.bark(); // Безопасно
}
```

**Байт-код (упрощённый)**:

```java
aload_1             // Загрузка объекта
instanceof Dog      // Проверка типа
ifne <label>        // Условный переход
```

**Особенности**:

- Возвращает `true`, если объект совместим с типом (включая подклассы).
- Для `null` возвращает `false`.
- Работает с классами, интерфейсами, массивами.

**Пример с интерфейсом**:

```java
interface Flyable {}
class Bird implements Flyable {}
Bird b = new Bird();
System.out.println(b instanceof Flyable); // true
System.out.println(b instanceof Bird); // true
System.out.println(null instanceof Flyable); // false
```

## 3. Pattern Matching для `instanceof` (Java 16+)

Pattern matching упрощает проверку и приведение, объединяя `instanceof` и кастинг.

**Пример**:

```java
Object obj = "Hello";
if (obj instanceof String s) {
    System.out.println(s.toUpperCase()); // HELLO
} else {
    System.out.println("Not a String");
}
```

**Байт-код (упрощённый)**:

```java
aload_1             // Загрузка объекта
instanceof String   // Проверка
ifne <label>        // Условный переход
checkcast String    // Приведение
astore_2            // Сохранение в s
```

**Пример с `switch` (Java 17+)**:

```java
Object obj = new Dog();
String result = switch (obj) {
    case String s -> "String: " + s;
    case Dog d -> "Dog: " + d.bark();
    default -> "Unknown";
};
System.out.println(result); // Dog: Woof!
```

**Преимущества**:

- Устраняет дублирование кода.
- Упрощает читаемость.
- Поддерживает сложные проверки (например, с условиями):
    
    ```java
    if (obj instanceof String s && s.length() > 3) {
        System.out.println(s); // Только строки длиной > 3
    }
    ```
    

## 4. Приведение с коллекциями: `EnumSet` и `EnumMap`

Перечисления (`enum`) часто используются с `EnumSet` и `EnumMap`, которые требуют приведения при работе с интерфейсами `Set` и `Map`.

### 4.1. `EnumSet`

`EnumSet` — специализированное множество для перечислений, использующее битовую карту.

**Пример**:

```java
import java.util.*;

enum Day { MONDAY, TUESDAY, WEDNESDAY }
Set<Day> set = EnumSet.of(Day.MONDAY, Day.TUESDAY); // Upcasting: EnumSet → Set
if (set instanceof EnumSet) {
    EnumSet<Day> enumSet = (EnumSet<Day>) set; // Downcasting
    System.out.println(enumSet); // [MONDAY, TUESDAY]
}
```

**Особенности**:

- **Реализация**: Битовая карта (`long` для ≤64 элементов, `long[]` для >64).
- **Производительность**: O(1) для операций (`add`, `remove`, `contains`).
- **Память**: 1 бит на элемент.
- **Потокобезопасность**: Не потокобезопасен, требует синхронизации:
    
    ```java
    Set<Day> syncSet = Collections.synchronizedSet(EnumSet.of(Day.MONDAY));
    ```
    

### 4.2. `EnumMap`

`EnumMap` — специализированная карта с ключами `enum`, использующая массив.

**Пример**:

```java
Map<Day, String> map = new EnumMap<>(Day.class); // Upcasting: EnumMap → Map
map.put(Day.MONDAY, "Start of week");
if (map instanceof EnumMap) {
    EnumMap<Day, String> enumMap = (EnumMap<Day, String>) map; // Downcasting
    System.out.println(enumMap.get(Day.MONDAY)); // Start of week
}
```

**Особенности**:

- **Реализация**: Массив, индексированный `ordinal` ключа.
- **Производительность**: O(1) для операций (`put`, `get`, `remove`).
- **Память**: ~8–16 байт на запись.
- **Потокобезопасность**: Требует синхронизации:
    
    ```java
    Map<Day, String> syncMap = Collections.synchronizedMap(new EnumMap<>(Day.class));
    ```
    

**Сравнение**:

|Коллекция|Операции|Память|Типобезопасность|
|---|---|---|---|
|`EnumSet`|O(1)|1 бит/элемент|Только `enum`|
|`HashSet`|O(1) ср.|~32–48 байт/элемент|Любые объекты|
|`EnumMap`|O(1)|~8–16 байт/запись|Ключи — `enum`|
|`HashMap`|O(1) ср.|~32–48 байт/запись|Любые ключи|

**Пример интеграции**:

```java
EnumSet<Day> workdays = EnumSet.of(Day.MONDAY, Day.TUESDAY);
EnumMap<Day, Integer> hours = new EnumMap<>(Day.class);
workdays.forEach(day -> hours.put(day, 8));
System.out.println(hours); // {MONDAY=8, TUESDAY=8}
```

## 5. JVM-реализация

- **Примитивы**:
    - Расширение: Инструкции вроде `i2l`, `f2d`.
    - Сужение: Инструкции вроде `i2b`, `d2i` (обрезка битов или дробной части).
- **Ссылочные типы**:
    - Upcasting: Без инструкций, так как подкласс совместим.
    - Downcasting: `checkcast` проверяет метаданные класса в Metaspace.
- **instanceof**: Инструкция `instanceof` проверяет иерархию класса объекта.

**Байт-код (для `instanceof` и `checkcast`)**:

```java
Animal a = new Dog();
if (a instanceof Dog) {
    Dog d = (Dog) a;
}
```

```java
aload_1             // Загрузка a
instanceof Dog      // Проверка типа
ifne <label>        // Условный переход
aload_1             // Загрузка a
checkcast Dog       // Приведение
astore_2            // Сохранение
```

## 6. Производительность

- **Примитивы**:
    - Расширение: O(1), минимальные затраты.
    - Сужение: O(1), но возможна потеря данных.
- **Ссылочные типы**:
    - Upcasting: O(1), без затрат.
    - Downcasting: O(1), `checkcast` проверяет тип.
- **instanceof**: O(1), но зависит от глубины иерархии.
- **EnumSet/EnumMap**: O(1) для всех операций, быстрее `HashSet`/`HashMap`.

**Рекомендация**:

- Минимизируйте downcasting в горячих участках.
- Используйте `EnumSet`/`EnumMap` для перечислений.

## 7. Дополнительные сценарии

### 7.1. Дженерики

```java
List<?> list = new ArrayList<String>();
if (list instanceof ArrayList) {
    ArrayList<?> arrayList = (ArrayList<?>) list;
    System.out.println(arrayList.getClass()); // class java.util.ArrayList
}
```

**Предупреждение**: Проверяйте тип элементов:

```java
List<Object> list = new ArrayList<>();
// List<String> strList = (List<String>) list; // Unchecked warning
```

### 7.2. Массивы

Массивы ковариантны, но это может вызвать `ArrayStoreException`:

```java
Dog[] dogs = new Dog[1];
Animal[] animals = dogs; // Upcasting
animals[0] = new Cat(); // ArrayStoreException
```

### 7.3. Вложенные классы

```java
class Outer {
    class Inner {}
}
Outer.Inner inner = new Outer().new Inner();
Object obj = inner;
if (obj instanceof Outer.Inner) {
    Outer.Inner casted = (Outer.Inner) obj;
}
```

### 7.4. Аннотации

```java
@interface MyAnnotation {}
@MyAnnotation class MyClass {}
MyClass obj = new MyClass();
System.out.println(obj instanceof MyAnnotation); // false (аннотации не наследуются)
```

## 8. Частые ошибки и решения

1. **ClassCastException**:
    
    ```java
    Animal a = new Animal();
    Dog d = (Dog) a; // ClassCastException
    ```
    
    - **Решение**: Используйте `instanceof` или pattern matching.
2. **Потеря данных при сужении**:
    
    ```java
    int i = 1000;
    byte b = (byte) i; // -24
    ```
    
    - **Решение**: Проверяйте диапазон:
        
        ```java
        if (i >= Byte.MIN_VALUE && i <= Byte.MAX_VALUE) {
            byte b = (byte) i;
        }
        ```
        
3. **Дженерики и unchecked warnings**:
    
    ```java
    List<Object> list = new ArrayList<>();
    // List<String> strList = (List<String>) list; // Unchecked warning
    ```
    
    - **Решение**: Используйте `List<?>` или проверяйте элементы.
4. **Модификация коллекций при итерации**:
    
    ```java
    EnumSet<Day> set = EnumSet.of(Day.MONDAY);
    for (Day d : set) { set.add(Day.TUESDAY); } // ConcurrentModificationException
    ```
    
    - **Решение**: Используйте итераторы или стримы.

## 9. Лучшие практики

1. **Используйте upcasting для полиморфизма**:
    
    ```java
    Animal animal = new Dog();
    ```
    
2. **Проверяйте перед downcasting**:
    
    ```java
    if (animal instanceof Dog d) {
        d.bark();
    }
    ```
    
3. **Предпочитайте pattern matching (Java 16+)**:
    
    ```java
    if (obj instanceof String s) {
        System.out.println(s.toUpperCase());
    }
    ```
    
4. **Используйте `EnumSet`/`EnumMap` для перечислений**:
    
    ```java
    EnumSet<Day> days = EnumSet.of(Day.MONDAY);
    EnumMap<Day, String> map = new EnumMap<>(Day.class);
    ```
    
5. **Проверяйте диапазон при сужении**:
    
    ```java
    int i = 1000;
    if (i >= Byte.MIN_VALUE && i <= Byte.MAX_VALUE) {
        byte b = (byte) i;
    }
    ```
    
6. **Избегайте downcasting**:
    - Используйте интерфейсы или абстрактные методы:
        
```java
interface Barkable {
	void bark();
}
class Dog extends Animal implements Barkable {
	public void bark() { System.out.println("Woof!"); }
}
```