---
title: "Java — Сериализация и Десериализация"
tags: [java, serialization, deserialization, serializable, externalizable, transient]
updated: 2026-08-15
---

# Java Serialization and Deserialization

> **Сериализация** — преобразование объекта в поток байт (`ObjectOutputStream`), **десериализация** — обратный процесс (`ObjectInputStream`). Требует `implements Serializable` (маркерный интерфейс, поля пишутся рефлексией, объект при чтении создаётся через `Unsafe.allocateInstance` — без вызова конструктора) либо `Externalizable` (полный ручной контроль, конструктор вызывается). `serialVersionUID` — версия класса для проверки совместимости, `transient` — поле исключено из потока. Применяется для персистентности состояния, передачи объектов по сети (RMI, JMX) и кэширования.
> На интервью: `Serializable` vs `Externalizable`, зачем `serialVersionUID`, `writeReplace`/`readResolve`, формат потока, почему десериализация недоверенных данных — RCE, `ObjectInputFilter`, сериализация `record`/`enum`.

## Связанные темы
[[NIO Networking]], [[Java Bytecode — структура и опкоды]], [[ClassLoaders]], [[Java Reflection API]]

---

## 1. Serializable vs Externalizable

Оба варианта делают объект переносимым в байты, но принципиально по-разному:

| | `Serializable` | `Externalizable` |
|---|---|---|
| Тип | Маркерный интерфейс (без методов) | Требует `writeExternal`/`readExternal` |
| Кто пишет поля | JVM автоматически (рефлексия), все не-`transient` | Разработчик вручную, все поля |
| Конструктор при чтении | Не вызывается (`Unsafe.allocateInstance`) | Вызывается public no-arg конструктор |
| Производительность | Медленнее — рефлексия, метаданные класса в потоке | Быстрее — компактный поток, нет рефлексии |
| Контроль формата | Ограниченный (`writeObject`/`readObject`) | Полный |
| Наследование полей | Сериализуются, если родитель тоже `Serializable` | Разработчик отвечает сам |

Несериализуемое поле (`Thread`, сокет, лямбда с несериализуемым захватом) в `Serializable`-классе, не помеченное `transient`, роняет запись `NotSerializableException` — в рантайме, не в компиляции.

```java
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private int age;
    private transient String password; // не сериализуется → null после чтения

    public Person(String name, int age, String password) { /* ... */ }

    public static void main(String[] args) throws Exception {
        try (var oos = new ObjectOutputStream(new FileOutputStream("person.ser"))) {
            oos.writeObject(new Person("Алексей", 30, "secret"));
        }
        try (var ois = new ObjectInputStream(new FileInputStream("person.ser"))) {
            Person p = (Person) ois.readObject(); // p.password == null
        }
    }
}
```

```java
public class Employee implements Externalizable {
    private String name;
    private int id;

    public Employee() {} // обязателен — JVM вызывает его перед readExternal

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
}
```

---

## 2. Управление процессом: serialVersionUID, transient, кастомная логика

**`serialVersionUID`.** Если не задан явно, JVM генерирует его из структуры класса (имена полей/методов/сигнатуры). Любое изменение класса → новый UID → старые сериализованные данные становятся несовместимы → `InvalidClassException` при чтении. Всегда задавай явно:
```java
private static final long serialVersionUID = 1L;
```
Сгенерировать конкретное значение можно через IDE (IntelliJ/Eclipse) или утилиту `serialver com.example.Person`.

**`transient`** — поле исключается из потока, получает значение по умолчанию (`null`/`0`) при чтении. Нужно восстанавливать вручную, если это не то, что требуется (см. пример `Department` ниже).

**`writeObject`/`readObject`** — точечная кастомизация без отказа от автосериализации:
```java
private void writeObject(ObjectOutputStream out) throws IOException {
    out.defaultWriteObject();          // сериализует не-transient поля как обычно
    out.writeUTF(encrypt(password));   // + собственная логика поверх
}

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
    in.defaultReadObject();
    this.password = decrypt(in.readUTF());
}
```
Вызов `defaultWriteObject`/`defaultReadObject` обязателен, если часть полей должна сериализоваться стандартным способом.

**`writeReplace`/`readResolve`** — подмена объекта до/после (де)сериализации целиком. Классика — защита синглтона от появления второго экземпляра через десериализацию:
```java
public class Singleton implements Serializable {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }

    private Object readResolve() { return INSTANCE; } // без этого — новый объект в куче!
}
```

