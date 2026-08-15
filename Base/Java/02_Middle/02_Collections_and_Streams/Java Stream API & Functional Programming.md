# Java Stream API & Functional Programming

> **Stream** — ленивый одноразовый pipeline: Источник → Промежуточные операции (lazy) → Терминальная операция (запускает выполнение). Лямбды реализованы через `invokedynamic` + `LambdaMetafactory`, не анонимные классы. Non-capturing лямбды — singleton.
> На интервью: stateless vs stateful операции, почему `sorted()` блокирует с infinite stream, capturing vs non-capturing лямбды, 4 типа method references, когда parallelStream вредит, Gatherers (Java 24).

## Связанные темы
[[Общая иерархия коллекций]], [[MethodHandle & LambdaMetafactory]], [[Java Generic]]

---

## Stream lifecycle

```java
// Pipeline: Head → StatelessOp → StatelessOp → StatefulOp → TerminalOp
List<String> result = list.stream()          // Head (Spliterator)
    .filter(s -> s.length() > 3)            // StatelessOp — lazy
    .map(String::toUpperCase)               // StatelessOp — lazy
    .sorted()                               // StatefulOp — буферизирует ВСЕ!
    .limit(5)                               // short-circuit
    .collect(Collectors.toList());          // TerminalOp — ЗАПУСКАЕТ выполнение
```

**Ключевые свойства:**
- **Lazy** — промежуточные операции не выполняются до терминальной
- **One-shot** — после терминальной операции stream закрыт (`IllegalStateException`)
- **Stateless operations** (`filter`, `map`, `peek`) — каждый элемент независим
- **Stateful operations** (`sorted`, `distinct`, `limit`) — нужны данные о других элементах

```java
// Создание stream:
list.stream()
Arrays.stream(arr)
Stream.of("a", "b", "c")
Stream.iterate(1, n -> n + 1)         // бесконечный
Stream.generate(Math::random)         // бесконечный
IntStream.range(0, 10)                // примитивный (нет boxing)
Files.lines(path)                     // lazy file reading
```

---

## Внутреннее устройство Stream API

### Иерархия AbstractPipeline

Каждый шаг pipeline — объект, связанный двусвязным списком (`previousStage` / `nextStage`):

```
AbstractPipeline
  ├── ReferencePipeline.Head          — source stage (list.stream())
  ├── ReferencePipeline.StatelessOp   — filter(), map(), peek()
  ├── ReferencePipeline.StatefulOp    — sorted(), distinct(), limit()
  └── TerminalOp (interface)          — collect(), forEach(), count()

// Каждый AbstractPipeline хранит:
AbstractPipeline {
    AbstractPipeline previousStage;  // ссылка назад по цепочке
    AbstractPipeline sourceStage;    // ссылка на Head (для Spliterator)
    int depth;                       // позиция в pipeline (для оптимизаций)
    int combinedFlags;               // ORDERED | SIZED | DISTINCT | SORTED | ...
}
```

### Sink chain — как элемент проходит pipeline

При вызове терминальной операции pipeline собирает **Sink chain** — обёрнутые вызовы в обратном направлении:

```
list.stream().filter(pred).map(fn).collect(toList())

Сборка chain (от конца к началу):
  terminalSink = collector.accumulator()     // добавить в список
  mapSink      = map(fn, terminalSink)       // применить fn, передать дальше
  filterSink   = filter(pred, mapSink)       // отфильтровать, передать дальше

Выполнение (tryAdvance по Spliterator):
  spliterator.tryAdvance(filterSink)
    → filterSink.accept(elem):  if (pred.test(elem)) mapSink.accept(elem)
    → mapSink.accept(elem):     terminalSink.accept(fn.apply(elem))
    → terminalSink.accept(r):   list.add(r)
```

