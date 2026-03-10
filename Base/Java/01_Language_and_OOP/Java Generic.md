---
title: "Java Generics — Дженерики"
tags: [java, generics, wildcards, type-erasure, pecs]
updated: 2026-03-04
---

# Java Generics (Дженерики)

> [!QUOTE] Суть
> **Generics** — параметризованные типы для типобезопасности. Работают только на уровне компилятора — в байткоде **type erasure** стирает параметры (`List<String>` → `List`). **PECS**: Producer Extends, Consumer Super (`<? extends T>` для чтения, `<? super T>` для записи).

**До дженериков:**
```java
List list = new ArrayList();
list.add("Привет");
list.add(42); // Можно добавить разные типы
String s = (String) list.get(0); // Требуется приведение
// Integer i = (Integer) list.get(1); // ClassCastException во время выполнения
```

**С дженериками:**
```java
List<String> list = new ArrayList<>();
list.add("Привет");
// list.add(42); // Ошибка компиляции
String s = list.get(0); // Приведение не требуется
```

**Преимущества:**
- Типобезопасность (ошибки ловятся на этапе компиляции)
- Нет необходимости в явном приведении типов
- Универсальность и переиспользуемость кода
- Более читаемый и самодокументируемый код

**Недостатки:**
- Сложность для новичков (особенно wildcards и type erasure)
- Ограничения из-за стирания типов

> **Вопрос на собеседовании:** Почему дженерики делают код безопаснее? Какой минус у raw types?
> **Ответ:** Дженерики позволяют компилятору проверять типы на этапе компиляции, предотвращая ошибки приведения типов и ClassCastException во время выполнения. Raw types (сырые типы) не обеспечивают такую проверку, что может привести к ошибкам в рантайме.

---

## Raw types (сырые типы)

**Raw type** — это использование обобщённого (generic) класса или интерфейса без указания параметра типа.

**Пример:**
```java
List list = new ArrayList(); // raw type
list.add("строка");
list.add(42);
```
- Здесь `List` и `ArrayList` используются без `<T>`, то есть без указания типа элементов.

**Чем опасен raw type?**
- Компилятор не проверяет типы элементов, можно добавить объекты любых типов.
- При извлечении данных требуется явное приведение типа, что может привести к ошибкам времени выполнения (`ClassCastException`).
- Использование raw types нарушает типобезопасность, ради которой и были введены дженерики.

**Правильно:**
```java
List<String> list = new ArrayList<>();
list.add("строка");
// list.add(42); // Ошибка компиляции
```

**Когда допустимо использовать raw type?**
- Только для обратной совместимости со старым кодом (до Java 5).
- В новых проектах использовать не рекомендуется.

> **Вопрос на собеседовании:** Что такое raw type и почему его использование опасно?
> **Ответ:** Raw type — это generic-класс без параметра типа. Компилятор не проверяет типы, возможны ошибки времени выполнения. Использовать только для совместимости со старым кодом.

---

## Синтаксис дженериков

### Обобщённые классы
```java
public class Box<T> {
    private T value;
    public Box(T value) { this.value = value; }
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
Box<String> stringBox = new Box<>("Привет");
Box<Integer> intBox = new Box<>(42);
```
- `T` — параметр типа, подставляется при создании экземпляра.
- Можно использовать несколько параметров: `<K, V>`.

### Обобщённые интерфейсы
```java
public interface Pair<K, V> {
    K getKey();
    V getValue();
}
public class OrderedPair<K, V> implements Pair<K, V> { ... }
Pair<String, Integer> pair = new OrderedPair<>("Возраст", 25);
```

### Обобщённые методы
```java
public static <T> void printArray(T[] array) {
    for (T elem : array) System.out.println(elem);
}
```

### Обобщённые конструкторы
```java
public class Account {
    private String id;
    public <T> Account(T id) { this.id = id.toString(); }
}
Account acc1 = new Account("ID123");
Account acc2 = new Account(456);
```

> **Важно:** Параметры типа видны только внутри класса/метода, где объявлены.

---

## Ограничения типов (Bounded Types)

### Верхняя граница (`extends`)
```java
public class NumberBox<T extends Number> { ... }
NumberBox<Integer> intBox = new NumberBox<>(42);
// NumberBox<String> stringBox = new NumberBox<>("text"); // Ошибка компиляции
```
- Можно ограничить несколькими интерфейсами: `<T extends Number & Comparable<T>>`
- Класс должен быть первым в списке ограничений.

### Ограничение снизу (`super`)
- Используется только в wildcards, не в определении типа: `List<? super Integer>`

