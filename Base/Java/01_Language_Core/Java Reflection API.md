# Java Reflection API

> Механизм для динамического анализа и изменения структуры классов в runtime. Основа Spring DI, Hibernate, JUnit, Jackson. Медленно (~10–100x vs прямой вызов), ломает инкапсуляцию, обходит compile-time проверки. Используй только когда нет альтернатив — предпочитай `MethodHandle`.

## Связанные темы
[[Java Assembling]], [[Java Memory Structure]], [[Интерфейсы]], [[MethodHandle & LambdaMetafactory]]

---

## Ключевые классы

| Класс | Назначение |
|---|---|
| `Class<T>` | Точка входа. Метаданные класса/интерфейса |
| `Field` | Поле (включая private) |
| `Method` | Метод (включая private) |
| `Constructor<T>` | Конструктор |
| `Proxy` | Генерация динамических прокси |
| `AccessibleObject` | Базовый для Field/Method/Constructor — управление доступом |
| `AnnotatedElement` | Интерфейс для чтения аннотаций (реализован всеми выше) |

---

## Получение Class<?>

```java
// Три способа:
Class<?> c1 = String.class;                     // compile-time
Class<?> c2 = "hello".getClass();               // runtime — фактический тип
Class<?> c3 = Class.forName("java.lang.String"); // по имени — ClassNotFoundException!

// Class.forName инициализирует класс (static-блоки выполняются)
// Если нужна только загрузка без инициализации:
Class<?> c4 = Class.forName("com.example.Foo", false, Thread.currentThread().getContextClassLoader());
```

---

## Работа с Field, Method, Constructor

**getDeclared\* vs get\*:**
- `getDeclaredFields()` — все поля **этого** класса (включая private), без унаследованных
- `getFields()` — все **public** поля, включая унаследованные

```java
class Person {
    private String name = "Alex";
    private String greet(String hi) { return hi + ", " + name; }
}

// Field:
Field f = Person.class.getDeclaredField("name");
f.setAccessible(true);  // обход private
Person p = new Person();
System.out.println(f.get(p));     // Alex
f.set(p, "Maria");
System.out.println(f.get(p));     // Maria

// Method:
Method m = Person.class.getDeclaredMethod("greet", String.class);
m.setAccessible(true);
String result = (String) m.invoke(p, "Hello"); // Hello, Maria

// Constructor (private singleton pattern):
Constructor<?> c = Person.class.getDeclaredConstructor(String.class);
c.setAccessible(true);
Person p2 = (Person) c.newInstance("Bob");
```

---

## Аннотации

```java
@Retention(RetentionPolicy.RUNTIME)  // только RUNTIME виден через Reflection
@Target(ElementType.METHOD)
@Repeatable(Tests.class)
@interface Test {
    String value() default "";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Tests { Test[] value(); }

// Чтение аннотаций:
Method method = MyTests.class.getDeclaredMethod("testFoo");

// Проверка наличия:
if (method.isAnnotationPresent(Test.class)) { ... }

// Получение:
Test ann = method.getAnnotation(Test.class);

// Повторяющиеся (Java 8+):
Test[] tests = method.getAnnotationsByType(Test.class);

// Простой тест-раннер через Reflection:
for (Method mth : MyTests.class.getDeclaredMethods()) {
    if (mth.isAnnotationPresent(Test.class)) {
        mth.invoke(new MyTests()); // вызываем @Test методы
    }
}
```

**RetentionPolicy:**
- `SOURCE` — только в исходниках, javac отбрасывает (например, `@Override`)
- `CLASS` — в `.class` файле, JVM не загружает (default)
- `RUNTIME` — доступна через Reflection

---

## Dynamic Proxy: внутреннее устройство

JDK Proxy генерирует новый `.class` в runtime, который реализует переданные интерфейсы и делегирует каждый вызов в `InvocationHandler.invoke()`.

```java
interface UserService {
    User findById(long id);
}

InvocationHandler handler = (proxy, method, args) -> {
    System.out.println("Before: " + method.getName());
    Object result = method.invoke(realService, args); // цепочка к реальному объекту
    System.out.println("After: " + method.getName());
    return result;
};

UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class<?>[]{ UserService.class },
    handler
);
proxy.findById(42L);
```

**Что генерирует JVM (`$Proxy0`):**

```java
// Эквивалент сгенерированного класса:
public final class $Proxy0 extends Proxy implements UserService {
    private static final Method m1; // findById(long)

    static {
        m1 = UserService.class.getMethod("findById", long.class);
    }

    public $Proxy0(InvocationHandler h) { super(h); }

    @Override
    public User findById(long id) {
        // каждый вызов = минимум 2 виртуальных вызова + boxing аргументов
        return (User) h.invoke(this, m1, new Object[]{ id }); // Object[] — allocation!
    }
}

// Посмотреть сгенерированный класс:
// Java 9+: -Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true
// Java 8:  -Dsun.misc.ProxyGenerator.saveGeneratedFiles=true
```

**Ограничения JDK Proxy:**
- Работает **только с интерфейсами** — класс напрямую нельзя проксировать
- Spring использует CGLIB (наследование) или ByteBuddy для классов без интерфейса
- `final`-классы нельзя проксировать через CGLIB
- Каждый прокси-класс занимает Metaspace

---

## MethodHandle vs Reflection

`MethodHandle` (Java 7, `java.lang.invoke`) — типобезопасная быстрая альтернатива `Method.invoke()`. JIT может инлайнить MethodHandle через `invokedynamic`.