Реализация `filter()` через `Sink.ChainedReference` (упрощённо из исходников JDK):
```java
@Override
ReferencePipeline<P_OUT, P_OUT> filter(Predicate<? super P_OUT> predicate) {
    return new StatelessOp<>(this, ...) {
        @Override
        Sink<P_OUT> opWrapSink(int flags, Sink<P_OUT> downstream) {
            return new Sink.ChainedReference<P_OUT, P_OUT>(downstream) {
                @Override
                public void accept(P_OUT u) {
                    if (predicate.test(u))
                        downstream.accept(u); // передаём следующему Sink в цепочке
                }
            };
        }
    };
}
```

### Запуск pipeline — evaluate()

```java
// Терминальная операция запускает:
AbstractPipeline.evaluate(terminalOp):
    if (isParallel())
        return terminalOp.evaluateParallel(this, sourceSpliterator(...));
    else
        return terminalOp.evaluateSequential(this, sourceSpliterator(...));

// Sequential:
//   PipelineHelper.wrapAndCopyInto(terminalSink, spliterator)
//   → wrapSink() — строит Sink chain
//   → copyInto(wrappedSink, spliterator) — гоняет tryAdvance() в цикле

// Parallel:
//   spliterator.trySplit() → ForkJoinTask для каждой части
//   → каждая задача строит свою Sink chain и выполняет
//   → combiner соединяет результаты
```

### Spliterator — разбивка для параллелизма

```java
interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);  // обработать один элемент
    void forEachRemaining(Consumer<? super T> action); // все оставшиеся
    Spliterator<T> trySplit();  // разделить пополам → для ForkJoin
    long estimateSize();        // для балансировки задач
    int characteristics();      // битовые флаги возможностей
}

// Characteristics (комбинируются битами):
ORDERED   0x00000010  // порядок определён → merge должен сохранять порядок
DISTINCT  0x00000001  // нет дубликатов → distinct() — no-op
SORTED    0x00000004  // отсортировано → sorted() — no-op
SIZED     0x00000040  // размер известен точно → limit/skip оптимизируются
SUBSIZED  0x00004000  // trySplit() возвращает SIZED части
IMMUTABLE 0x00000400  // источник не изменяется → нет modCount проверок
NONNULL   0x00000100  // нет null элементов → filter(x -> x != null) — no-op

// ArrayList.spliterator() — ORDERED | SIZED | SUBSIZED | IMMUTABLE (если List.of)
// HashSet.spliterator()   — DISTINCT | SIZED | SUBSIZED
// TreeSet.spliterator()   — DISTINCT | SORTED | ORDERED | SIZED
```

**trySplit стратегии** влияют на параллельную производительность:
```java
// ArrayList: ArraySpliterator.trySplit() — O(1) разбивка по индексу
//   [0..n] → [0..n/2] + [n/2..n]  — идеально для ForkJoin

// LinkedList: IteratorSpliterator.trySplit() — O(n/2) обход до середины
//   Поэтому parallelStream() на LinkedList неэффективен!

// IntStream.range(0, 1000).parallel() — RangeIntSpliterator, O(1) split
```

### PipelineHelper — мост между pipeline и terminal op

`PipelineHelper<P_OUT>` — абстрактный контракт, который `AbstractPipeline` реализует для terminal op:

```java
abstract class PipelineHelper<P_OUT> {
    abstract StreamShape getSourceShape();       // REFERENCE / INT_VALUE / LONG_VALUE / DOUBLE_VALUE
    abstract int getStreamAndOpFlags();          // combined flags всей цепочки
    abstract long exactOutputSizeIfKnown(Spliterator<?> spliterator);
    abstract <P_IN, S extends Sink<P_OUT>> S wrapAndCopyInto(S sink, Spliterator<P_IN> spliterator);
    abstract <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator);
}
// Terminal ops (Collectors) работают через PipelineHelper, не знают о конкретных промежуточных операциях
```

### Флаги и оптимизации

Stream **отслеживает флаги** цепочки и устраняет ненужные операции:

