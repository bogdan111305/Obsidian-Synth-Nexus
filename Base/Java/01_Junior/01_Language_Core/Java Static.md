# Ключевое слово static в Java

> `static` привязывает поле/метод/блок/вложенный класс к **классу**, а не к объекту. Существует в единственном экземпляре. Инициализируется при загрузке класса. Static поля хранятся в **Heap** — как часть объекта-зеркала `java.lang.Class` (Java 8+); в Metaspace лежат только метаданные класса (байткод, constant pool, vtable). Нет доступа к `this` и нестатическим членам.
> На интервью: где хранится static, порядок static initialization, проблема Initialization On Demand Holder, static import злоупотребление.

## Связанные темы

[[Инициализация объектов с учётом наследования]], [[Java Memory Structure]], [[ClassLoaders]], [[Enum]]

---

## Static поля и методы

```java
public class Counter {
    // Static поле — одно на класс, хранится в Heap (в объекте-зеркале Class)
    private static int count = 0;

    // Static метод — нет доступа к this, нестатическим полям
    public static int getCount() { return count; }

    // Static final константа — inline'd компилятором
    public static final int MAX = 100;

    public Counter() { count++; }
}

// Доступ через имя класса (не через объект — компилятор выдаст warning):
Counter.getCount();
System.out.println(Counter.MAX); // компилятор inline'd: System.out.println(100)
```

**Память:**
```
Heap (объект-зеркало java.lang.Class<Counter>):
  Counter.count   → int 0   (примитив — прямо в объекте-зеркале)
  Counter.someRef → ↓       (ссылка — в объекте-зеркале, объект — тоже в Heap)
Heap:
  someObject
Metaspace:
  метаданные класса Counter (байткод методов, constant pool, vtable)
```

---

## Static initializer block

```java
public class DatabaseConfig {
    private static final DataSource dataSource;
    private static final Map<String, String> settings;

    static {
        // Выполняется ОДИН РАЗ при первой загрузке класса
        // Порядок: сначала static поля (сверху вниз), затем static блоки
        Properties props = loadProperties("db.properties");
        settings = Collections.unmodifiableMap(props);
        dataSource = new HikariDataSource(props);
    }

    // Несколько static блоков выполняются в порядке объявления:
    static { System.out.println("First"); }
    static { System.out.println("Second"); }
}
```

**Когда класс загружается:**
- При первом `new DatabaseConfig()`
- При первом обращении к static члену
- При `Class.forName("DatabaseConfig")`
- При явной загрузке ClassLoader'ом

**Гарантия потокобезопасности:** инициализация класса (выполнение static-полей и static-блоков) сама по себе thread-safe — это прямая гарантия JVM (JVM Spec §12.4). Если несколько потоков одновременно триггерят загрузку одного и того же класса, JVM пропускает через инициализацию только один поток, а остальные блокируются на этот момент и ждут её завершения — двойного выполнения static-блока не бывает. Именно на этой гарантии, а не на ручной синхронизации, построен паттерн Initialization On Demand Holder ниже.

**Если static-блок бросает исключение:** класс переходит в состояние `failed to initialize` (Erroneous). Любая последующая попытка обратиться к классу — в том же или другом потоке, хоть сразу, хоть намного позже — заканчивается не повторной попыткой инициализации, а `NoClassDefFoundError`, оборачивающим исходную причину. JVM не даёт классу «шанс на пересдачу»: раз инициализация упала, класс считается сломанным до перезапуска JVM (новой загрузки другим ClassLoader'ом).

---

## Initialization On Demand Holder — lazy Singleton

```java
// Лучший lazy Singleton без явной синхронизации:
public class LazySingleton {
    private LazySingleton() {}

    // Holder загружается ТОЛЬКО при первом вызове getInstance()
    private static final class Holder {
        static final LazySingleton INSTANCE = new LazySingleton();
    }

    public static LazySingleton getInstance() {
        return Holder.INSTANCE;  // JVM гарантирует: class init — thread-safe
    }
}
// JVM spec §12.4: инициализация класса thread-safe, нет двойной инициализации
```

Почему лучше double-checked locking: нет `volatile`, нет `synchronized`, нет ошибок с JMM.

---

## Static context — ограничения

```java
public class Example {
    private int instanceField = 10;
    private static int staticField = 20;

    // В static методе нет доступа к this и instance членам:
    public static void staticMethod() {
        System.out.println(staticField);     // OK
        // System.out.println(instanceField); // ОШИБКА компиляции — no implicit this
        // System.out.println(this);          // ОШИБКА — нет this в static context
    }

    // Но можно через явный экземпляр:
    public static void process(Example e) {
        System.out.println(e.instanceField); // OK — явная ссылка на объект
    }
}
```

**Почему так:** instance-поле физически живёт внутри конкретного объекта в Heap — чтобы его прочитать, нужна ссылка на этот объект. Обычный (нестатический) метод получает такую ссылку неявно как первый скрытый параметр `this` при вызове `obj.method()`. Static-метод вызывается через имя класса, а не через объект (`Example.staticMethod()`), поэтому у него просто нет `this`, который можно было бы подставить, — обратиться к `instanceField` из статического контекста в принципе не к чему привязать, если явно не передать объект параметром, как в `process(Example e)`.

---

## Static import

```java
import static java.lang.Math.*;
import static java.util.Collections.*;

double r = sqrt(pow(x, 2) + pow(y, 2)); // Math.sqrt, Math.pow
List<String> sorted = singletonList("a"); // Collections.singletonList

// Используй только для часто используемых констант/методов
// Избегай для неочевидных имён — ухудшает читаемость
```

---

## Подводные камни

- **Static блок бросает исключение** — класс помечается как `failed to initialize`. Повторная загрузка → `NoClassDefFoundError`. Нельзя исправить без перезапуска JVM.
- **Static поля — память никогда не освобождается** — пока ClassLoader жив. В веб-приложениях после undeploy static поля могут удерживать большие объекты (логгеры, кэши) → Metaspace leak.
- **Static поля в тестах** — остаются между тестами если не сбросить вручную. Используй `@BeforeEach` или не используй изменяемые static состояния.
- **Thread-safety static полей** — несколько потоков обращаются к `static int counter` без синхронизации → data race. Используй `AtomicInteger` или `volatile` для простых случаев.
- **`static` метод в интерфейсе** — с Java 8. Не наследуется реализующим классом (в отличие от `default`). Вызывается только через имя интерфейса: `Interface.staticMethod()`.
