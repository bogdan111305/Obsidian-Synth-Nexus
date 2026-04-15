# Valhalla и коллекции

> **Главная мотивация Valhalla для коллекций**: `ArrayList<Integer>` сейчас хранит ссылки на boxed Integer объекты. После Valhalla — сможет хранить `int` напрямую в массиве, без boxing overhead и cache miss. **Specialized generics** (JEP 218/402) — ключевой механизм.

## Связанные темы
[[Value Classes и Primitive Classes]], [[Object Identity — что меняется]], [[Интерфейс List]], [[Интерфейс Map]], [[Java Stream API]]

---

## Проблема: Boxing в Generic Collections

```java
// Сейчас: ArrayList<Integer>
//   Object[] elementData — массив ссылок на heap objects
//
// Memory layout для [1, 2, 3, 4, 5]:
//   elementData: [ref0, ref1, ref2, ref3, ref4]  ← 5 указателей
//   Heap:        Integer(1), Integer(2), ...      ← 5 объектов по 16 байт
//
// Доступ к элементу = 2 memory access (ref → object → field)
// Cache miss гарантирован (объекты разбросаны по heap)

List<Integer> nums = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    nums.add(i);  // boxing: int → Integer
}
// Память: ~20 МБ вместо ~4 МБ для int[]
// GC: 1M дополнительных объектов в heap
```

## Specialized Generics (Valhalla)

После Valhalla generic типы смогут быть специализированы для value/primitive types:

```java
// Будущий Java (концептуально):
List<int> nums = new ArrayList<int>();  // специализация для int
// Компилятор генерирует ArrayList.class специализированный для int
// elementData: int[] — плоский массив примитивов

// Или с value class:
value class Point { int x, y; }
List<Point> points = new ArrayList<Point>();
// elementData: Point[] → [x0,y0, x1,y1, ...]  ← flat layout
```

**Примитивные специализации**: `ArrayList<int>`, `HashMap<long, double>` — без boxing.

## Проблема: Type Erasure

Текущая реализация generics использует **type erasure**: `List<T>` → `List<Object>` в байткоде. Это несовместимо с primitive/value specialization:

```java
// После erasure:
class ArrayList<T> {
    Object[] elementData;  // ← всегда Object[]
}

// Для int specialization нужно:
class ArrayList$int {
    int[] elementData;  // ← специализированный
}
```

**Решение Valhalla**: **Reified Generics** или **Value-Based Specialization** — JVM создаёт отдельные версии generic классов для каждого type argument.

Статус: сложная работа, JEP ещё не финализирован.

## Что доступно сейчас без Valhalla

### Primitive коллекции (third-party):

```java
// Eclipse Collections
MutableIntList list = IntLists.mutable.empty();
list.add(1); list.add(2);
// Хранится как int[] — zero boxing

MutableLongObjectMap<String> map = LongObjectMaps.mutable.empty();
map.put(1L, "one");
// long → String без Long boxing

// HPPC (High Performance Primitive Collections)
IntArrayList list = new IntArrayList();
LongLongHashMap map = new LongLongHashMap();

// Agrona (для HFT/networking)
IntArrayList = new org.agrona.collections.IntArrayList();
```

### Arrays vs Collections для performance:

```java
// Горячий path → массивы вместо коллекций
int[] data = new int[size];  // cache-friendly, no GC overhead
// vs
List<Integer> data = new ArrayList<>(size);  // boxing + pointer indirection
```

## HashMap и Value Classes

`HashMap<Key, Value>` при переходе Key на value type:

```java
value class CustomerId { long id; }

// Сейчас:
Map<CustomerId, Order> orders = new HashMap<>();
// CustomerId — identity object → hashCode() + equals() по объекту

// После Valhalla:
// CustomerId — value class → hashCode() по полям автоматически
// HashMap работает корректно (за счёт field-based equals/hashCode)
// Но: нет IdentityHashMap semantics (что логично)
```

## ArrayList<Value> — Memory Layout

```
ArrayList<Point> (сейчас, Point = identity class):
  Object[]  elementData = [ref→P0, ref→P1, ref→P2, ...]
  Heap:                   [P0{x,y}, P1{x,y}, ...]
  Stride:  8 bytes per ref + 16 bytes per object = 24 bytes/element

ArrayList<Point> (после Valhalla, Point = value class):
  Point[]   elementData = [x0, y0, x1, y1, x2, y2, ...]  ← flat!
  Stride:  8 bytes per element
  Cache lines: 8 Points / 64-byte cache line  (vs 2-3 с boxing)
```

## Stream API с Value Types

```java
// Сейчас: Stream<Integer> boxing overhead
IntStream.range(0, 1_000_000)
    .map(i -> i * 2)
    .sum();  // IntStream — специализированный, без boxing

// После Valhalla: Stream<T> где T = value class, без boxing
Stream<Point> points = Stream.of(new Point(1,2), new Point(3,4));
// JVM может специализировать под Point если он value class
```

## Когда это будет

| Фича | JEP | Статус (2026) |
|------|-----|---------------|
| Value Classes | JEP 401 | Preview Java 23+ |
| Primitive Classes | JEP 402 | Preview Java 24 |
| Specialized Generics | Нет финал. JEP | В разработке |
| Collections specialization | — | Зависит от Generics |
| Integer как value | — | После Specialized Generics |

Полная коллекция-поддержка — Java 28-30 (ориентировочно).

---

## Вопросы на интервью
- Почему `ArrayList<Integer>` неэффективна по памяти?
- Что такое Specialized Generics и почему Type Erasure мешает?
- Как third-party библиотеки решают проблему boxing сейчас?
- Что изменится в HashMap при использовании Value Class как ключа?
- Почему flat array layout value objects лучше для CPU cache?

## Подводные камни
- Eclipse Collections / HPPC нужны уже сейчас для hot path с примитивами, не ждать Valhalla
- Specialized Generics могут увеличить размер bytecode (code bloat) — JVM должна хранить отдельные версии классов
- При миграции к value class проверяй: нет ли `synchronized` на объектах, нет ли `IdentityHashMap` с этими типами, нет ли code `ref == null` для nullable value
