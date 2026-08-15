# Kafka Streams: потоковая обработка данных

> [!QUOTE] Kafka Streams — Java-библиотека для потоковой обработки данных из Kafka. Не требует отдельного кластера (запускается как обычное Java-приложение). Обеспечивает exactly-once семантику, stateful обработку и горизонтальное масштабирование.

## Оглавление
1. [Введение в Kafka Streams](#введение)
2. [Архитектура](#архитектура)
3. [Основные концепции](#концепции)
4. [DSL API](#dsl)
5. [Processor API](#processor)
6. [Агрегация и оконная обработка](#агрегация)
7. [Состояние и хранилища](#состояние)
8. [Мониторинг и отладка](#мониторинг)
9. [Лучшие практики](#практики)
10. [Вопросы для собеседования](#вопросы)

## Введение в Kafka Streams

**Kafka Streams** — клиентская библиотека для построения потоковых приложений, где и источник, и результат — топики Kafka.

### Ключевые свойства:
- **Простота развёртывания**: обычное Java-приложение, не нужен отдельный кластер
- **Горизонтальная масштабируемость**: запустить несколько инстансов — нагрузка распределится автоматически
- **Отказоустойчивость**: state восстанавливается из changelog-топиков Kafka
- **Exactly-once семантика**: настраивается через `processing.guarantee`
- **Интеграция**: Schema Registry, Kafka Connect

## Архитектура

### Компоненты

#### 1. Streams Application

```java
public class WordCountApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "word-count-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        Topology topology = builder.build();

        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();

        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

#### 2. Topology (граф обработки)
- **Source**: входные топики
- **Processors**: операторы преобразования (filter, map, aggregate)
- **Sink**: выходные топики

#### 3. Tasks и Threads
- **Tasks**: единицы параллелизма. Количество tasks = количество партиций входного топика
- **Threads**: выполняют tasks. Настраивается через `num.stream.threads`

> [!INFO] Масштабирование Kafka Streams: запустите несколько инстансов приложения с одинаковым `application.id`. Kafka Streams автоматически распределит партиции (и соответствующие tasks) между инстансами.

## Основные концепции

### 1. KStream и KTable

```java
// KStream — поток событий. Каждое сообщение — независимое событие.
KStream<String, String> textLines = builder.stream("text-lines");

// KTable — таблица состояний. Новая запись с тем же ключом обновляет предыдущую.
KTable<String, Long> wordCounts = builder.table("word-counts");

// GlobalKTable — реплицируется на все инстансы (используется для справочных данных)
GlobalKTable<String, String> userProfiles = builder.globalTable("user-profiles");
```

### 2. Serdes (Serializer/Deserializer)

```java
// Стандартные Serdes
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

// Кастомные Serdes
Serde<User> userSerde = Serdes.serdeFrom(new UserSerializer(), new UserDeserializer());
```

> [!WARNING] KStream vs KTable — важное различие:
> - KStream: `key=user-1, value=100` затем `key=user-1, value=200` — это два события. Оба будут обработаны.
> - KTable: та же последовательность означает, что значение для `user-1` обновилось с 100 до 200. Агрегация использует только последнее значение.

## DSL API

### 1. Базовые операции

```java
public class BasicStreamsApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "basic-streams-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        builder.stream("input-topic")
            .filter((key, value) -> value != null && !value.isEmpty())  // фильтрация
            .mapValues(value -> value.toUpperCase())                      // преобразование
            .to("output-topic");                                          // запись

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();

        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### 2. Агрегация

```java
public class AggregationApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "aggregation-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        KGroupedStream<String, String> groupedStream = builder
            .stream("input-topic")
            .groupBy((key, value) -> key);

        // Подсчёт
        KTable<String, Long> countTable = groupedStream.count();

        // Суммирование
        KTable<String, Integer> sumTable = groupedStream
            .aggregate(
                () -> 0,
                (key, value, aggregate) -> aggregate + Integer.parseInt(value),
                Materialized.with(Serdes.String(), Serdes.Integer())
            );

        countTable.toStream().to("count-topic");
        sumTable.toStream().to("sum-topic");

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### 3. Оконная обработка

```java
public class WindowedAggregationApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "windowed-aggregation-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        builder.stream("input-topic")
            .groupBy((key, value) -> key)
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
            .count()
            .toStream()
            .map((key, value) -> KeyValue.pair(key.key(), value))
            .to("windowed-count-topic");

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### 4. Join операции

```java
public class JoinApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "join-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        KStream<String, String> ordersStream = builder.stream("orders");
        KTable<String, String> usersTable = builder.table("users");

        // Left join: обогащение потока заказов данными пользователей
        ordersStream
            .leftJoin(usersTable,
                (orderValue, userValue) -> orderValue + " | " + userValue)
            .to("enriched-orders");

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

> [!WARNING] Join KStream с KTable требует, чтобы оба использовали одинаковые ключи и одинаковое число партиций. Если топики с разным числом партиций — join не сработает. Используйте `repartition()` для выравнивания.

## Processor API

### 1. Кастомный процессор

```java
public class CustomProcessor implements Processor<String, String> {
    private ProcessorContext context;
    private KeyValueStore<String, Integer> kvStore;

    @Override
    public void init(ProcessorContext context) {
        this.context = context;
        this.kvStore = context.getStateStore("word-count-store");
    }

    @Override
    public void process(String key, String value) {
        String[] words = value.toLowerCase().split("\\W+");

        for (String word : words) {
            if (!word.isEmpty()) {
                Integer count = kvStore.get(word);
                count = (count == null) ? 1 : count + 1;
                kvStore.put(word, count);
                context.forward(word, count);
            }
        }
    }

    @Override
    public void close() {}
}

// Использование в топологии
public class CustomProcessorApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "custom-processor-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        Topology topology = new Topology();
        topology.addSource("source", "input-topic")
                .addProcessor("processor", CustomProcessor::new)
                .addSink("sink", "output-topic", "processor");

        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### 2. State Store

```java
// Создание State Store
StoreBuilder<KeyValueStore<String, Integer>> storeBuilder =
    Stores.keyValueStoreBuilder(
        Stores.persistentKeyValueStore("word-count-store"),
        Serdes.String(),
        Serdes.Integer()
    );

topology.addStateStore(storeBuilder);
topology.addSource("source", "input-topic")
        .addProcessor("processor", CustomProcessor::new, "word-count-store")
        .addSink("sink", "output-topic", "processor");
```

## Агрегация и оконная обработка

### 1. Типы окон

```java
StreamsBuilder builder = new StreamsBuilder();
KGroupedStream<String, String> groupedStream = builder
    .stream("input-topic")
    .groupBy((key, value) -> key);

// Tumbling Window — фиксированное неперекрывающееся окно
TimeWindows tumblingWindow = TimeWindows.of(Duration.ofMinutes(5));

// Hopping Window — скользящее окно с шагом
HoppingWindows hoppingWindow = HoppingWindows.of(
    Duration.ofMinutes(5), Duration.ofMinutes(1));

// Session Window — окно, закрываемое по неактивности
SessionWindows sessionWindow = SessionWindows.with(Duration.ofMinutes(5));

groupedStream.windowedBy(tumblingWindow).count()
    .toStream().to("tumbling-count-topic");

groupedStream.windowedBy(hoppingWindow).count()
    .toStream().to("hopping-count-topic");

groupedStream.windowedBy(sessionWindow).count()
    .toStream().to("session-count-topic");
```

### 2. Кастомная агрегация

```java
KTable<String, CustomAggregate> customAggregate = groupedStream
    .aggregate(
        CustomAggregate::new,
        (key, value, aggregate) -> aggregate.add(value),
        Materialized.with(Serdes.String(), new CustomAggregateSerde())
    );

class CustomAggregate {
    private int count;
    private double sum;
    private double min = Double.MAX_VALUE;
    private double max = Double.MIN_VALUE;

    public CustomAggregate add(String value) {
        double num = Double.parseDouble(value);
        count++;
        sum += num;
        min = Math.min(min, num);
        max = Math.max(max, num);
        return this;
    }

    public double getAverage() { return count > 0 ? sum / count : 0.0; }
    public int getCount() { return count; }
    public double getSum() { return sum; }
    public double getMin() { return min; }
    public double getMax() { return max; }
}
```

## Состояние и хранилища

### 1. Типы State Stores

```java
// Key-Value Store
StoreBuilder<KeyValueStore<String, Integer>> kvStore =
    Stores.keyValueStoreBuilder(
        Stores.persistentKeyValueStore("kv-store"),
        Serdes.String(), Serdes.Integer()
    );

// Window Store
StoreBuilder<WindowStore<String, Integer>> windowStore =
    Stores.windowStoreBuilder(
        Stores.persistentWindowStore("window-store",
            Duration.ofMinutes(5), Duration.ofMinutes(1), false),
        Serdes.String(), Serdes.Integer()
    );

// Session Store
StoreBuilder<SessionStore<String, Integer>> sessionStore =
    Stores.sessionStoreBuilder(
        Stores.persistentSessionStore("session-store", Duration.ofMinutes(5)),
        Serdes.String(), Serdes.Integer()
    );
```

### 2. Интерактивные запросы (Interactive Queries)

```java
public class InteractiveQueriesApp {
    private KafkaStreams streams;

    public void start() {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "interactive-queries-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        StreamsBuilder builder = new StreamsBuilder();

        builder.stream("input-topic")
            .groupBy((key, value) -> value)
            .count(Materialized.as("word-count-store"));

        streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }

    public Integer getWordCount(String word) {
        ReadOnlyKeyValueStore<String, Integer> store =
            streams.store("word-count-store", QueryableStoreTypes.keyValueStore());
        return store.get(word);
    }

    public void close() {
        if (streams != null) streams.close();
    }
}
```

## Мониторинг и отладка

### 1. Метрики

```java
Properties props = new Properties();
// ...
props.put(StreamsConfig.METRICS_RECORDING_LEVEL_CONFIG, "INFO");
props.put(StreamsConfig.METRICS_NUM_SAMPLES_CONFIG, 2);
props.put(StreamsConfig.METRICS_SAMPLE_WINDOW_MS_CONFIG, 30000);

KafkaStreams streams = new KafkaStreams(topology, props);

streams.setUncaughtExceptionHandler((thread, throwable) -> {
    System.err.println("Uncaught exception in thread " + thread + ": " + throwable);
});
```

### 2. Логирование

```properties
# log4j.properties
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[%d] %p %m (%c)%n

log4j.logger.org.apache.kafka.streams=INFO
log4j.logger.org.apache.kafka.clients=INFO
```

### 3. Отладка топологии

```java
StreamsBuilder builder = new StreamsBuilder();

builder.stream("input-topic")
    .filter((key, value) -> value != null)
    .mapValues(value -> value.toUpperCase())
    .to("output-topic");

Topology topology = builder.build();

// Вывод описания топологии для отладки
System.out.println(topology.describe());
```

## Лучшие практики

### 1. Конфигурация производительности

```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "optimized-streams-app");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

// Производительность
props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);
props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024);

// Exactly-once
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);

// Отказоустойчивость
props.put(StreamsConfig.REPLICATION_FACTOR_CONFIG, 3);
props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);
```

> [!WARNING] `EXACTLY_ONCE_V2` снижает пропускную способность по сравнению с `AT_LEAST_ONCE`. Включайте только если дублирование сообщений недопустимо. Требует Kafka 2.5+ и `replication.factor >= 3`.

### 2. Обработка ошибок

```java
KStream<String, String> inputStream = builder.stream("input-topic");

KStream<String, String> processedStream = inputStream
    .filter((key, value) -> {
        try {
            return value != null && !value.isEmpty();
        } catch (Exception e) {
            System.err.println("Filter error: " + e.getMessage());
            return false;
        }
    })
    .mapValues(value -> {
        try {
            return value.toUpperCase();
        } catch (Exception e) {
            System.err.println("Map error: " + e.getMessage());
            return "ERROR";
        }
    });

// Разветвление: успешные и ошибочные записи
processedStream.filter((key, value) -> !"ERROR".equals(value)).to("success-topic");
processedStream.filter((key, value) -> "ERROR".equals(value)).to("error-topic");
```

### 3. Тестирование

```java
public class StreamsTest {
    @Test
    public void testWordCount() {
        TopologyTestDriver testDriver = new TopologyTestDriver(createTopology());

        TestInputTopic<String, String> inputTopic =
            testDriver.createInputTopic("input-topic",
                new StringSerializer(), new StringSerializer());
        TestOutputTopic<String, Long> outputTopic =
            testDriver.createOutputTopic("output-topic",
                new StringDeserializer(), new LongDeserializer());

        inputTopic.pipeInput("key1", "hello world");
        inputTopic.pipeInput("key2", "hello kafka");

        List<KeyValue<String, Long>> results = outputTopic.readKeyValuesToList();
        assertFalse(results.isEmpty());

        testDriver.close();
    }

    private Topology createTopology() {
        StreamsBuilder builder = new StreamsBuilder();

        builder.stream("input-topic")
            .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
            .groupBy((key, value) -> value)
            .count()
            .toStream()
            .to("output-topic");

        return builder.build();
    }
}
```

## Вопросы для собеседования

### Базовые вопросы
1. **Что такое Kafka Streams?**
   - Java-библиотека для потоковой обработки, не требует отдельного кластера
   - Данные читаются из Kafka и записываются в Kafka
   - Поддерживает stateful обработку через State Stores

2. **В чём разница между KStream и KTable?**
   - KStream: поток событий, каждая запись обрабатывается независимо
   - KTable: таблица состояний, новая запись с тем же ключом заменяет предыдущую

3. **Что такое Topology?**
   - Граф операций: Source → Processor(s) → Sink
   - Определяет поток обработки данных

### Продвинутые вопросы
4. **Как обеспечить exactly-once?**
   - `processing.guarantee=exactly_once_v2`
   - Требует `replication.factor >= 3`, Kafka 2.5+

5. **Как масштабировать Kafka Streams приложение?**
   - Запустить несколько инстансов с одинаковым `application.id`
   - Tasks автоматически распределятся между инстансами
   - Добавить `num.standby.replicas` для быстрого восстановления

6. **Как обрабатывать ошибки?**
   - Try-catch внутри операторов DSL
   - Ветвление потока на успешные и ошибочные записи
   - `setUncaughtExceptionHandler` для необработанных исключений

### Практические вопросы
7. **Как создать оконную агрегацию?**
   ```java
   stream.groupBy((k, v) -> k)
         .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
         .count()
         .toStream()
         .to("output-topic");
   ```

8. **Как использовать State Store?**
   ```java
   StoreBuilder<KeyValueStore<String, Integer>> storeBuilder =
       Stores.keyValueStoreBuilder(
           Stores.persistentKeyValueStore("store"),
           Serdes.String(), Serdes.Integer()
       );
   ```

9. **Как тестировать Kafka Streams?**
   - `TopologyTestDriver` — не требует реального Kafka
   - `TestInputTopic` и `TestOutputTopic` для подачи и чтения данных
   - Быстро, детерминированно, без Docker