**Enum и Record — особые случаи:**
- **Enum** сериализуется только по имени константы (`name()`), поля в поток не пишутся; `readResolve()` неявно возвращает `Enum.valueOf(...)`. Собственный `readObject`/`writeObject`/`readResolve` в enum игнорируется JVM.
- **Record** (Java 16+), в отличие от обычного класса, **всегда вызывает канонический конструктор** при десериализации — единственное официальное исключение из правила «конструктор не вызывается». Значит валидация внутри канонического конструктора автоматически защищает record от повреждённого/подделанного потока.

---

## 3. Формат потока и механика (де)сериализации

```
Поток: AC ED 00 05 <class descriptor + serialVersionUID> <значения полей...>
       magic  version
```

**Запись:** JVM рефлексией обходит не-`transient` поля (включая `private`), пишет метаданные класса (имя, `serialVersionUID`), затем значения; коллекции и массивы — рекурсивно по элементам. Для `Externalizable` просто вызывается `writeExternal`.

**Чтение (`ObjectInputStream.readObject()`), пошагово:**
```
1. readStreamHeader() → проверка магического числа AC ED 00 05
2. readClassDesc()    → имя класса и serialVersionUID из потока
3. resolveClass()     → Class.forName(className)     ← точка проверки безопасности (см. §5)
4. Allocate object БЕЗ конструктора (Unsafe.allocateInstance) — кроме record, см. §2
5. defaultReadFields()→ установка не-transient полей
6. readObject(), если определён                       ← точка входа gadget-цепочек
7. readResolve(), если определён                       ← защита синглтона / enum
```
Ключевой факт для раздела о безопасности: объект материализуется из сырых байт потока **до** вызова `readObject` и без единой строчки конструктора — контролировать здесь нечего, кроме `resolveClass` и фильтров.

---

## 4. Производительность

- Сериализация дороже за счёт рефлексии и записи метаданных класса при первом использовании типа (кэшируются в `ObjectStreamClass`, повторные вызовы дешевле).
- `Externalizable` быстрее `Serializable`: нет рефлексии, компактный поток (нет имён полей — только то, что явно написано).
- Большие/циклические графы объектов обрабатываются корректно (JVM ведёт таблицу handle для уже записанных ссылок), но чем крупнее граф — тем дороже проход.
- Практические рычаги: `transient` для лишнего, `Externalizable` в hot path, минимизация размера графа, при серьёзных требованиях — Protobuf/Kryo вместо Java-сериализации (§6).

---

## 5. Безопасность: десериализация как вектор RCE

> Десериализация Java-native данных из недоверенного источника — не гипотетическая, а одна из самых опасных категорий уязвимостей (Remote Code Execution). Показательный пример — CVE-2015-4852 через gadget chain в Apache Commons Collections ≤ 3.2.1.

**Механизм атаки (gadget chain):** атакующему не нужно внедрять свой код — только собрать валидные байты потока из классов, уже присутствующих в classpath.
```
1. При readObject() JVM вызывает readObject() каждого вложенного объекта в графе
2. Commons Collections ≤ 3.2.1:
   LazyMap.get() → ChainedTransformer.transform()
     → InvokerTransformer("exec", Runtime.getRuntime()) → Runtime.exec(...)
3. ysoserial генерирует такие payload'ы для тестирования:
   java -jar ysoserial.jar CommonsCollections1 "touch /tmp/pwned" > payload.ser
   cat payload.ser | nc vulnerable-server 8080
```
Log4Shell (CVE-2021-44228) — атака того же семейства «доверие к недоверенному вводу ведёт к произвольному коду» (там — через JNDI lookup, не через `ObjectInputStream`), полезно держать в голове как соседний, но механически другой пример.

**Векторы:** Java RMI, JMX, HTTP-эндпоинты, принимающие `application/x-java-serialized-object`, любой кастомный `ObjectInputStream` на сетевой границе.