```java
// DISTINCT флаг: если источник Set (DISTINCT), distinct() — no-op
new HashSet<>(list).stream().distinct().count(); // distinct() пропускается компилятором pipeline

// SORTED флаг: TreeSet → stream().sorted() — no-op (уже отсортировано)
new TreeSet<>(list).stream().sorted().toList(); // sorted() пропускается!

// SIZED флаг: если размер известен — toArray() аллоцирует точный массив без resize
list.stream().filter(x -> x > 0).toArray(); // SIZED потерян после filter → внутренний буфер
list.stream().toArray();                     // SIZED сохранён → точная аллокация
```

---

## Промежуточные операции

```java
// Stateless:
stream.filter(Predicate)              // оставить подходящие
stream.map(Function)                  // трансформировать
stream.flatMap(Function<T, Stream<R>>)// развернуть в один stream
stream.peek(Consumer)                 // side effect (отладка)
stream.mapToInt/Long/Double(fn)       // → IntStream (без boxing)

// Stateful:
stream.sorted()                       // сортировка — буферизирует всё!
stream.sorted(Comparator)
stream.distinct()                     // уникальные — буферизирует всё!
stream.limit(n)                       // short-circuit (может остановить раньше)
stream.skip(n)

// Short-circuit: findFirst/findAny/anyMatch/allMatch — могут остановить поток раньше
// Проблема: sorted() + infinite stream → никогда не закончится!
Stream.iterate(1, n -> n + 1).sorted().findFirst(); // НИКОГДА не завершится
```

---

## Collectors

```java
// Базовые:
Collectors.toList()                              // ArrayList (не гарантирует modifiability)
Collectors.toUnmodifiableList()                  // Java 10+
Collectors.toSet()
Collectors.toMap(keyFn, valueFn)
Collectors.toMap(keyFn, valueFn, mergeFunction)  // при дублирующихся ключах

// Группировка:
Map<Department, List<Employee>> byDept =
    employees.stream().collect(Collectors.groupingBy(Employee::getDepartment));

// Двухуровневая группировка:
Map<Dept, Map<City, List<Emp>>> byDeptAndCity = stream.collect(
    Collectors.groupingBy(Emp::getDept, Collectors.groupingBy(Emp::getCity))
);

// Группировка с downstream collector:
Map<Dept, Long> countByDept = stream.collect(
    Collectors.groupingBy(Emp::getDept, Collectors.counting())
);
Map<Dept, Double> avgSalaryByDept = stream.collect(
    Collectors.groupingBy(Emp::getDept, Collectors.averagingDouble(Emp::getSalary))
);

// Разбивка по условию (partitioningBy):
Map<Boolean, List<Integer>> evenOdd = stream.collect(
    Collectors.partitioningBy(n -> n % 2 == 0)
);

// joining:
String result = stream.collect(Collectors.joining(", ", "[", "]"));

// toMap с merge (избегать дублей):
Map<String, Long> wordCount = words.stream().collect(
    Collectors.toMap(w -> w, w -> 1L, Long::sum)
);
```

**Collectors.teeing() (Java 12) — два коллектора за один проход:**
```java
record Stats(long count, double sum) {}

Stats stats = numbers.stream().collect(Collectors.teeing(
    Collectors.counting(),
    Collectors.summingDouble(Double::doubleValue),
    Stats::new
));

// Min и max за один проход:
record MinMax(Optional<Integer> min, Optional<Integer> max) {}
MinMax minMax = stream.collect(Collectors.teeing(
    Collectors.minBy(Integer::compareTo),
    Collectors.maxBy(Integer::compareTo),
    MinMax::new
));
```

---

## Parallel Streams

```java
list.parallelStream()   // через ForkJoinPool.commonPool()
stream.parallel()       // переключить существующий

// Рекомендованный размер пула:
// Runtime.getRuntime().availableProcessors() - 1 потоков в commonPool
```

**Когда parallelStream ВРЕДИТ:**

