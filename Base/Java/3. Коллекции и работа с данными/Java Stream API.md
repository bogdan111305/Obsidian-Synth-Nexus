Java Stream API, представленная в Java 8, предоставляет функциональный подход к обработке коллекций данных в декларативном стиле через цепочки операций (pipelines). Эта статья глубоко погружает в механизмы работы Stream API, рассматривая архитектуру, ключевые компоненты, поток выполнения и аспекты производительности.
## Общая структура потока

Поток (stream) представляет собой последовательность операций, преобразующих источник данных в результат. Структура выглядит следующим образом:

```
Источник данных → 
    Поток (AbstractPipeline) → 
        [Промежуточные операции] → 
            Терминальная операция → 
                Результат
```

- **Источник данных**: Обычно коллекция (например, `List`, `Set`), массив или другой источник данных.
- **Поток (AbstractPipeline)**: Абстракция, представляющая цепочку операций.
- **Промежуточные операции**: Операции, такие как `filter`, `map` или `sorted`, которые преобразуют или фильтруют элементы, создавая новый поток.
- **Терминальная операция**: Операция, такая как `collect`, `reduce` или `forEach`, которая запускает выполнение цепочки и возвращает результат.
- **Результат**: Итоговый вывод, который может быть коллекцией, одиночным значением или побочным эффектом (например, вывод в консоль).

Каждая операция в цепочке выполняется **лениво** (lazily), то есть вычисления не происходят до вызова терминальной операции.
## Ключевые классы и интерфейсы

Stream API построена на наборе классов и интерфейсов, которые управляют жизненным циклом и выполнением цепочки операций. Ниже представлена обновленная таблица с дополнительными деталями:

|Компонент|Назначение|Основные характеристики|
|---|---|---|
|`Stream<T>`|Публичный интерфейс для взаимодействия с потоками.|Определяет методы для промежуточных (`filter`, `map`) и терминальных (`collect`, `forEach`) операций.|
|`AbstractPipeline`|Абстрактный базовый класс для всех этапов цепочки, управляющий связыванием и выполнением.|Хранит ссылки на предыдущий и следующий этапы, отслеживает флаги операций.|
|`StatelessOp`|Промежуточные операции без состояния (например, `map`, `filter`, `peek`).|Не зависят от предыдущих элементов, потокобезопасны, легко параллелизуются.|
|`StatefulOp`|Промежуточные операции с состоянием (например, `sorted`, `distinct`, `limit`).|Требуют отслеживания состояния, что влияет на производительность и параллелизм.|
|`Sink<T>`|Абстракция для обработки элементов в цепочке.|Передает элементы следующему этапу, реализует методы `begin`, `accept`, `end`.|
|`PipelineHelper`|Вспомогательный класс для управления выполнением, флагами и взаимодействием со `Spliterator`.|Скрывает низкоуровневые детали выполнения и оптимизации.|
|`Spliterator<T>`|Итератор с поддержкой разделения для параллельной обработки.|Предоставляет методы `tryAdvance`, `trySplit` для последовательной и параллельной итерации.|
|`TerminalOp<T,R>`|Интерфейс для терминальных операций, возвращающих результат или побочный эффект.|Запускает выполнение цепочки, поддерживает последовательное и параллельное выполнение.|
|`LambdaMetafactory`|Механизм времени выполнения для генерации оптимизированных лямбда-выражений.|Использует `invokedynamic` для создания эффективных реализаций функциональных интерфейсов.|
|`ReferencePipeline`|Конкретная реализация `AbstractPipeline` для потоков объектов.|Работает с потоками ссылочных типов (например, `Stream<String>`).|
|`StreamShape`|Перечисление, определяющее тип потока (REFERENCE, INT, LONG, DOUBLE).|Обеспечивает типобезопасность и оптимизацию для примитивных потоков.|
### Дополнительные замечания