> **Вопрос на собеседовании:** Чем отличается `extends` от `super` в дженериках?
> **Ответ:**
> - `? extends T` (ковариантность) — позволяет читать элементы как тип T, но не добавлять (кроме null). Используется, когда коллекция — источник данных (producer).
> - `? super T` (контравариантность) — позволяет добавлять элементы типа T и его подтипов, но читать только как Object. Используется, когда коллекция — потребитель данных (consumer).

---

## Wildcards (`?`) — Подстановочные типы

### Основные виды wildcards
| Wildcard         | Описание                        | Пример                   | Чтение | Запись |
|------------------|---------------------------------|--------------------------|--------|--------|
| `<?>`            | Любой тип                       | `List<?>`                | Да     | Нет    |
| `<? extends T>`  | Любой наследник T               | `List<? extends Number>` | Да     | Нет    |
| `<? super T>`    | Любой родитель T                | `List<? super Integer>`  | Нет    | Да     |

### Неограниченный wildcard (`<?>`)
```java
public void printList(List<?> list) {
    for (Object item : list) System.out.println(item);
}
```
- Можно читать как Object, нельзя добавлять (кроме null).

### Верхняя граница (`<? extends T>`)
- Используется, когда коллекция — источник данных (producer):
```java
public double sum(List<? extends Number> list) { ... }
```
- Можно читать как T, нельзя добавлять.

### Нижняя граница (`<? super T>`)
- Используется, когда коллекция — потребитель данных (consumer):
```java
public void addNumbers(List<? super Integer> list) { list.add(42); }
```
- Можно добавлять T и подтипы, читать только как Object.

### Правило PECS
- **Producer Extends, Consumer Super**: если коллекция — источник, используйте `extends`; если потребитель — `super`.

> **Вопрос на собеседовании:** Когда использовать `? extends T`, а когда `? super T`?
> **Ответ:**
> - Если коллекция только **отдаёт** данные (вы читаете из неё) — используйте `? extends T`.
> - Если коллекция только **принимает** данные (вы записываете в неё) — используйте `? super T`.
> - Это правило называется PECS: Producer Extends, Consumer Super.

---

## Стирание типов (Type Erasure)

- Дженерики реализованы через стирание типов для обратной совместимости.
- Во время компиляции параметры типа заменяются на Object или верхнюю границу.
- В рантайме нет информации о типе параметра.

**Пример:**
```java
List<String> strings = new ArrayList<>();
List<Integer> numbers = new ArrayList<>();
System.out.println(strings.getClass() == numbers.getClass()); // true
```
- Нельзя создавать экземпляры T: `new T()` — ошибка компиляции.
- Нельзя создавать массивы обобщённых типов: `new List<String>[10]` — ошибка.
- Нельзя использовать `instanceof` с параметризованным типом: `if (obj instanceof List<String>)` — ошибка.

> **Вопрос на собеседовании:** Почему нельзя создать `new T()` или `new List<String>[]`?
> **Ответ:** Из-за стирания типов (type erasure) информация о параметрах типа удаляется на этапе компиляции, и в рантайме неизвестно, какой именно тип подставлен. Поэтому нельзя создать экземпляр параметра типа или массив обобщённого типа.

---

## Дженерики и наследование

- `List<String>` не является подтипом `List<Object>`, хотя `String` — подтип `Object`.
- Для ковариантности используйте wildcards: `List<? extends Object>`.

**Пример:**
```java
List<? extends Object> objects = new ArrayList<String>();
```

### Ковариантность и контравариантность в дженериках Java

**Ковариантность** — это свойство, при котором подтип параметра типа делает весь обобщённый тип подтипом:
- В Java ковариантность реализуется через `? extends T` (верхняя граница wildcard).
- Пример: `List<? extends Number>` — можно присвоить `List<Integer>`, `List<Double>` и т.д.
- Ковариантные коллекции — только для чтения (нельзя безопасно добавлять элементы).

**Контравариантность** — это свойство, при котором супертип параметра типа делает весь обобщённый тип подтипом:
- В Java контравариантность реализуется через `? super T` (нижняя граница wildcard).
- Пример: `List<? super Integer>` — можно присвоить `List<Integer>`, `List<Number>`, `List<Object>`.
- Контравариантные коллекции — только для записи (можно добавлять элементы типа T и его подтипов, но читать только как Object).

