# Object Identity — что меняется

> **Object Identity** — свойство объекта иметь уникальный адрес в памяти, позволяющий отличить два объекта с одинаковым состоянием. Project Valhalla вводит объекты без identity (Value Objects), что ломает ряд предположений Java-кода и требует пересмотра паттернов.

## Связанные темы
[[Value Classes и Primitive Classes]], [[Valhalla и коллекции]], [[Типы ссылок в Java (Reference Types)]], [[Concurrent/CAS и Unsafe]]

---

## Что такое Object Identity сейчас

```java
// Identity = уникальный адрес объекта в памяти
String a = new String("hello");
String b = new String("hello");

a == b;                          // false — разные адреса, разная identity
a.equals(b);                     // true  — одинаковое состояние
System.identityHashCode(a);      // уникален для a (пока не переедет GC)
System.identityHashCode(b);      // другой
synchronized(a) { ... }          // лок на конкретный объект a
```

Identity используется для:
1. **Referential equality** (`==`)
2. **Synchronization** (`synchronized`, `Object.wait/notify`)
3. **Identity hash code** (`System.identityHashCode`)
4. **Reference tracking** (WeakReference, identity-based containers)
5. **Mutable state** (изменяемые поля)

## Что теряет Value Object

```java
value class Point { int x, y; }

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);

// p1 == p2: ??? 
// Для value objects == сравнивает состояние (как equals)
// p1 и p2 "одинаковы" — нет смысла различать их по адресу

// synchronized(p1) {}  // ← CompileError или RuntimeException
// Нет identity → нет объектного lock'а
```

## Иерархия типов после Valhalla

```
java.lang.Object
  ├── IdentityObject (интерфейс, все обычные классы)
  │     ├── String (останется identity)
  │     ├── List, Map, ...
  │     └── Ваши обычные классы
  └── ValueObject (интерфейс, value classes)
        ├── Point, Money, Complex
        └── (будущие) Integer, Long, Double
```

Все существующие классы автоматически реализуют `IdentityObject`.
Value classes реализуют `ValueObject` (marker interface).

## Synchronized — почему запрещён

```java
// Synchronized работает через object header (Mark Word):
// - хранит thin lock (thread ID)
// - inflated до heavyweight Monitor
// Value object не имеет стабильного адреса в памяти
// → нет Mark Word → нет места хранить lock state

value class Counter {
    int value;
    
    // synchronized void increment() { ... }  // ← CompileError
    
    // Альтернатива: явный lock объект
}

// Если нужна мутация — используй Identity Object:
class AtomicCounter {
    private volatile int value;
    synchronized void increment() { value++; }
}
```

## Identity Hash Code — проблема

```java
// System.identityHashCode() вычисляется из адреса объекта (обычно)
// Value object может копироваться, переезжать, не иметь stable address

System.identityHashCode(new Point(1, 2));  // что вернуть?
// Решение Valhalla: для value types → возвращает hash от полей
// Но тогда нарушается контракт: identityHashCode != address-based
```

## Wrapper Classes: Integer, Long, Double

Одна из главных целей Valhalla — сделать `Integer`, `Long`, `Double` value classes:

```java
// Сейчас:
Integer a = 127;  // cached pool
Integer b = 127;
a == b;  // true (из кэша)

Integer c = 1000;
Integer d = 1000;
c == d;  // false (!!) — разные объекты, разная identity

// После Valhalla (Integer как value class):
Integer c = 1000;
Integer d = 1000;
c == d;  // true — сравнение по значению, нет identity
         // Breaking change для кода который сравнивает Integer через ==
```

> [!WARNING]
> Если существующий код использует `==` для сравнения `Integer` вне диапазона [-128, 127] — поведение изменится. Это один из главных breaking changes Valhalla.

## Влияние на Collections

```java
// IdentityHashMap — требует identity objects
IdentityHashMap<Object, String> map = new IdentityHashMap<>();
map.put(new Point(1,2), "a");
map.put(new Point(1,2), "b");
// Сейчас: два разных entry (разные identity)
// С value Point: один entry (одинаковое состояние = одинаковый "identity")
// → IdentityHashMap будет запрещён или изменён для value objects

// WeakReference<ValueObject>
// Нет смысла: value object не имеет stable identity
// → WeakReference<Point> → компиляционная ошибка или предупреждение
```

## Обратная совместимость

Valhalla разработан с учётом совместимости:
- Существующий код продолжает работать (все классы ← IdentityObject)
- `value` keyword — новый модификатор, не конфликтует
- Migration path: постепенный переход wrapper классов к value semantics
- Source/binary compatibility maintenance для большинства кода

```java
// Код который ЛОМАЕТСЯ с Valhalla:
Integer x = 1000, y = 1000;
assert x == y;  // было false, станет true → assertion не сломается, но логика может

// Код который работает корректно:
Integer x = 1000, y = 1000;
assert x.equals(y);  // всегда true — безопасно
```

---

## Вопросы на интервью
- Что такое Object Identity в Java?
- Для чего используется identity сейчас (4 примера)?
- Почему synchronized запрещён на Value Objects?
- Как изменится поведение `==` для Integer после Valhalla?
- Что такое IdentityObject и ValueObject интерфейсы?

## Подводные камни
- `IdentityHashMap` и `WeakHashMap` семантически несовместимы с Value Objects — review при миграции кода
- Паттерн "double-checked locking с volatile ссылкой на value object" работает иначе — нет guarantee что old/new значение — разные объекты
- Код типа `if (obj == SENTINEL)` где SENTINEL — value class → неожиданно всегда true если SENTINEL и obj имеют одинаковые поля
