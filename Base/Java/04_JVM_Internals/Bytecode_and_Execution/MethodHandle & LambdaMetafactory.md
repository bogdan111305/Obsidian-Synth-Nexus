# MethodHandle & LambdaMetafactory

> **MethodHandle** — типобезопасный "указатель на метод" в JVM. Быстрее рефлексии: JIT инлайнит `invokeExact` как прямой вызов. **LambdaMetafactory** + `invokedynamic` — механизм реализации лямбд: не анонимный класс-файл, а динамически генерируемая реализация в памяти.
> На интервью: `invokeExact` vs `invoke`, почему лямбды через `invokedynamic` а не анонимные классы, VarHandle memory ordering, `privateLookupIn` для фреймворков, String concat в Java 9+.

## Связанные темы
[[Java Reflection API]], [[CAS и Unsafe]], [[Java Generic]], [[JIT Compiler & Optimizations]], [[Java Agents & Instrumentation API]]

---

## Reflection vs MethodHandle

```java
// Reflection: медленно, нет JIT-инлайнинга
Method m = String.class.getMethod("length");
int len = (int) m.invoke("hello"); // boxing + security checks каждый раз

// MethodHandle: JIT инлайнит как прямой вызов!
MethodHandle mh = MethodHandles.lookup()
    .findVirtual(String.class, "length", MethodType.methodType(int.class));
int len = (int) mh.invokeExact("hello"); // компилируется в прямой вызов
```

| Способ | Overhead | JIT Inlining |
|---|---|---|
| Прямой вызов | 0 | Да |
| MethodHandle (`invokeExact`) | ~0% | Да (после прогрева) |
| MethodHandle (`invoke`) | ~5% | Частично (boxing) |
| Reflection (setAccessible) | ~50-200% | Нет |

---

## MethodHandles.Lookup — контекст доступа

```java
MethodHandles.Lookup lookup = MethodHandles.lookup();        // права текущего класса
MethodHandles.Lookup pub = MethodHandles.publicLookup();     // только public API

// Java 9+: приватный lookup чужого класса (для фреймворков):
MethodHandles.Lookup priv = MethodHandles.privateLookupIn(
    TargetClass.class,
    MethodHandles.lookup() // caller должен иметь доступ к модулю
);
// priv может искать private/package-private поля и методы TargetClass
```

**Основные методы find:**
```java
// instance method:
MethodHandle mh = lookup.findVirtual(String.class, "toUpperCase",
    MethodType.methodType(String.class));

// static method:
MethodHandle parseInt = lookup.findStatic(Integer.class, "parseInt",
    MethodType.methodType(int.class, String.class));

// constructor:
MethodHandle ctor = lookup.findConstructor(StringBuilder.class,
    MethodType.methodType(void.class, String.class)); // конструктор: void return

// field getter/setter:
MethodHandle getter = lookup.findGetter(MyClass.class, "value", int.class);
MethodHandle setter = lookup.findSetter(MyClass.class, "value", int.class);

// super/private (только тот же класс):
MethodHandle superStr = lookup.findSpecial(Object.class, "toString",
    MethodType.methodType(String.class), MyClass.class);
```

---

## invokeExact vs invoke

```java
MethodHandle mh = lookup.findVirtual(String.class, "charAt",
    MethodType.methodType(char.class, int.class));

// invokeExact — точное совпадение типов, максимальная скорость:
char c = (char) mh.invokeExact("hello", 1); // 'e' — JIT: прямой вызов
// mh.invokeExact("hello", Integer.valueOf(1)) → WrongMethodTypeException!

// invoke — автоматические конверсии (boxing, widening):
char c2 = (char) mh.invoke("hello", Integer.valueOf(1)); // работает, чуть медленнее
```

---

## Комбинаторы MethodHandle

```java
// bindTo — частичное применение (currying):
MethodHandle helloLen = strlen.bindTo("hello"); // "hello".length()
int len = (int) helloLen.invoke();              // 5

// filterReturnValue — трансформация результата:
MethodHandle intToStr = lookup.findStatic(Integer.class, "toString",
    MethodType.methodType(String.class, int.class));
MethodHandle strlenAsStr = MethodHandles.filterReturnValue(strlen, intToStr);
String s = (String) strlenAsStr.invoke("hello"); // "5"

// filterArguments — трансформация аргументов:
MethodHandle trimmer = lookup.findVirtual(String.class, "trim",
    MethodType.methodType(String.class));
MethodHandle trimmedLen = MethodHandles.filterArguments(strlen, 0, trimmer);
int n = (int) trimmedLen.invoke("  hi  "); // 2, не 6
```

---

## VarHandle — замена Unsafe (Java 9+)

Типобезопасные атомарные операции без `sun.misc.Unsafe`. Используется в `java.util.concurrent`.

```java
public class AtomicCounter {
    private volatile int count = 0;

    private static final VarHandle COUNT;
    static {
        try {
            COUNT = MethodHandles.lookup()
                .findVarHandle(AtomicCounter.class, "count", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    public int getAndIncrement() {
        return (int) COUNT.getAndAdd(this, 1); // атомарный fetch-and-add
    }

    public boolean compareAndSet(int expected, int update) {
        return COUNT.compareAndSet(this, expected, update);
    }

    // Memory ordering — от слабого к сильному:
    public int getPlain()   { return (int) COUNT.get(this); }        // нет барьеров
    public int getOpaque()  { return (int) COUNT.getOpaque(this); }  // нет reorder для этой переменной
    public int getAcquire() { return (int) COUNT.getAcquire(this); } // LoadLoad + LoadStore после
    public void setRelease(int v) { COUNT.setRelease(this, v); }     // StoreStore перед
    // volatile: полный барьер с обеих сторон (дорого)
}
```