**Определения:**
- **Ковариантность**: если `A` — подтип `B`, то `G<A>` — подтип `G<B>` (в Java — только с wildcard `extends`).
- **Контравариантность**: если `A` — подтип `B`, то `G<B>` — подтип `G<A>` (в Java — только с wildcard `super`).
- **Инвариантность**: ни ковариантности, ни контравариантности нет (обычные дженерики Java).

**Вопрос на собеседовании:**
- Чем отличается ковариантность от контравариантности?
- Почему нельзя присвоить `List<String>` переменной типа `List<Object>`?
- Как реализовать ковариантность и контравариантность в Java?
> **Ответ:** Ковариантность — через wildcard `? extends T`, контравариантность — через `? super T`.

---

## Дженерики и Reflection API

- Информация о дженериках сохраняется в метаданных класса (через `getGenericType`, `ParameterizedType` и др.).
- Стирание типов ограничивает работу с типами в рантайме, но анализ возможен через Reflection API.

**Пример:**
```java
Field field = GenericExample.class.getDeclaredField("strings");
Type type = field.getGenericType();
if (type instanceof ParameterizedType) {
    ParameterizedType pType = (ParameterizedType) type;
    System.out.println(pType.getRawType()); // interface java.util.List
    System.out.println(pType.getActualTypeArguments()[0]); // class java.lang.String
}
```

---

## Дженерики в стандартной библиотеке Java

- Все коллекции (`List`, `Set`, `Map`), компараторы, Optional, Stream API используют дженерики.
- Примеры:
```java
List<String> list = new ArrayList<>();
Map<Integer, String> map = new HashMap<>();
Optional<Double> opt = Optional.of(3.14);
```
> **Вопрос на собеседовании:** Как дженерики реализованы в стандартной библиотеке Java?
> **Ответ:** Все коллекции (`List`, `Set`, `Map`), компараторы, Optional, Stream API используют дженерики для типобезопасности и универсальности.

---

## Подводные камни и ограничения

1. **Стирание типов**: нет информации о типе в рантайме, нельзя создавать экземпляры T, массивы обобщённых типов, использовать instanceof с параметром типа.
2. **Static-поля и дженерики**: нельзя использовать параметр типа в static-полях.
3. **Исключения**: нельзя создавать обобщённые исключения.
4. **Wildcards**: неправильное использование приводит к ошибкам и запутанному коду.

---

## Heap Pollution и @SafeVarargs

**Heap Pollution** — ситуация, когда переменная параметризованного типа ссылается на объект не того типа. Это возникает при смешивании generics и varargs или при небезопасных приведениях типов.

### Проблема: varargs + generics = heap pollution

```java
// ОПАСНО: компилятор выдаст предупреждение
@SuppressWarnings("unchecked")
static <T> T[] toArray(T... args) {
    return args; // JVM создаёт Object[] вместо T[]
}

// Вызов:
String[] arr = toArray("a", "b", "c"); // ClassCastException в runtime!
// JVM создала Object[], но мы пытаемся присвоить String[]
```

**Почему это происходит**: При erasure `T... args` превращается в `Object[] args`. JVM создаёт массив типа `Object[]`, а не `String[]`. Присвоение `String[] arr = (Object[]) ...` нарушает type safety.

### @SafeVarargs — контракт безопасности

```java
// БЕЗОПАСНО: метод только читает varargs, не записывает в массив
@SafeVarargs
static <T> List<T> asList(T... args) {
    return Arrays.asList(args); // читаем элементы, не модифицируем массив
}

// НЕ БЕЗОПАСНО — нельзя применять @SafeVarargs:
@SafeVarargs // ОШИБКА: метод сохраняет ссылку на массив varargs
static <T> void addToList(List<T> list, T... args) {
    list.add(args[0]); // OK
    // НО: args — это Object[], это проблема если caller ожидает T[]
}
```

**Правило применения `@SafeVarargs`:**
- Метод не записывает в массив varargs через generic-тип
- Метод не передаёт массив varargs в "недоверенный" код
- Применяется к `final`, `static` или `private` методам (Java 9+: также к конструкторам)

### Heap Pollution через unchecked cast

```java
List<String> strings = new ArrayList<>();
// Raw type — отключает type checking:
List rawList = strings;
rawList.add(42); // компилятор не видит проблемы (только предупреждение)

// ClassCastException возникнет ПОЗЖЕ, не здесь:
String s = strings.get(0); // OK
String s2 = strings.get(1); // ClassCastException: Integer cannot be cast to String
```

**Диагностика**: `-Xlint:unchecked` при компиляции выведет все потенциально опасные места.

---

## Bridge Methods (Мостовые методы)

При наследовании и переопределении обобщённых методов компилятор генерирует **bridge methods** для обеспечения полиморфизма после type erasure.