- **Потоки примитивов**: Специализированные потоки (`IntStream`, `LongStream`, `DoubleStream`) избегают накладных расходов на упаковку/распаковку примитивных типов, используя `IntPipeline`, `LongPipeline` и т.д.
- **Параллелизм**: Компоненты, такие как `Spliterator` и `AbstractPipeline`, разработаны для поддержки параллельного выполнения с минимальными накладными расходами.
- **LambdaMetafactory**: Повышает производительность за счет генерации оптимизированного байт-кода для лямбда-выражений, избегая накладных расходов анонимных классов.
## Этапы работы Stream API

### 1. Создание источника потока

Поток начинается с источника данных, такого как коллекция, массив или генератор. Источник оборачивается в `Spliterator`, предоставляющий интерфейс для доступа к элементам.

```java
List<String> list = List.of("яблоко", "банан", "вишня");
Stream<String> stream = list.stream();
```

**Внутренние механизмы**:

- Метод `stream()` создает экземпляр `ReferencePipeline.Head`, подкласса `AbstractPipeline`.
- `ReferencePipeline.Head` инициализирует:
    - `Spliterator` из источника (например, `ArrayListSpliterator` для `List`).
    - Ссылку на себя как на `sourceStage` (корень цепочки).
    - Флаги, описывающие свойства источника (например, `ORDERED`, `SIZED`).

**Упрощенное представление кода**:

```java
public class ReferencePipeline {
    static class Head<T, S extends BaseStream<T, S>> 
	    extends ReferencePipeline<T, T> {
        Head(Spliterator<T> source, int sourceFlags) {
            super(source, sourceFlags);
            this.sourceStage = this;
        }
    }
}
```

**Ключевые моменты**:

- `Spliterator` является точкой входа для доступа к элементам источника.
- Флаги источника (например, `ORDERED`, `DISTINCT`) влияют на оптимизации в цепочке.
- Узел `Head` не выполняет операций, а служит началом цепочки.
### 2. Добавление промежуточных операций

Промежуточные операции, такие как `filter`, `map` или `sorted`, преобразуют или фильтруют поток, создавая новый экземпляр `Stream`. Эти операции ленивы и не обрабатывают элементы до вызова терминальной операции.

```java
Stream<String> filtered = stream
    .filter(s -> s.length() > 4)
    .map(String::toUpperCase);
```

**Внутренние механизмы**:

- Каждая промежуточная операция создает новый узел в цепочке — либо `StatelessOp` (например, `filter`, `map`), либо `StatefulOp` (например, `sorted`, `distinct`).
- Операция представлена как `Sink`, который оборачивает следующий (downstream) `Sink`, формируя цепочку.

**Пример: Операция `filter`**:

```java
public final Stream<T> filter(Predicate<? super T> predicate) {
    return new StatelessOp<T, T>(
	    this, StreamShape.REFERENCE, StreamOpFlag.NOT_SIZED
    ) {
        @Override
        Sink<T> opWrapSink(int flags, Sink<T> downstream) {
            return new Sink.ChainedReference<T, T>(downstream) {
                @Override
                public void accept(T t) {
                    if (predicate.test(t)) {
                        downstream.accept(t);
                    }
                }
            };
        }
    };
}
```

- **Построение цепочки**: Каждая операция добавляет новый узел `AbstractPipeline`, ссылающийся на предыдущий узел, формируя цепочку:

```
Head → filterOp → mapOp
```

- **Цепочка Sink**: Метод `opWrapSink` оборачивает следующий `Sink`, создавая цепочку операций:

```
mapSink(filterSink(terminalSink))
```

**Ключевые моменты**:

- **Без состояния vs. Состояние**: Операции без состояния (`filter`, `map`) обрабатывают элементы независимо, а операции с состоянием (`sorted`, `distinct`) требуют знания всего потока, что влияет на производительность.
- **Ленивое выполнение**: Промежуточные операции только определяют цепочку; вычисления не выполняются до вызова терминальной операции.
### 3. Терминальная операция

Терминальная операция запускает выполнение цепочки, производя результат или побочный эффект.

```java
List<String> result = stream.collect(Collectors.toList());
```

**Внутренние механизмы**:

- Терминальная операция вызывает метод `AbstractPipeline.evaluate`, который:
    1. Формирует цепочку `Sink`, вызывая `opWrapSink` для каждой операции в обратном порядке.
    2. Получает `Spliterator` из источника.
    3. Итерирует по элементам, пропуская их через цепочку `Sink`.

**Поток выполнения**:

```java
public <R> R evaluate(TerminalOp<T, R> terminalOp) {
    return isParallel()
        ? terminalOp.evaluateParallel(
	        this, sourceSpliterator(terminalOp.getOpFlags())
        )
        : terminalOp.evaluateSequential(
	        this, sourceSpliterator(terminalOp.getOpFlags())
        );
}
```

**Последовательное выполнение**:

- Элементы обрабатываются по одному через цепочку `Sink` с помощью `Spliterator.tryAdvance`.

```java
Spliterator<T> spliterator = sourceSpliterator();
Sink<T> sinkChain = mapOp.opWrapSink(0, filterOp.opWrapSink(0, terminalSink));
while (spliterator.tryAdvance(sinkChain::accept)) {
    // Обработка каждого элемента через цепочку Sink
}
```

**Параллельное выполнение**:

- `Spliterator` разделяется на под-`Spliterator`-ы с помощью `trySplit`, и каждый под-поток обрабатывается параллельно.

**Ключевые моменты**:

- Цепочка `Sink` обеспечивает последовательное прохождение каждого элемента через все операции.
- Параллельное выполнение требует разделяемого источника и потокобезопасных операций.
- Тип результата определяется терминальной операцией (например, `List` для `collect`, `void` для `forEach`).
## Подробный разбор компонентов

### `AbstractPipeline`

- **Роль**: Управляет структурой и выполнением цепочки.
- **Основные функции**:
    - Связывает этапы цепочки.
    - Отслеживает флаги операций (например, `ORDERED`, `SIZED`).
    - Выполняет цепочку через метод `evaluate`.
- **Структура**:
    - Хранит ссылки `previousStage` и `nextStage`.
    - Содержит исходный `Spliterator` и флаги.
### `Sink<T>`

- **Роль**: Обрабатывает элементы на каждом этапе цепочки.
- **Методы**:
```java
    void begin(long size); // Инициализация обработки
    void accept(T t);     // Обработка элемента
    void end();           // Завершение обработки
    ```
- **Пример**: Для `filter` метод `accept` проверяет предикат и передает подходящие элементы следующему `Sink`.
- **Цепочка**: Каждый `Sink` оборачивает следующий, формируя цепочку обработки.
### `Spliterator<T>`

- **Роль**: Предоставляет интерфейс, подобный итератору, для доступа к элементам источника с поддержкой разделения для параллельной обработки.
- **Основные методы**:
    
    ```java
    // Обработка следующего элемента
    boolean tryAdvance(Consumer<? super T> action);
    // Разделение для параллельной обработки
    Spliterator<T> trySplit();
    // Оценка оставшихся элементов
    long estimateSize();                           
    // Свойства источника (например, ORDERED)
    int characteristics();
    ```
    
- **Пример**: `ArrayListSpliterator`:
    
    ```java
    public boolean tryAdvance(Consumer<? super T> action) {
        if (index < array.length) {
            action.accept(array[index++]);
            return true;
        }
        return false;
    }
    ```
    
- **Характеристики**: Флаги, такие как `ORDERED`, `DISTINCT`, `SIZED`, помогают оптимизировать выполнение.
### `PipelineHelper`

- **Роль**: Управляет низкоуровневыми деталями выполнения, такими как вычисление флагов и построение цепочки `Sink`.
- **Основные функции**:
    - Вычисляет комбинированные флаги для цепочки.
    - Помогает в создании цепочки `Sink`.
    - Управляет буферизацией для операций с состоянием.
### `TerminalOp<T,R>`

- **Роль**: Выполняет цепочку для получения результата.
- **Методы**:
    - `evaluateSequential`: Выполняет цепочку последовательно.
    - `evaluateParallel`: Разделяет цепочку для параллельного выполнения.