1. **Малые коллекции** (< ~10 000) — overhead fork/join > выигрыш
2. **I/O-bound операции** — блокируют CPU-потоки commonPool
3. **Stateful лямбды** — data race
4. **Упорядоченные операции** (`findFirst`, `sorted`) — нужна синхронизация
5. **LinkedList** — `trySplit()` O(n), неравномерное распределение

```java
// Хорошо: CPU-intensive, большой массив, без side effects
long primes = LongStream.rangeClosed(1, 10_000_000).parallel()
    .filter(n -> isPrime(n)).count();

// Плохо: data race
List<String> result = new ArrayList<>();
list.parallelStream().forEach(result::add); // RACE CONDITION!
// Правильно:
List<String> result = list.parallelStream().collect(Collectors.toList());

// Если порядок не важен — unordered() снимает ограничение:
list.parallelStream().unordered().filter(...).limit(5);
```

---

## Gatherers (Java 24, JEP 485)

Кастомные stateful **промежуточные** операции. `Collector` = терминальная операция, `Gatherer` = промежуточная.

```java
import java.util.stream.Gatherers;

// 1. windowFixed(n) — скользящие окна фиксированного размера:
List<List<Integer>> windows = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.windowFixed(3)).toList();
// [[1,2,3], [4,5]]

// 2. windowSliding(n) — скользящее окно с шагом 1:
Stream.of(1, 2, 3, 4, 5).gather(Gatherers.windowSliding(3)).toList();
// [[1,2,3], [2,3,4], [3,4,5]]

// 3. scan(init, fn) — running total (как reduce, но выдаёт промежуточные):
Stream.of(1, 2, 3, 4, 5).gather(Gatherers.scan(() -> 0, Integer::sum)).toList();
// [1, 3, 6, 10, 15]

// 4. mapConcurrent(n, fn) — параллельный map с ограниченным concurrency:
urls.stream().gather(Gatherers.mapConcurrent(10, url -> fetchUrl(url))).toList();
```

**Custom Gatherer:**
```java
// Группировать последовательные числа в run:
Gatherer<Integer, List<Integer>, List<Integer>> consecutiveRuns =
    Gatherer.ofSequential(
        ArrayList::new,                              // initializer: state
        (state, elem, downstream) -> {               // integrator
            if (state.isEmpty() || elem == state.getLast() + 1) {
                state.add(elem);
            } else {
                downstream.push(new ArrayList<>(state));
                state.clear();
                state.add(elem);
            }
            return true;
        },
        (state, downstream) -> {                     // finisher
            if (!state.isEmpty()) downstream.push(new ArrayList<>(state));
        }
    );

Stream.of(1, 2, 3, 7, 8, 9, 15).gather(consecutiveRuns).toList();
// [[1,2,3], [7,8,9], [15]]
```

---

## Функциональные интерфейсы

```java
// Основные из java.util.function:
Function<T, R>        T → R           andThen, compose
BiFunction<T, U, R>   (T, U) → R
UnaryOperator<T>      T → T           (Function<T,T>)
BinaryOperator<T>     (T, T) → T

Consumer<T>           T → void        andThen
BiConsumer<T, U>      (T, U) → void

Predicate<T>          T → boolean     and, or, negate
BiPredicate<T, U>     (T, U) → boolean

Supplier<T>           () → T

// Примитивные (без boxing!):
IntFunction<R>        int → R
IntUnaryOperator      int → int
IntPredicate          int → boolean
IntSupplier           () → int
IntConsumer           int → void
ToIntFunction<T>      T → int
```

---

## Method References — 4 типа