### Проблема без bridge methods

```java
public interface Comparable<T> {
    int compareTo(T other); // После erasure → compareTo(Object other)
}

public class MyString implements Comparable<MyString> {
    @Override
    public int compareTo(MyString other) { ... } // compareTo(MyString)
}
```

После erasure интерфейс требует `compareTo(Object)`, но класс имеет `compareTo(MyString)`. Это разные сигнатуры — нет полиморфизма!

### Что генерирует компилятор

```java
// Ваш код:
public class MyString implements Comparable<MyString> {
    public int compareTo(MyString other) {
        return this.value.compareTo(other.value);
    }
}

// Компилятор ДОБАВЛЯЕТ (bridge method):
public class MyString implements Comparable<MyString> {
    public int compareTo(MyString other) { ... }  // ← ваш метод

    // Synthetic bridge method (видно через javap -p):
    public synthetic bridge int compareTo(Object other) {
        return compareTo((MyString) other); // делегирует с приведением типа
    }
}
```

**Увидеть bridge methods**: `javap -verbose -p MyString.class`
Флаги в байт-коде: `ACC_BRIDGE` + `ACC_SYNTHETIC`

### Когда ещё генерируются bridge methods

```java
// Ковариантный возвращаемый тип (Java 5+):
class Animal {
    Animal clone() { return new Animal(); }
}

class Dog extends Animal {
    @Override
    Dog clone() { return new Dog(); } // ковариантный тип

    // Компилятор генерирует:
    // synthetic bridge Animal clone() { return clone(); } // вызывает Dog.clone()
}
```

---

## Reifiable и Non-Reifiable Types

**Reifiable type (воспроизводимый тип)** — тип, информация о котором полностью доступна в runtime.

| Reifiable | Non-Reifiable |
|---|---|
| Примитивы (`int`, `long`) | `List<String>` |
| Сырые типы (`List`, `Map`) | `Map<String, Integer>` |
| Unbounded wildcards (`List<?>`) | `T` (type parameter) |
| Массивы reifiable типов (`String[]`) | `List<String>[]` (generic array) |
| Non-generic классы (`String`) | `Comparable<String>` |

### Почему generic array запрещён

```java
// Это НЕ компилируется:
List<String>[] arr = new List<String>[10]; // Generic Array Creation error

// Почему? После erasure:
List[] arr = new List[10]; // ОК

// Проблема:
Object[] objects = arr;      // OK (массивы ковариантны)
objects[0] = new List<Integer>(); // runtime: OK (оба List)
String s = arr[0].get(0);   // ClassCastException! Но нет предупреждения

// Допускается только с wildcard (reifiable type):
List<?>[] arr2 = new List<?>[10]; // OK
```

### Instanceof с generics

```java
// Запрещено (non-reifiable):
if (obj instanceof List<String>) { } // ошибка компиляции

// Разрешено (reifiable):
if (obj instanceof List<?>) { }   // OK
if (obj instanceof List) { }       // OK (raw type)

// Java 16+ Pattern Matching с generics:
if (obj instanceof List<?> list) {
    // list — List<?>, type-safe
}
```

---

## Ограничения Generics в Runtime (таблица)

| Операция | Запрещено | Альтернатива |
|---|---|---|
| `new T()` | Нет конструктора для type param | Передать `Class<T>`, использовать фабрику |
| `new T[]` | Generic array creation | `(T[]) new Object[n]` + `@SuppressWarnings` или `Array.newInstance(clazz, n)` |
| `instanceof T` | Non-reifiable | `instanceof T` с Class token |
| `T.class` | Нет class literal | Передать `Class<T> clazz` как параметр |
| `catch (T e)` | Generic exception | Ловить конкретный тип |
| `static T field` | Static не может быть generic | Убрать generic из статического контекста |
| `List<int>` | Примитивы не поддерживаются | `List<Integer>` (autoboxing) |

---

## Лучшие практики

- Всегда указывайте типы в коллекциях и обобщённых классах.
- Следуйте правилу PECS для wildcards.
- Избегайте raw types.
- Документируйте ограничения типов.
- Тестируйте сложные случаи с wildcards и наследованием.
- Компилируйте с `-Xlint:unchecked` для обнаружения heap pollution.
- Применяйте `@SafeVarargs` только когда метод не записывает в массив varargs.

---

## Interview Q&A (Senior Level)

**Q1: Что такое Heap Pollution и как оно может возникнуть в generic-коде без явных ошибок компиляции?**