**Защита, от базового к строгому:**
```java
// Java 9+ (JEP 290): белый список классов через ObjectInputFilter
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.myapp.*;java.util.*;java.lang.*;!*");
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(filter);

// Глобальный фильтр на весь JVM (например, в premain или static init):
ObjectInputFilter.Config.setSerialFilter(filter);

// Java 17+ (JEP 415): фабрика контекстных фильтров — разные правила
// для разных потоков/подсистем через системное свойство jdk.serialFilterFactory

// До JEP 290 — единственный способ: явная проверка в resolveClass
ObjectInputStream ois2 = new ObjectInputStream(stream) {
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
        if (!ALLOWED_CLASSES.contains(desc.getName()))
            throw new InvalidClassException("Blocked: " + desc.getName());
        return super.resolveClass(desc);
    }
};

// Лучшая защита: вообще не десериализовать недоверенные байты Java-сериализацией —
// JSON (Jackson/Gson), Protobuf, Avro на любой внешней границе
```

Риски помимо прямого RCE: DoS через специально сконструированные глубокие или циклические графы объектов (раздувают время/память обработки), утечка приватных полей при неаккуратном кастомном `readObject`.

---

## 6. Альтернативы

| Формат | Когда использовать |
|---|---|
| JSON (Jackson, Gson) | Внешние API, читаемость, кросс-языковая совместимость |
| XML (JAXB, XStream) | Legacy-интеграции, схемы |
| Protobuf / Avro | Высокая производительность, строгая схема, межсервисный RPC |
| Kryo | Быстрая бинарная сериализация внутри JVM (не для внешних границ доверия) |
| Redis / Ehcache | Кэш состояния вместо сериализации в файл |

```java
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(person);
Person restored = mapper.readValue(json, Person.class);
```

---

## Комплексный пример

Кастомная (де)сериализация с `transient`-полем, которое нужно не занулить, а пересоздать логикой:

```java
public class Department implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private List<Person> employees;
    private transient Logger logger = Logger.getLogger(Department.class.getName());

    public Department(String name) {
        this.name = name;
        this.employees = new ArrayList<>();
    }

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        logger.info("Сериализация отдела: " + name);
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        logger = Logger.getLogger(Department.class.getName()); // transient — восстанавливаем вручную
        logger.info("Десериализация отдела: " + name);
    }
}
```

---

## Вопросы на интервью

- Чем `Serializable` отличается от `Externalizable`? Что быстрее и почему?
- Зачем нужен `serialVersionUID`? Что произойдёт, если не задать его явно?
- Как работают `writeReplace`/`readResolve`? Как они защищают синглтон?
- Вызывается ли конструктор класса при десериализации? Есть ли исключение из этого правила?
- Как устроен формат потока `ObjectOutputStream` (magic number, class descriptor, порядок шагов чтения)?
- Почему десериализация недоверенных данных — RCE? Что такое gadget chain и зачем нужен `resolveClass`?
- Какие уровни защиты есть: `ObjectInputFilter`, `resolveClass`, отказ от Java-сериализации?
- Чем сериализация `enum` и `record` отличается от обычного класса?

---

## Подводные камни

- **Несовместимость версий** — изменение полей класса без явного `serialVersionUID` → `InvalidClassException` на старых сериализованных данных. Решение: всегда задавать `serialVersionUID`, тестировать совместимость при рефакторинге класса.
- **Десериализация недоверенных данных = RCE** — см. §5. Единственная надёжная защита — не десериализовывать чужие байты Java-сериализацией вообще; если необходимо — строгий `ObjectInputFilter` с белым списком.
- **`NotSerializableException` в рантайме, не в компиляции** — любое несериализуемое поле (`Thread`, сокет, лямбда с несериализуемым захватом), не помеченное `transient`, ломает запись только при первом реальном вызове `writeObject` — обнаруживается тестами, а не компилятором.
- **`transient`-поле без восстановления в `readObject`** — остаётся `null`/`0` навсегда, если не пересоздано вручную (как `logger` в примере `Department` выше). Забыли восстановить → NPE при первом использовании после чтения объекта.
- **Утечки через графы объектов** — сериализация тянет весь связанный граф по сильным ссылкам; неаккуратная расстановка `transient` может как раздувать поток лишними данными, так и терять нужное состояние.
- **Тихое проглатывание ошибок чтения** — `catch (IOException | ClassNotFoundException e) { e.printStackTrace(); }` без логирования скрывает как баги совместимости версий, так и попытки отправить вредоносный payload. Логируй `resolveClass`/фильтр-отказы отдельно от обычных IO-ошибок.
- **Производительность на больших графах** — глубокая рекурсивная сериализация вложенных коллекций дорога; для hot path предпочти `Externalizable` или уход от Java-сериализации к Protobuf/Kryo.