```java
// 1. Статический метод: Class::staticMethod
Function<String, Integer> f1 = Integer::parseInt;
// = s -> Integer.parseInt(s)

// 2. Unbound instance method: Class::instanceMethod
// Первый параметр = receiver объект
Function<String, Integer> f2 = String::length;
// = s -> s.length()

BiFunction<String, String, Boolean> f3 = String::startsWith;
// = (s, prefix) -> s.startsWith(prefix)

// 3. Bound instance method: instance::instanceMethod
// Receiver зафиксирован (capturing → объект лямбды создаётся каждый раз)
String hello = "Hello";
Predicate<String> f4 = hello::startsWith;
// = s -> hello.startsWith(s)

// 4. Constructor: Class::new
Supplier<ArrayList<String>> f5 = ArrayList::new;
// = () -> new ArrayList<String>()

Function<Integer, ArrayList<String>> f6 = ArrayList::new;
// = n -> new ArrayList<String>(n)
```

---

## Lambda internals

**Как компилируется лямбда:**
```java
Runnable r = () -> System.out.println("hello");
// Компилируется в:
// 1. private static void lambda$0() { System.out.println("hello"); }
// 2. invokedynamic → LambdaMetafactory.metafactory() (bootstrap, 1 раз)
//    → генерирует класс implements Runnable
//    → CallSite кэшируется → последующие вызовы прямые
```

**Capturing vs Non-capturing:**
```java
// Non-capturing — не захватывает переменные:
Runnable r1 = () -> System.out.println("hello");
// JVM может вернуть ОДИН singleton объект → нет аллокаций!

// Capturing — захватывает переменную:
String prefix = "Hello";
Function<String, String> f = name -> prefix + name;
// При каждом создании: новый объект с захваченным prefix
// (JIT может оптимизировать если prefix — compile-time constant)
```

**effectively final:**
```java
int counter = 0;
list.forEach(x -> counter++); // ОШИБКА КОМПИЛЯЦИИ

// Почему: лямбда захватывает значение (копию).
// Изменяемая переменная снаружи создаёт неопределённость.

// Обходной путь для счётчика:
AtomicInteger counter = new AtomicInteger(0);
list.forEach(x -> counter.incrementAndGet()); // OK

// Или reduce:
long count = list.stream().filter(predicate).count(); // лучше
```

---

## Вопросы на интервью

- Почему `sorted()` нельзя использовать с бесконечным Stream?
- В чём разница stateless и stateful промежуточных операций?
- Как лямбда компилируется? Почему не анонимный класс?
- Что такое capturing лямбда? Чем она медленнее non-capturing?
- Назови 4 типа method references. Чем Unbound отличается от Bound?
- Когда `parallelStream()` работает хуже `stream()`? (малые коллекции, I/O, порядок)
- Что такое `Collectors.teeing()`? Для чего он нужен?
- Что такое Gatherer? Чем отличается от Collector?

---

## Подводные камни

- **Stream после терминальной операции** — `IllegalStateException`. Создавай новый stream для каждой цепочки. Не сохраняй stream в поле.
- **`sorted()` с бесконечным stream** — `Stream.iterate(...).sorted()` → никогда не завершится (буферизирует всё). Добавляй `limit()` до `sorted()`.
- **`forEach` в parallel stream с non-thread-safe коллекцией** — data race в `ArrayList`. Всегда `collect(Collectors.toList())` вместо `forEach(result::add)`.
- **`toMap()` с дублирующимися ключами** — `IllegalStateException` без merge-функции. Используй `toMap(k, v, (a, b) -> b)` или `groupingBy`.
- **`Collectors.toList()` vs `Stream.toList()`** — `Stream.toList()` (Java 16) возвращает **unmodifiable** список. `Collectors.toList()` возвращает modifiable. Не взаимозаменяемы.
- **`flatMap` с null** — `flatMap(fn)` где fn возвращает null → `NullPointerException`. Возвращай `Stream.empty()` вместо null.
- **parallelStream + ForkJoinPool.commonPool()** — если приложение использует VirtualThread или CF, они тоже делят commonPool. Тяжёлый параллельный stream может блокировать асинхронные задачи. Для изоляции: `.collect(Collectors.toList())` в кастомном ForkJoinPool через `pool.submit(() -> stream.parallel()...)`.