- **Пример**: `Collectors.toList` создает `TerminalOp`, который собирает элементы в `List`.
### `LambdaMetafactory` и `invokedynamic`

- **Роль**: Оптимизирует лямбда-выражения, используемые в операциях, таких как `filter` и `map`.
- **Механизм**:
    - Лямбда-выражения компилируются в вызовы `invokedynamic`.
    - `LambdaMetafactory.metafactory` генерирует оптимизированные классы во время выполнения.
- **Преимущества**:
    - Быстрее, чем анонимные классы.
    - Кэширует сгенерированные классы для повторного использования.
- **Пример**: `s -> s.length() > 4` становится реализацией `Predicate`.
## Путь элемента через цепочку

Путь элемента через поток:

```
Источник данных → 
	Spliterator → 
		Head → 
			Sink фильтрации → 
				Sink преобразования → 
					Терминальный Sink
```

1. **Источник данных**: Предоставляет элементы через `Spliterator`.
2. **Spliterator**: Доставляет элементы с помощью `tryAdvance`.
3. **Head**: Передает элементы в `Sink` первой операции.
4. **Sink фильтрации**: Применяет предикат, пропуская подходящие элементы.
5. **Sink преобразования**: Преобразует элементы и передает их дальше.
6. **Терминальный Sink**: Собирает или обрабатывает элементы для получения результата.

**Пример**:

```java
List<String> list = List.of("яблоко", "банан", "вишня");
List<String> result = list.stream()
    .filter(s -> s.length() > 4)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// Результат: ["БАНАН", "ВИШНЯ"]
```

**Поток**:

- `Spliterator` доставляет "яблоко", "банан", "вишня".
- `filter` проверяет `length > 4`, отбрасывает "яблоко".
- `map` преобразует "банан" в "БАНАН", "вишня" в "ВИШНЯ".
- `collect` добавляет элементы в `List`.
## Оценка сложности Big-O

|Операция|Сложность|Примечания|
|---|---|---|
|`filter`, `map`|O(n)|Обрабатывает каждый элемент один раз; без состояния.|
|`sorted`|O(n log n)|Требует сортировки всего потока; с состоянием.|
|`distinct`|O(n log n)|Использует хэш-множество или сортировку для дедупликации; с состоянием.|
|`limit`, `skip`|O(n)|Может обрабатывать меньше элементов, но итерация последовательная.|
|`collect`|O(n)|Зависит от коллектора; `toList` — O(n), `groupingBy` может быть сложнее.|
|`reduce`|O(n)|Обрабатывает каждый элемент один раз; может включать накладные расходы.|
**Параллельная сложность**:

- Операции без состояния (`filter`, `map`) хорошо масштабируются, приближаясь к O(n/p) при p процессорах.
- Операции с состоянием (`sorted`, `distinct`) имеют ограниченную масштабируемость из-за накладных расходов на синхронизацию.
## Расширенная структура цепочки

```
ReferencePipeline.Head (источник)
└─ StatelessOp (filter, map, peek и др.)
   └─ StatefulOp (sorted, distinct, limit и др.)
      └─ TerminalOp (collect, reduce, forEach и др.)
```

- **Связи узлов**: Каждый узел ссылается на предыдущий и следующий.
- **Оборачивание Sink**: Метод `opWrapSink` каждого узла оборачивает следующий `Sink`.
- **Выполнение**: Терминальная операция запускает итерацию через `Spliterator`.
## Детали производительности и оптимизации

### Ленивое выполнение

- Промежуточные операции формируют цепочку без обработки элементов.
- Терминальные операции запускают выполнение, минимизируя затраты памяти и вычислений.
### Слияние операций

- JVM объединяет операции без состояния (например, несколько `map`) в один проход по данным, снижая накладные расходы на итерацию.
- Пример:

```java
stream.map(s -> s.toUpperCase()).map(s -> s + "!");
```

Оба `map` объединяются в один цикл.
### Короткое замыкание

