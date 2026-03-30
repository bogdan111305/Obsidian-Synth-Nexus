# Ключевое слово static в Java

> `static` привязывает поле/метод/блок/вложенный класс к **классу**, а не к объекту. Существует в единственном экземпляре. Инициализируется при загрузке класса. Static поля хранятся в **Metaspace** (Java 8+). Нет доступа к `this` и нестатическим членам.
> На интервью: где хранится static, порядок static initialization, проблема Initialization On Demand Holder, static import злоупотребление.

## Связанные темы

[[Инициализация объектов с учётом наследования]], [[Java Memory Structure]], [[ClassLoaders]], [[Enum]]

---

## Static поля и методы

```java
public class Counter {
    // Static поле — одно на класс, в Metaspace (ссылка) / Heap (объект)
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
Metaspace:
  Counter.count   → int 0   (примитив — прямо в Metaspace)
  Counter.someRef → ↓       (ссылка — в Metaspace, объект — в Heap)
Heap:
  someObject
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

## Вопросы на интервью

- Где хранятся static поля? Что хранится в Metaspace vs Heap?
- Когда выполняется static initializer? Гарантирован ли thread-safety?
- Что такое Initialization On Demand Holder? Почему лучше double-checked locking?
- Почему нельзя обращаться к instance полям из static метода?
- Что произойдёт если static блок бросит исключение?

---

## Подводные камни

- **Static блок бросает исключение** — класс помечается как `failed to initialize`. Повторная загрузка → `NoClassDefFoundError`. Нельзя исправить без перезапуска JVM.
- **Static поля — память никогда не освобождается** — пока ClassLoader жив. В веб-приложениях после undeploy static поля могут удерживать большие объекты (логгеры, кэши) → Metaspace leak.
- **Static поля в тестах** — остаются между тестами если не сбросить вручную. Используй `@BeforeEach` или не используй изменяемые static состояния.
- **Thread-safety static полей** — несколько потоков обращаются к `static int counter` без синхронизации → data race. Используй `AtomicInteger` или `volatile` для простых случаев.
- **`static` метод в интерфейсе** — с Java 8. Не наследуется реализующим классом (в отличие от `default`). Вызывается только через имя интерфейса: `Interface.staticMethod()`.
