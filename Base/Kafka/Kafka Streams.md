# Kafka Streams: потоковая обработка данных

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

**Kafka Streams** — это клиентская библиотека для построения приложений потоковой обработки данных и микросервисов, где входные и выходные данные хранятся в Apache Kafka. Она обеспечивает высокую производительность, горизонтальную масштабируемость, отказоустойчивость и exactly-once семантику.

### Основные преимущества:
- **Простота развёртывания**: обычные Java приложения
- **Горизонтальная масштабируемость**: автоматическое перераспределение нагрузки
- **Отказоустойчивость**: автоматическое восстановление после сбоев
- **Exactly-once семантика**: гарантия однократной обработки
- **Интеграция с экосистемой**: Schema Registry, Kafka Connect

## Архитектура

### Компоненты Kafka Streams

#### 1. Streams Application
```java
// Основной класс приложения
public class WordCountApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "word-count-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        StreamsBuilder builder = new StreamsBuilder();
        // Определение топологии
        Topology topology = builder.build();
        
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

#### 2. Topology (Топология)
- **Source**: входные топики
- **Processors**: операторы обработки
- **Sink**: выходные топики

#### 3. Tasks и Threads
- **Tasks**: единицы параллелизма
- **Threads**: выполняют задачи
- **Partitions**: определяют количество задач

## Основные концепции

### 1. KStream и KTable

```java
// KStream - поток событий
KStream<String, String> textLines = builder.stream("text-lines");

// KTable - таблица состояний
KTable<String, Long> wordCounts = builder.table("word-counts");

// GlobalKTable - глобальная таблица
GlobalKTable<String, String> userProfiles = builder.globalTable("user-profiles");
```

### 2. Serdes (Serializer/Deserializer)

```java
// Настройка Serdes
Properties props = new Properties();
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