- Операции, такие как `limit`, `findFirst` и `anyMatch`, прекращают обработку после выполнения условия.
- Пример:

```java
stream.filter(s -> s.length() > 4).findFirst();
```

Останавливается после нахождения первого подходящего элемента.
### Параллелизм

- `parallelStream()` разделяет `Spliterator` на под-`Spliterator`-ы, обрабатываемые в пуле потоков (обычно `ForkJoinPool`).
- **Лучшие сценарии**:
    - Большие наборы данных с операциями без состояния.
    - Задачи, связанные с вычислениями (например, `map` с тяжелыми операциями).
- **Проблемы**:
    - Операции с состоянием (`sorted`, `distinct`) создают накладные расходы на синхронизацию.
    - Маленькие наборы данных могут иметь большие накладные расходы на управление потоками.
### Характеристики источника

- Характеристики `Spliterator` (например, `SIZED`, `ORDERED`) обеспечивают оптимизации:
    - `SIZED`: Позволяет заранее выделить память для контейнеров результата.
    - `ORDERED`: Гарантирует сохранение порядка элементов.
    - `DISTINCT`: Пропускает дедупликацию, если источник уже уникален.
## Проблемные случаи и подводные камни

### Проблемы операций с состоянием

- Операции, такие как `sorted` и `distinct`, буферизируют весь поток, увеличивая использование памяти.
- Пример:

```java
stream.distinct().collect(Collectors.toList());
```

Требует O(n) памяти для дедупликации.
### Проблемы параллелизма

- Параллельные потоки не всегда улучшают производительность для маленьких наборов данных или задач, связанных с вводом-выводом.
- Конфликты потоков могут ухудшить производительность для операций с состоянием.
### Побочные эффекты

- Потоки должны быть без побочных эффектов для предсказуемого поведения.
- Плохая практика:

```java
List<String> sideEffectList = new ArrayList<>();
stream.forEach(s -> sideEffectList.add(s));
```

Используйте `collect`:

```java
List<String> result = stream.collect(Collectors.toList());
```
### Ненавязчивость

- Источник не должен изменяться во время выполнения потока.
- Пример ошибки:

```java
List<String> list = new ArrayList<>(List.of("а", "б"));
list.stream().filter(s -> s.equals("а")).forEach(s -> list.remove(s));
```

Вызывает `ConcurrentModificationException`.
## Расширенный пример

```java
List<String> result = List.of("яблоко", "банан", "вишня", "финик")
    .stream()
    .filter(s -> s.length() > 4)
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());
// Результат: ["БАНАН", "ВИШНЯ"]
```

**Разбор выполнения**:

1. **Источник**: `ArrayListSpliterator` доставляет элементы.
2. **Filter**: Проверяет `length > 4`, отбрасывает "финик".
3. **Map**: Преобразует в верхний регистр ("банан" → "БАНАН").
4. **Sorted**: Буферизирует элементы, сортирует их (O(n log n)).
5. **Collect**: Собирает элементы в `List`.

**Параллельная версия**:

```java
List<String> result = List.of("яблоко", "банан", "вишня", "финик")
    .parallelStream()
    .filter(s -> s.length() > 4)
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());
```

- `Spliterator` разделяет источник на под-потоки.
- `filter` и `map` выполняются параллельно.
- `sorted` объединяет под-потоки, вызывая накладные расходы на синхронизацию.
## Лучшие практики

1. **Используйте операции без состояния**: Предпочитайте `filter` и `map` для лучшего параллелизма и производительности.
2. **Минимизируйте операции с состоянием**: Избегайте `sorted` и `distinct` для больших наборов данных, если это не необходимо.
3. **Выбирайте подходящие коллекторы**: Используйте `toList`, `joining` или `groupingBy` в зависимости от задачи.
4. **Тестируйте параллелизм**: Проводите тесты производительности для параллельных потоков.
5. **Избегайте побочных эффектов**: Используйте коллекторы вместо изменения внешнего состояния.
6. **Понимайте характеристики источника**: Используйте свойства источника для оптимизации цепочки.