```java
MethodHandles.Lookup lookup = MethodHandles.lookup();

// findVirtual — instance method
MethodHandle mh = lookup.findVirtual(
    String.class, "length",
    MethodType.methodType(int.class) // (returnType, paramTypes...)
);
int len = (int) mh.invoke("hello"); // ~1x прямой вызов после прогрева

// findStatic — статический метод
MethodHandle parseInt = lookup.findStatic(
    Integer.class, "parseInt",
    MethodType.methodType(int.class, String.class)
);

// invokeExact — точные типы, быстрее, но строже:
int n = (int) parseInt.invokeExact("42"); // типы должны совпадать точно
```

**Бенчмарк (JMH, приблизительно):**

| Способ | Скорость | Причина |
|---|---|---|
| Прямой вызов | 1x (baseline) | JIT инлайнит |
| MethodHandle (прогрев) | ~1–2x | JIT инлайнит через `invokedynamic` |
| MethodHandle (без прогрева) | ~5–10x медленнее | интерпретатор |
| `Method.invoke()` (cached) | ~10–50x медленнее | boxing, security checks |
| `Method.invoke()` без кэша | ~1000x медленнее | lookup при каждом вызове |

**Паттерн — кэшировать MethodHandle в статическом поле:**

```java
class FastMapper {
    private static final MethodHandle GET_NAME;
    static {
        try {
            GET_NAME = MethodHandles.lookup().findVirtual(
                Person.class, "getName",
                MethodType.methodType(String.class));
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    String getName(Person p) throws Throwable {
        return (String) GET_NAME.invokeExact(p); // быстрее invoke: точные типы
    }
}
```

> [!INFO] `invoke` vs `invokeExact`
> `invoke` делает приведение типов автоматически. `invokeExact` требует точного совпадения типов на стеке вызова — иначе `WrongMethodTypeException`. `invokeExact` быстрее.

---

## Java 9+ Module System и Reflection

JPMS (Project Jigsaw) ограничивает `setAccessible(true)` для закрытых модулей:

```java
// Ошибка без opens:
Field f = String.class.getDeclaredField("value");
f.setAccessible(true);
// → InaccessibleObjectException:
//   "module java.base does not 'opens java.lang' to unnamed module"

// Решение 1 — module-info.java:
// opens com.example.internal to my.framework;

// Решение 2 — JVM флаги (для тестов/инструментов, не production):
// --add-opens java.base/java.lang=ALL-UNNAMED

// Решение 3 — MethodHandles.privateLookupIn (правильный способ для фреймворков):
MethodHandles.Lookup privateLookup = MethodHandles.privateLookupIn(
    TargetClass.class, MethodHandles.lookup()
);
// Требует: TargetClass.module opens пакет для вызывающего модуля
VarHandle vh = privateLookup.findVarHandle(
    TargetClass.class, "fieldName", String.class
);
String value = (String) vh.get(target); // работает без setAccessible!
```

Spring 6+ и Hibernate 6+ используют `MethodHandles.privateLookupIn` + `VarHandle` вместо `Field.setAccessible(true)`.

---

## Вопросы на интервью

- Чем `getDeclaredFields()` отличается от `getFields()`?
- Что делает `setAccessible(true)` внутри JVM? Что такое `AccessibleObject`?
- Почему `Method.invoke()` медленнее прямого вызова?
- Что такое Dynamic Proxy? Как работает `$Proxy0` внутри?
- Чем JDK Proxy отличается от CGLIB? Когда Spring использует каждый?
- Что такое `MethodHandle`? Чем `invokeExact` отличается от `invoke`?
- Почему MethodHandle быстрее Reflection после прогрева? Роль JIT и `invokedynamic`.
- Что изменила Java 9 в работе с Reflection? Что такое `opens` в `module-info.java`?
- Как Spring/Hibernate работают с полями в Java 17+ без нарушения JPMS?
- Для чего нужен `RetentionPolicy.RUNTIME`? Что произойдёт с `RetentionPolicy.CLASS`?
- Как реализовать простой test runner, используя только Reflection?

## Подводные камни

- **Кэшируй Field/Method/MethodHandle** — lookup дорог (~1000x). Создавай один раз в static-блоке, переиспользуй.
- **`setAccessible(true)` + JPMS** — в Java 9+ без `opens` получишь `InaccessibleObjectException`. Флаг `--add-opens` — костыль, не production-решение.
- **Proxy только для интерфейсов** — JDK `Proxy.newProxyInstance` не работает с классами. Для классов нужен CGLIB/ByteBuddy, но `final` классы нельзя.
- **boxing в `Method.invoke()`** — примитивные аргументы упаковываются в `Object[]`, что даёт allocation на каждый вызов. В hot path используй `MethodHandle.invokeExact`.
- **`RetentionPolicy.CLASS`** — аннотация есть в `.class` файле, но Reflection её не видит. Для runtime-анализа обязателен `RUNTIME`.
- **Аннотации не наследуются** — `@Inherited` работает только для аннотаций на **классах**, не на методах/полях. `method.getAnnotation()` не найдёт аннотацию из суперкласса.
- **`Class.forName()` инициализирует класс** — выполняются static-блоки. Если нужна только загрузка — используй трёхаргументный вариант с `initialize=false`.
- **Имена параметров недоступны без `-parameters`** — `Parameter.getName()` вернёт `arg0`, `arg1`. Spring Boot включает этот флаг в компиляцию автоматически.
