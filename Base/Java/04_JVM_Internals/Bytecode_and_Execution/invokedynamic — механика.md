# invokedynamic — механика

> `invokedynamic` (Java 7, JEP 292) — опкод байткода для динамически связанных вызовов. При первом выполнении вызывает **bootstrap method**, который возвращает `CallSite` с `MethodHandle`. Последующие вызовы идут напрямую через MH — без JVM overhead. Используется для лямбд, String concatenation (Java 9+), MethodHandles.

## Связанные темы
[[Java Bytecode — структура и опкоды]], [[MethodHandle & LambdaMetafactory]], [[JIT Compiler & Optimizations]], [[Java Reflection API]]

---

## Зачем invokedynamic

До Java 7: JVM имела 4 опкода вызова (`invokevirtual`, `invokestatic`, `invokespecial`, `invokeinterface`) — все статически связанные на уровне байткода. Динамические языки (Groovy, JRuby) не могли эффективно работать на JVM.

`invokedynamic` добавил **отложенное связывание**: кто именно будет вызван — определяется в runtime через bootstrap method.

---

## Механика работы

```
Первый вызов invokedynamic:
1. JVM смотрит на bootstrap method spec в constant pool
2. Вызывает bootstrap method (статический Java-метод)
3. Bootstrap method возвращает CallSite (с MethodHandle внутри)
4. JVM "связывает" invokedynamic с этим CallSite
5. Вызов делегируется MethodHandle из CallSite

Последующие вызовы:
- CallSite уже связан → напрямую через MethodHandle
- JIT может инлайнить MethodHandle → near-zero overhead
```

```java
// Типы CallSite:
ConstantCallSite  // target MethodHandle никогда не меняется (лямбды)
MutableCallSite   // target можно заменить (polyglot runtime)
VolatileCallSite  // замена видна немедленно всем потокам (горячая замена)
```

---

## invokedynamic для лямбда-выражений

```java
// Исходный код:
List<String> list = List.of("b", "a", "c");
list.sort((a, b) -> a.compareTo(b));
```

```
Байткод (упрощённо):
invokedynamic #3, 0   // ← один invoke для создания лямбды
invokevirtual List.sort(Comparator)
```

**Что делает bootstrap method при первом вызове:**

1. Вызывается `LambdaMetafactory.metafactory()`
2. LambdaMetafactory генерирует класс в runtime (через ASM или MethodHandles)
3. Возвращает `ConstantCallSite` с MH, создающим экземпляр функционального интерфейса

```java
// LambdaMetafactory (упрощённо):
public static CallSite metafactory(
    MethodHandles.Lookup caller,       // контекст вызова
    String invokedName,                // "compare" (имя метода SAM)
    MethodType invokedType,            // ()Comparator
    MethodType samMethodType,          // (Object,Object)int
    MethodHandle implMethod,           // ссылка на лямбда-тело
    MethodType instantiatedMethodType  // (String,String)int
) {
    // Генерирует и возвращает ConstantCallSite
}
```

**Преимущество перед анонимным классом:**
- Анонимный класс: компилятор создаёт файл `Outer$1.class` → загружается при старте
- Лямбда через `invokedynamic`: класс генерируется только при первом вызове → lazy
- При повторных вызовах без замыкания (stateless): может переиспользовать один экземпляр

---

## invokedynamic для String concatenation (Java 9+)

```java
String s = "Hello, " + name + "! Count: " + count;
```

**До Java 9 (StringBuilder):**
```
new StringBuilder
invokespecial StringBuilder.<init>
ldc "Hello, "
invokevirtual StringBuilder.append(String)
aload name
invokevirtual StringBuilder.append(String)
ldc "! Count: "
invokevirtual StringBuilder.append(String)
iload count
invokevirtual StringBuilder.append(int)
invokevirtual StringBuilder.toString()
```

**Java 9+ (invokedynamic):**
```
aload name
iload count
invokedynamic #4  // StringConcatFactory.makeConcatWithConstants
                  // bootstrap получает шаблон "Hello, \u0001! Count: \u0001"
                  // возвращает оптимальную имплементацию
```

`StringConcatFactory` может выбирать оптимальную стратегию конкатенации в зависимости от типов и количества аргументов (StringBuilder, VarHandle-based, и др.).

---

## Пример: самостоятельный invokedynamic

```java
import java.lang.invoke.*;

public class InvokeDynamicExample {
    // Bootstrap method — должен иметь точную сигнатуру:
    public static CallSite bootstrap(
            MethodHandles.Lookup lookup,
            String name,        // имя из invokedynamic байткода
            MethodType type     // тип из байткода
    ) throws NoSuchMethodException, IllegalAccessException {
        // Ищем реальный метод для вызова
        MethodHandle mh = lookup.findStatic(
            InvokeDynamicExample.class,
            "greet",
            MethodType.methodType(String.class, String.class)
        );
        return new ConstantCallSite(mh);
    }

    public static String greet(String name) {
        return "Hello, " + name + "!";
    }
}
// invokedynamic используют напрямую через ASM или Bytecode manipulation
// В обычном Java коде — через лямбды/method references
```

---

## invokedynamic и JIT

JIT обрабатывает `invokedynamic` особым образом:

```
1. При первом вызове: bootstrap → CallSite → MethodHandle
2. JIT видит ConstantCallSite → знает target не изменится
3. JIT инлайнит MethodHandle напрямую в call site
4. После warmup: overhead ≈ 0 (как прямой вызов)

MutableCallSite: JIT не может агрессивно инлайнить — target может измениться
VolatileCallSite: каждое чтение target — volatile read → overhead
```

---

## Вопросы на интервью

- Что такое `invokedynamic`? Зачем он появился в Java 7?
- Что такое bootstrap method? Что он возвращает?
- Что такое `CallSite`? Чем `ConstantCallSite` отличается от `MutableCallSite`?
- Как лямбды реализованы через `invokedynamic`? Что такое `LambdaMetafactory`?
- Как изменилась конкатенация строк в Java 9 через `invokedynamic`?
- Почему `ConstantCallSite` лучше оптимизируется JIT?

## Подводные камни

- **Bootstrap method вызывается один раз** — при первом `invokedynamic` для данного call site. Если bootstrap дорогой — latency при первом запросе. Прогрев (warmup) важен.
- **Лямбда без замыкания переиспользует экземпляр** — `x -> x.length()` без захвата внешних переменных: JVM может создать singleton. Лямбда с захватом (`() -> name.length()`) — новый объект при каждом создании лямбды.
- **`MutableCallSite.syncAll()`** — обновление target в `MutableCallSite` видно другим потокам только после `MutableCallSite.syncAll()`. Иначе JIT может закэшировать старый target.
- **`invokedynamic` нельзя "вызвать" напрямую из Java** — только через компилятор (лямбды, switch, String+) или байткод-манипуляцию (ASM, ByteBuddy).