## Современные возможности Java 21+

### Виртуальные потоки и Stream API

Java 21 представил виртуальные потоки (Project Loom), которые интегрируются со Stream API:

```java
// Создание виртуальных потоков для обработки данных
List<String> data = List.of("a", "b", "c");
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<List<String>> future = CompletableFuture.supplyAsync(() -> {
    return data.stream()
        .map(String::toUpperCase)
        .collect(Collectors.toList());
}, executor);
```

### Pattern Matching в Stream API

Java 21 улучшил pattern matching, что полезно в Stream API:

```java
List<Object> mixed = List.of("string", 42, 3.14, null);

List<String> strings = mixed.stream()
    .filter(obj -> obj instanceof String s && s.length() > 3)
    .map(String.class::cast)
    .collect(Collectors.toList());
```

### Structured Concurrency

Новые возможности для управления параллельными операциями:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<List<String>> future1 = scope.fork(() -> 
        processDataStream(data1.stream()));
    Future<List<String>> future2 = scope.fork(() -> 
        processDataStream(data2.stream()));
    
    scope.join();
    scope.throwIfFailed();
    
    List<String> result = Stream.concat(
        future1.resultNow().stream(),
        future2.resultNow().stream()
    ).collect(Collectors.toList());
}
```

## Расширенные примеры использования

### Обработка больших файлов

```java
public class LargeFileProcessor {
    
    public static void processLargeFile(String filename) {
        try (Stream<String> lines = Files.lines(Paths.get(filename))) {
            Map<String, Long> wordCount = lines
                .flatMap(line -> Arrays.stream(line.split("\\s+")))
                .filter(word -> !word.isEmpty())
                .map(String::toLowerCase)
                .collect(Collectors.groupingBy(
                    word -> word,
                    Collectors.counting()
                ));
            
            // Найти топ-10 слов
            wordCount.entrySet().stream()
                .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
                .limit(10)
                .forEach(entry -> 
                    System.out.println(entry.getKey() + ": " + entry.getValue()));
        } catch (IOException e) {
            throw new RuntimeException("Error processing file", e);
        }
    }
}
```

### Параллельная обработка с мониторингом

```java
public class ParallelStreamMonitor {
    