> Heap Pollution возникает когда переменная параметризованного типа хранит объект несовместимого типа. Типичный сценарий: передача generic varargs. При `void method(T... args)` компилятор создаёт `Object[]`, а не `T[]`. Если метод сохранит этот массив и caller ожидает `T[]`, при присвоении будет `ClassCastException`. Компилятор выдаёт только предупреждение (не ошибку), потому что после erasure всё выглядит как `Object[]`. `@SafeVarargs` говорит компилятору "метод не нарушает type safety", подавляя предупреждение — это контракт, нарушение которого разработчик берёт на себя.

**Q2: Почему `new T()` невозможно в обобщённом коде, и какие есть обходные пути?**

> Из-за type erasure во время выполнения не существует информации о том, что такое `T`. JVM видит только `Object`, и вызвать `new Object()` вместо `new T()` семантически неверно. Обходные пути: (1) передать `Class<T> clazz` и вызвать `clazz.getDeclaredConstructor().newInstance()` — но это через reflection, медленнее; (2) передать `Supplier<T> factory` — функциональный подход без reflection; (3) использовать abstract factory method — паттерн, где подкласс знает свой тип: `abstract T createInstance()`.

**Q3: Что такое bridge method и при каких условиях его генерирует компилятор?**

> Bridge method — синтетический метод с флагами `ACC_BRIDGE + ACC_SYNTHETIC` в байт-коде, генерируемый компилятором в двух случаях: (1) При реализации generic интерфейса — если `class Foo implements Comparable<Foo>`, компилятор добавляет `compareTo(Object)` который делает downcast и вызывает `compareTo(Foo)`. Без этого polymorphism через `Comparable` не работал бы после erasure. (2) При ковариантном возвращаемом типе — `Dog clone()` в подклассе требует bridge `Animal clone()` для совместимости с виртуальной таблицей. Bridge methods видны через `javap -p`, их не следует вызывать напрямую через reflection (нужно фильтровать `isBridge()`).

**Q4: Чем `List<?>` отличается от `List<Object>` и от `List`?**

> `List<Object>` принимает только коллекции с точным типом `Object` — нельзя передать `List<String>`. `List<?>` (upper-bounded wildcard) принимает `List` любого типа, но запрещает `add()` (кроме `null`) — это read-only view. `List` (raw type) полностью отключает type checking и генерирует warning — опасно использовать. Практически: `List<?>` идеален для параметров методов, которые только читают коллекцию (`void printAll(List<?> list)`). `List<Object>` нужен когда метод гетерогенно работает с объектами (`void addAnything(List<Object> list)`).

**Q5: Как Type Erasure взаимодействует с перегрузкой методов и может ли это вызвать конфликт?**

> Да, Type Erasure может вызвать `erasure of ... is the same` ошибку компиляции. Два метода с одинаковой erasure не могут сосуществовать:
> ```java
> // НЕ КОМПИЛИРУЕТСЯ:
> void process(List<String> list) {}
> void process(List<Integer> list) {} // erasure: оба process(List)
> ```
> После erasure обе сигнатуры становятся `process(List)` — конфликт. JVM различает методы только по стёртой сигнатуре. Решения: переименовать методы или добавить дополнительный параметр для disambiguation. Это также означает, что нельзя override только по параметризации — bridge methods решают проблему полиморфизма, но не перегрузки.

---

## Мини-глоссарий и чек-лист

- **Дженерик (Generic)** — обобщённый класс, интерфейс или метод с параметром типа.
- **Wildcard (`?`)** — подстановочный тип для гибкости API.
- **Bounded Type** — ограничение типа (`extends`, `super`).
- **Type Erasure** — стирание информации о типе при компиляции.
- **Raw Type** — использование обобщённого класса без указания типа.
- **PECS** — Producer Extends, Consumer Super.

**Чек-лист для собеседования:**
- [ ] Могу объяснить, зачем нужны дженерики и их преимущества.
- [ ] Знаю синтаксис обобщённых классов, интерфейсов, методов.
- [ ] Понимаю разницу между `extends` и `super`.
- [ ] Могу объяснить, что такое wildcard и когда его использовать.
- [ ] Знаю, что такое стирание типов и его последствия.
- [ ] Понимаю ограничения дженериков (static, массивы, исключения).
- [ ] Могу привести примеры из стандартной библиотеки и Stream API.
- [ ] Знаю best practices и типичные ошибки.

---

> **Подробнее о современных возможностях Java (Pattern Matching, Sealed-классы, Records) см. в [[Современные возможности Java]].**
> **Подробнее о стримах см. в [[Java Stream API]].**