// Кастомные Serdes
Serde<User> userSerde = Serdes.serdeFrom(new UserSerializer(), new UserDeserializer());
```

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
        
        // Чтение из топика
        KStream<String, String> inputStream = builder.stream("input-topic");
        
        // Фильтрация
        KStream<String, String> filteredStream = inputStream
            .filter((key, value) -> value != null && !value.isEmpty());
        
        // Преобразование
        KStream<String, String> transformedStream = filteredStream
            .mapValues(value -> value.toUpperCase());
        
        // Запись в топик
        transformedStream.to("output-topic");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
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
        
        KStream<String, String> inputStream = builder.stream("input-topic");
        
        // Группировка по ключу
        KGroupedStream<String, String> groupedStream = inputStream
            .groupBy((key, value) -> key);
        
        // Подсчёт количества
        KTable<String, Long> countTable = groupedStream.count();
        
        // Суммирование
        KTable<String, Integer> sumTable = groupedStream
            .aggregate(
                () -> 0,
                (key, value, aggregate) -> aggregate + Integer.parseInt(value),
                Materialized.with(Serdes.String(), Serdes.Integer())
            );
        
        // Запись результатов
        countTable.toStream().to("count-topic");
        sumTable.toStream().to("sum-topic");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
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
        
        KStream<String, String> inputStream = builder.stream("input-topic");
        
        // Группировка
        KGroupedStream<String, String> groupedStream = inputStream
            .groupBy((key, value) -> key);
        
        // Окно в 5 минут
        TimeWindows timeWindows = TimeWindows.of(Duration.ofMinutes(5));
        
        // Агрегация в окне
        KTable<Windowed<String>, Long> windowedCount = groupedStream
            .windowedBy(timeWindows)
            .count();
        
        // Запись результатов
        windowedCount.toStream()
            .map((key, value) -> KeyValue.pair(key.key(), value))
            .to("windowed-count-topic");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
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
        
        // Основной поток
        KStream<String, String> ordersStream = builder.stream("orders");
        
        // Таблица пользователей
        KTable<String, String> usersTable = builder.table("users");
        
        // Join поток с таблицей
        KStream<String, String> enrichedOrders = ordersStream
            .leftJoin(usersTable, 
                (orderValue, userValue) -> orderValue + " | " + userValue);
        
        // Запись результата
        enrichedOrders.to("enriched-orders");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

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
        // Обработка сообщения
        String[] words = value.toLowerCase().split("\\W+");
        
        for (String word : words) {
            if (!word.isEmpty()) {
                Integer count = kvStore.get(word);
                if (count == null) {
                    count = 0;
                }
                kvStore.put(word, count + 1);
                
                // Отправка результата
                context.forward(word, count + 1);
            }
        }
    }
    
    @Override
    public void close() {
        // Очистка ресурсов
    }
}

// Использование кастомного процессора
public class CustomProcessorApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "custom-processor-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        Topology topology = new Topology();
        
        // Добавление источника
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
public class StateStoreApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "state-store-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        Topology topology = new Topology();
        
        // Создание State Store
        StoreBuilder<KeyValueStore<String, Integer>> storeBuilder = 
            Stores.keyValueStoreBuilder(
                Stores.persistentKeyValueStore("word-count-store"),
                Serdes.String(),
                Serdes.Integer()
            );
        
        topology.addStateStore(storeBuilder);
        
        // Добавление процессора с доступом к State Store
        topology.addSource("source", "input-topic")
                .addProcessor("processor", CustomProcessor::new, "word-count-store")
                .addSink("sink", "output-topic", "processor");
        
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

## Агрегация и оконная обработка

### 1. Типы окон

```java
public class WindowTypesApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "window-types-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        StreamsBuilder builder = new StreamsBuilder();
        
        KStream<String, String> inputStream = builder.stream("input-topic");
        KGroupedStream<String, String> groupedStream = inputStream
            .groupBy((key, value) -> key);
        
        // Tumbling Window (фиксированное окно)
        TimeWindows tumblingWindow = TimeWindows.of(Duration.ofMinutes(5));
        
        // Hopping Window (скользящее окно)
        HoppingWindows hoppingWindow = HoppingWindows.of(
            Duration.ofMinutes(5), Duration.ofMinutes(1));
        
        // Session Window (сессионное окно)
        SessionWindows sessionWindow = SessionWindows.with(Duration.ofMinutes(5));
        
        // Агрегация в разных типах окон
        groupedStream.windowedBy(tumblingWindow).count()
            .toStream().to("tumbling-count-topic");
        
        groupedStream.windowedBy(hoppingWindow).count()
            .toStream().to("hopping-count-topic");
        
        groupedStream.windowedBy(sessionWindow).count()
            .toStream().to("session-count-topic");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### 2. Агрегация с кастомными функциями

```java
public class CustomAggregationApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "custom-aggregation-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        StreamsBuilder builder = new StreamsBuilder();
        
        KStream<String, String> inputStream = builder.stream("input-topic");
        KGroupedStream<String, String> groupedStream = inputStream
            .groupBy((key, value) -> key);
        
        // Кастомная агрегация
        KTable<String, CustomAggregate> customAggregate = groupedStream
            .aggregate(
                CustomAggregate::new, // Initializer
                (key, value, aggregate) -> aggregate.add(value), // Aggregator
                Materialized.with(Serdes.String(), new CustomAggregateSerde())
            );
        
        // Запись результата
        customAggregate.toStream().to("custom-aggregate-topic");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}

// Кастомный класс агрегации
class CustomAggregate {
    private int count;
    private double sum;
    private double min;
    private double max;
    
    public CustomAggregate() {
        this.count = 0;
        this.sum = 0.0;
        this.min = Double.MAX_VALUE;
        this.max = Double.MIN_VALUE;
    }
    
    public CustomAggregate add(String value) {
        double numValue = Double.parseDouble(value);
        this.count++;
        this.sum += numValue;
        this.min = Math.min(this.min, numValue);
        this.max = Math.max(this.max, numValue);
        return this;
    }
    
    // Геттеры
    public int getCount() { return count; }
    public double getSum() { return sum; }
    public double getMin() { return min; }
    public double getMax() { return max; }
    public double getAverage() { return count > 0 ? sum / count : 0.0; }
}
```

## Состояние и хранилища

### 1. Типы State Stores