**Массивы:**
```java
VarHandle arr = MethodHandles.arrayElementVarHandle(int[].class);
arr.setVolatile(array, 5, 42);
boolean ok = arr.compareAndSet(array, 5, 42, 100); // CAS на array[5]
```

---

## LambdaMetafactory — как работают лямбды

Лямбды — не обычные анонимные классы на диске. Компилятор генерирует `invokedynamic`, JVM при первом вызове создаёт реализацию в памяти.

**Что делает компилятор:**
```java
// Ваш код:
Supplier<Integer> s = () -> 42;

// javac генерирует:
// 1. Приватный статический метод:
private static Integer lambda$0() { return 42; }

// 2. Инструкцию invokedynamic с bootstrap методом:
// invokedynamic "get":()LSupplier;
//   Bootstrap: LambdaMetafactory.metafactory(lookup, "get", ...)
//   → JVM генерирует анонимный класс impl Supplier в памяти
//   → CallSite кэшируется → повторные вызовы без генерации
```

**Программное использование (для фреймворков):**
```java
// Создать Function<String, String> из метода через LambdaMetafactory:
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle target = lookup.findVirtual(String.class, "toLowerCase",
    MethodType.methodType(String.class));

CallSite callSite = LambdaMetafactory.metafactory(
    lookup,
    "apply",                                               // SAM метод
    MethodType.methodType(Function.class),                 // factory type
    MethodType.methodType(Object.class, Object.class),     // SAM erased
    target,                                                // implementation
    MethodType.methodType(String.class, String.class)      // SAM instantiated
);

Function<String, String> toLower = (Function<String, String>) callSite.getTarget().invoke();
toLower.apply("HELLO"); // → "hello", JIT инлайнит как прямой вызов
```

**Быстрый getter для сериализатора (без рефлексии в hot path):**
```java
public static Function<Object, Object> createGetter(Field field) throws Throwable {
    MethodHandles.Lookup lookup = MethodHandles.privateLookupIn(
        field.getDeclaringClass(), MethodHandles.lookup());
    MethodHandle getter = lookup.unreflectGetter(field);

    CallSite cs = LambdaMetafactory.metafactory(
        lookup, "apply",
        MethodType.methodType(Function.class),
        MethodType.methodType(Object.class, Object.class),
        getter,
        MethodType.methodType(field.getType(), field.getDeclaringClass())
    );
    return (Function<Object, Object>) cs.getTarget().invoke();
}

// Один раз создать, кэшировать, вызывать миллионы раз:
Function<Object, Object> nameGetter = createGetter(User.class.getDeclaredField("name"));
String name = (String) nameGetter.apply(user); // как entity.getName(), JIT инлайнит
```

---

## invokedynamic за пределами лямбд

```
invokedynamic используется в:
- Лямбды и method references
- String concatenation (Java 9+) → StringConcatFactory
- Records toString/equals/hashCode (Java 16+)
- Switch pattern matching (Java 21+)
- Dynamic proxies в фреймворках

String concat до Java 9:
  "Hello " + name → new StringBuilder().append("Hello ").append(name).toString()
  → bytecode фиксирован, StringBuilder overhead

String concat Java 9+:
  invokedynamic → StringConcatFactory генерирует оптимальный код
  → знает типы и количество аргументов заранее
  → аллоцирует массив нужного размера сразу
  → ~15-30% быстрее StringBuilder-based подхода
```

---

## Вопросы на интервью

- Чем `invokeExact` отличается от `invoke`? Когда `invokeExact` бросает исключение?
- Почему лямбды реализованы через `invokedynamic`, а не как анонимные `.class` файлы?
- Как `LambdaMetafactory` используется для создания быстрых геттеров без рефлексии?
- Что такое `privateLookupIn`? Зачем он нужен фреймворкам (Jackson, Hibernate)?
- Чем VarHandle лучше `Unsafe`? Какие memory ordering режимы он предоставляет?
- Как изменилась String concatenation в Java 9? Почему она стала быстрее?
- Когда использовать `acquire`/`release` вместо `volatile`?

---

## Подводные камни

- **`invokeExact` с autoboxing** — `mh.invokeExact(42)` где параметр `Integer` → `WrongMethodTypeException`. Нужно явное `Integer.valueOf(42)` или используй `invoke`. Частая ошибка при портировании с рефлексии.
- **Lookup объект нельзя передавать** — `MethodHandles.lookup()` создаёт Lookup с правами вызывающего класса. Если передать его в другой класс или сохранить в static поле — права остаются от создателя, но вызов происходит из другого контекста. `privateLookupIn` решает это явно.
- **CallSite кэшировать, не создавать повторно** — `LambdaMetafactory.metafactory()` это дорогая операция (генерация класса). Кэшируй результирующий `Function`/`Supplier` в static поле или ConcurrentHashMap по ключу. Повторный вызов `metafactory` = повторная генерация анонимного класса → Metaspace leak.
- **VarHandle: `get()` не volatile** — `COUNT.get(this)` = plain read без барьеров. Если нужна видимость между потоками — `getVolatile()` или `getAcquire()`. Ошибка: заменить `AtomicInteger.get()` на `VarHandle.get()` и получить данные не видимые другому потоку.
- **MethodHandle + generics** — MethodHandle работает со стёртыми типами. `findVirtual(List.class, "get", MethodType.methodType(Object.class, int.class))` — параметр `int` (примитив), не `Integer`. Точное соответствие дескриптору метода в байт-коде, а не в исходнике.
- **Bridge методы в bridge/method resolution** — `findVirtual` при поиске метода может найти bridge метод если он имеет подходящую сигнатуру стёртых типов. Используй `m.isBridge()` через reflection при `unreflect()` чтобы убедиться что берёшь нужный overload.