    public static <T> List<T> processWithMonitoring(
            Stream<T> stream, 
            Function<T, T> processor,
            Consumer<String> logger) {
        
        AtomicLong processed = new AtomicLong(0);
        AtomicLong errors = new AtomicLong(0);
        
        return stream
            .parallel()
            .peek(item -> {
                long count = processed.incrementAndGet();
                if (count % 1000 == 0) {
                    logger.accept("Processed: " + count + " items");
                }
            })
            .map(item -> {
                try {
                    return processor.apply(item);
                } catch (Exception e) {
                    errors.incrementAndGet();
                    logger.accept("Error processing item: " + e.getMessage());
                    return null;
                }
            })
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

### Кастомные коллекторы

```java
public class CustomCollectors {
    
    // Коллектор для вычисления статистики
    public static <T extends Number> Collector<T, ?, DoubleSummaryStatistics> 
            toStatistics() {
        return Collector.of(
            DoubleSummaryStatistics::new,
            (stats, value) -> stats.accept(value.doubleValue()),
            (left, right) -> {
                left.combine(right);
                return left;
            },
            Collector.Characteristics.UNORDERED
        );
    }
    
    // Коллектор для группировки с ограничением
    public static <T, K> Collector<T, ?, Map<K, List<T>>> 
            groupingByWithLimit(Function<T, K> classifier, int limit) {
        return Collector.of(
            HashMap::new,
            (map, item) -> {
                K key = classifier.apply(item);
                map.computeIfAbsent(key, k -> new ArrayList<>())
                   .add(item);
                if (map.get(key).size() > limit) {
                    map.get(key).remove(0); // Удаляем старый элемент
                }
            },
            (left, right) -> {
                right.forEach((key, value) -> 
                    left.merge(key, value, (oldList, newList) -> {
                        oldList.addAll(newList);
                        if (oldList.size() > limit) {
                            return oldList.subList(oldList.size() - limit, oldList.size());
                        }
                        return oldList;
                    }));
                return left;
            }
        );
    }
}
```

### Интеграция с внешними системами

```java
public class StreamIntegration {
    
    // Потоковая обработка с базой данных
    public static void processDatabaseRecords(DataSource dataSource) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users");
             ResultSet rs = stmt.executeQuery()) {
            
            StreamSupport.stream(
                Spliterators.spliteratorUnknownSize(
                    new Iterator<User>() {
                        @Override
                        public boolean hasNext() {
                            try {
                                return rs.next();
                            } catch (SQLException e) {
                                throw new RuntimeException(e);
                            }
                        }
                        
                        @Override
                        public User next() {
                            try {
                                return new User(rs.getString("name"), rs.getInt("age"));
                            } catch (SQLException e) {
                                throw new RuntimeException(e);
                            }
                        }
                    },
                    Spliterator.ORDERED
                ),
                false
            )
            .filter(user -> user.getAge() > 18)
            .map(User::getName)
            .forEach(System.out::println);
            
        } catch (SQLException e) {
            throw new RuntimeException("Database error", e);
        }
    }
    
    // Потоковая обработка с HTTP API
    public static CompletableFuture<List<String>> processApiData(List<String> urls) {
        return urls.stream()
            .map(url -> CompletableFuture.supplyAsync(() -> fetchData(url)))
            .collect(Collectors.collectingAndThen(
                Collectors.toList(),
                futures -> CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                    .thenApply(v -> futures.stream()
                        .map(CompletableFuture::join)
                        .collect(Collectors.toList()))
            ));
    }
    
    private static String fetchData(String url) {
        // Реализация HTTP запроса
        return "data from " + url;
    }
}
```

## Вопросы для собеседования

### Базовые вопросы

1. **Что такое Stream API и чем он отличается от обычных коллекций?**
   - Stream API предоставляет функциональный подход к обработке данных
   - Ленивое выполнение операций
   - Поддержка параллельной обработки
   - Неизменяемость (каждая операция возвращает новый Stream)

2. **Объясните разницу между промежуточными и терминальными операциями**
   - Промежуточные: `filter`, `map`, `sorted` - возвращают Stream
   - Терминальные: `collect`, `forEach`, `reduce` - запускают выполнение

3. **Что такое ленивое выполнение в Stream API?**
   - Операции не выполняются до вызова терминальной операции
   - Позволяет оптимизировать цепочки операций

### Продвинутые вопросы

4. **Как работает параллелизм в Stream API?**
   - Использует `ForkJoinPool.commonPool()`
   - `Spliterator` для разделения данных
   - Требует потокобезопасных операций

5. **Объясните внутреннюю архитектуру Stream API**
   - `AbstractPipeline` - базовая цепочка операций
   - `Sink` - обработка элементов
   - `Spliterator` - итерация и разделение

6. **Какие проблемы могут возникнуть при использовании параллельных потоков?**
   - Гонки данных при изменении состояния
   - Накладные расходы на синхронизацию
   - Непредсказуемый порядок выполнения

### Практические вопросы

7. **Напишите код для нахождения дубликатов в списке**
```java
List<String> duplicates = list.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .filter(entry -> entry.getValue() > 1)
    .map(Map.Entry::getKey)
    .collect(Collectors.toList());
```

8. **Как оптимизировать производительность Stream API?**
   - Используйте примитивные потоки (`IntStream`, `LongStream`)
   - Избегайте операций с состоянием для больших данных
   - Выбирайте подходящие коллекторы
   - Используйте `unordered()` когда порядок не важен

9. **Реализуйте кастомный коллектор**
```java
public static <T> Collector<T, ?, List<T>> toReversedList() {
    return Collector.of(
        ArrayList::new,
        (list, item) -> list.add(0, item),
        (left, right) -> {
            left.addAll(0, right);
            return left;
        }
    );
}
```