```java
public class StateStoresApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "state-stores-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Key-Value Store
        StoreBuilder<KeyValueStore<String, Integer>> kvStoreBuilder = 
            Stores.keyValueStoreBuilder(
                Stores.persistentKeyValueStore("kv-store"),
                Serdes.String(),
                Serdes.Integer()
            );
        
        // Window Store
        StoreBuilder<WindowStore<String, Integer>> windowStoreBuilder = 
            Stores.windowStoreBuilder(
                Stores.persistentWindowStore("window-store", 
                    Duration.ofMinutes(5), Duration.ofMinutes(1), false),
                Serdes.String(),
                Serdes.Integer()
            );
        
        // Session Store
        StoreBuilder<SessionStore<String, Integer>> sessionStoreBuilder = 
            Stores.sessionStoreBuilder(
                Stores.persistentSessionStore("session-store", Duration.ofMinutes(5)),
                Serdes.String(),
                Serdes.Integer()
            );
        
        // Добавление State Stores
        builder.addStateStore(kvStoreBuilder);
        builder.addStateStore(windowStoreBuilder);
        builder.addStateStore(sessionStoreBuilder);
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### 2. Интерактивные запросы

```java
public class InteractiveQueriesApp {
    private KafkaStreams streams;
    private ReadOnlyKeyValueStore<String, Integer> kvStore;
    
    public void start() {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "interactive-queries-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Создание State Store
        StoreBuilder<KeyValueStore<String, Integer>> storeBuilder = 
            Stores.keyValueStoreBuilder(
                Stores.persistentKeyValueStore("word-count-store"),
                Serdes.String(),
                Serdes.Integer()
            );
        
        builder.addStateStore(storeBuilder);
        
        // Обработка потока
        builder.stream("input-topic")
            .groupBy((key, value) -> value)
            .count(Materialized.as("word-count-store"));
        
        Topology topology = builder.build();
        streams = new KafkaStreams(topology, props);
        streams.start();
        
        // Ожидание готовности
        streams.cleanUp();
        streams.start();
        
        // Получение доступа к State Store
        kvStore = streams.store("word-count-store", 
            QueryableStoreTypes.keyValueStore());
    }
    
    public Integer getWordCount(String word) {
        return kvStore.get(word);
    }
    
    public void close() {
        if (streams != null) {
            streams.close();
        }
    }
}
```

## Мониторинг и отладка

### 1. Метрики

```java
public class MetricsApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "metrics-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        // Включение метрик
        props.put(StreamsConfig.METRICS_RECORDING_LEVEL_CONFIG, "INFO");
        props.put(StreamsConfig.METRICS_NUM_SAMPLES_CONFIG, 2);
        props.put(StreamsConfig.METRICS_SAMPLE_WINDOW_MS_CONFIG, 30000);
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Простая топология
        builder.stream("input-topic")
            .mapValues(value -> value.toUpperCase())
            .to("output-topic");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        
        // Добавление listener для метрик
        streams.setUncaughtExceptionHandler((thread, throwable) -> {
            System.err.println("Uncaught exception in thread " + thread + ": " + throwable);
        });
        
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### 2. Логирование

```properties
# log4j.properties
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[%d] %p %m (%c)%n

# Логирование Kafka Streams
log4j.logger.org.apache.kafka.streams=INFO
log4j.logger.org.apache.kafka.clients=INFO
```

### 3. Отладка топологии

```java
public class TopologyDebugApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "topology-debug-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Сложная топология
        KStream<String, String> inputStream = builder.stream("input-topic");
        
        KStream<String, String> processedStream = inputStream
            .filter((key, value) -> value != null)
            .mapValues(value -> value.toUpperCase())
            .filter((key, value) -> value.length() > 5);
        
        processedStream.to("output-topic");
        
        Topology topology = builder.build();
        
        // Вывод топологии для отладки
        System.out.println("Topology description:");
        System.out.println(topology.describe());
        
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

## Лучшие практики

### 1. Конфигурация производительности

```java
public class OptimizedStreamsApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "optimized-streams-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        // Настройки производительности
        props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 4);
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
        props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024);
        props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
        
        // Настройки восстановления
        props.put(StreamsConfig.REPLICATION_FACTOR_CONFIG, 3);
        props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Оптимизированная топология
        builder.stream("input-topic")
            .filter((key, value) -> value != null)
            .mapValues(value -> value.toUpperCase())
            .to("output-topic");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        
        // Graceful shutdown
        streams.setUncaughtExceptionHandler((thread, throwable) -> {
            System.err.println("Uncaught exception in thread " + thread + ": " + throwable);
        });
        
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            streams.close(Duration.ofSeconds(10));
        }));
    }
}
```

### 2. Обработка ошибок

```java
public class ErrorHandlingApp {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "error-handling-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        
        StreamsBuilder builder = new StreamsBuilder();
        
        KStream<String, String> inputStream = builder.stream("input-topic");
        
        // Обработка с проверкой ошибок
        KStream<String, String> processedStream = inputStream
            .filter((key, value) -> {
                try {
                    return value != null && !value.isEmpty();
                } catch (Exception e) {
                    System.err.println("Error filtering record: " + e.getMessage());
                    return false;
                }
            })
            .mapValues(value -> {
                try {
                    return value.toUpperCase();
                } catch (Exception e) {
                    System.err.println("Error processing value: " + e.getMessage());
                    return "ERROR";
                }
            });
        
        // Разделение на успешные и ошибочные записи
        processedStream.filter((key, value) -> !"ERROR".equals(value))
            .to("success-topic");
        
        processedStream.filter((key, value) -> "ERROR".equals(value))
            .to("error-topic");
        
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### 3. Тестирование

```java
public class StreamsTest {
    @Test
    public void testWordCount() {
        // Создание тестового драйвера
        TopologyTestDriver testDriver = new TopologyTestDriver(createTopology());
        
        // Создание тестовых данных
        TestInputTopic<String, String> inputTopic = testDriver.createInputTopic("input-topic");
        TestOutputTopic<String, Long> outputTopic = testDriver.createOutputTopic("output-topic");
        
        // Отправка тестовых сообщений
        inputTopic.pipeInput("key1", "hello world");
        inputTopic.pipeInput("key2", "hello kafka");
        inputTopic.pipeInput("key1", "hello streams");
        
        // Проверка результатов
        List<KeyValue<String, Long>> results = outputTopic.readKeyValuesToList();
        
        assertEquals(3, results.size());
        assertEquals("hello", results.get(0).key);
        assertEquals(1L, results.get(0).value);
        
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
   - Библиотека для потоковой обработки данных
   - Клиентская библиотека для Java
   - Обеспечивает exactly-once семантику

2. **В чём разница между KStream и KTable?**
   - KStream: поток событий
   - KTable: таблица состояний
   - KTable обновляется при изменении ключа

3. **Что такое Topology в Kafka Streams?**
   - Граф операций обработки данных
   - Состоит из source, processors, sink
   - Определяет поток данных

### Продвинутые вопросы
4. **Как обеспечить exactly-once семантику?**
   - Настройка processing.guarantee
   - Использование транзакций
   - Правильная настройка idempotence

5. **Как масштабировать Kafka Streams приложение?**
   - Увеличение количества потоков
   - Партиционирование данных
   - Добавление standby replicas

6. **Как обрабатывать ошибки в Kafka Streams?**
   - Try-catch блоки
   - Фильтрация ошибочных записей
   - Dead Letter Queue

### Практические вопросы
7. **Как создать оконную агрегацию?**
   ```java
   KGroupedStream<String, String> grouped = stream.groupBy((k, v) -> k);
   grouped.windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
         .count()
         .toStream()
         .to("output-topic");
   ```

8. **Как использовать State Store?**
   ```java
   StoreBuilder<KeyValueStore<String, Integer>> storeBuilder = 
       Stores.keyValueStoreBuilder(
           Stores.persistentKeyValueStore("store"),
           Serdes.String(),
           Serdes.Integer()
       );
   ```

9. **Как тестировать Kafka Streams приложение?**
   - TopologyTestDriver
   - Тестовые топики
   - Проверка результатов

---

**Дополнительные ресурсы:**
- [Kafka Streams Documentation](https://kafka.apache.org/documentation/streams/)
- [Kafka Streams Developer Guide](https://docs.confluent.io/platform/current/streams/developer-guide/)
- [Kafka Streams Examples](https://github.com/confluentinc/kafka-streams-examples) 