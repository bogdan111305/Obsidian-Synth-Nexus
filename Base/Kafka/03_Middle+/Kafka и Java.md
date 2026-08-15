# Kafka и Java: руководство по разработке

> [!QUOTE] KafkaProducer — потокобезопасный клиент для отправки сообщений в Kafka. Буферизует записи в батчи и отправляет асинхронно. Один инстанс Producer можно использовать для нескольких топиков.

## Оглавление
1. [Настройка проекта](#настройка)
2. [Producer API](#producer)
3. [Consumer API](#consumer)
4. [Транзакции](#транзакции)
5. [Сериализация](#сериализация)
6. [Обработка ошибок](#ошибки)
7. [Мониторинг](#мониторинг)
8. [Лучшие практики](#практики)
9. [Вопросы для собеседования](#вопросы)

## Настройка проекта

### Maven зависимости

```xml
<dependencies>
    <!-- Kafka Client -->
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>3.5.1</version>
    </dependency>

    <!-- JSON обработка -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.15.2</version>
    </dependency>

    <!-- Логирование -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>2.0.7</version>
    </dependency>

    <!-- Тестирование с реальным Kafka -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>kafka</artifactId>
        <version>1.19.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle зависимости

```groovy
dependencies {
    implementation 'org.apache.kafka:kafka-clients:3.5.1'
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.15.2'
    implementation 'org.slf4j:slf4j-api:2.0.7'
    testImplementation 'org.testcontainers:kafka:1.19.0'
}
```

## Producer API

### 1. Базовый Producer

```java
import org.apache.kafka.clients.producer.*;
import java.util.Properties;

public class SimpleProducer {
    private final KafkaProducer<String, String> producer;

    public SimpleProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);

        this.producer = new KafkaProducer<>(props);
    }

    public void sendMessage(String topic, String key, String value) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

        producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                System.err.println("Error sending message: " + exception.getMessage());
            } else {
                System.out.println("Sent to " + metadata.topic() +
                                 " partition=" + metadata.partition() +
                                 " offset=" + metadata.offset());
            }
        });
    }

    public void close() {
        producer.close();
    }
}
```

> [!INFO] `acks=all` + `retries=3` + `enable.idempotence=true` — рекомендуемая комбинация для надёжной отправки. Без `enable.idempotence` при retry возможны дублирующиеся сообщения.

### 2. Асинхронный Producer с CompletableFuture

```java
public class AsyncProducer {
    private final KafkaProducer<String, String> producer;

    public AsyncProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.ACKS_CONFIG, "1");
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

        this.producer = new KafkaProducer<>(props);
    }

    public CompletableFuture<RecordMetadata> sendAsync(String topic, String key, String value) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);
        CompletableFuture<RecordMetadata> future = new CompletableFuture<>();

        producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                future.completeExceptionally(exception);
            } else {
                future.complete(metadata);
            }
        });

        return future;
    }

    public void close() { producer.close(); }
}
```

### 3. Producer с кастомным партиционером

```java
public class CustomPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        if (keyBytes == null) {
            return new Random().nextInt(numPartitions);
        }

        int hash = Arrays.hashCode(keyBytes);
        return Math.abs(hash) % numPartitions;
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}

// Использование:
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomPartitioner.class.getName());
```

## Consumer API

### 1. Базовый Consumer

```java
import org.apache.kafka.clients.consumer.*;
import java.time.Duration;
import java.util.Arrays;
import java.util.Properties;

public class SimpleConsumer {
    private final KafkaConsumer<String, String> consumer;

    public SimpleConsumer(String groupId) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "30000");
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, "3000");

        this.consumer = new KafkaConsumer<>(props);
    }

    public void consumeMessages(String topic) {
        consumer.subscribe(Arrays.asList(topic));

        try {
            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("offset=%d, key=%s, value=%s%n",
                                    record.offset(), record.key(), record.value());
                    processMessage(record);
                }
            }
        } finally {
            consumer.close();
        }
    }

    private void processMessage(ConsumerRecord<String, String> record) {
        System.out.println("Processing: " + record.value());
    }

    public void close() { consumer.close(); }
}
```

> [!WARNING] `enable.auto.commit=true` — offset коммитится автоматически по таймеру. Если приложение упадёт после poll, но до завершения обработки, сообщения могут быть потеряны (обработаны не будут, но offset уже сдвинут). Для надёжности используйте `enable.auto.commit=false` и ручной commit после обработки.

### 2. Consumer с ручным commit

```java
public class ManualCommitConsumer {
    private final KafkaConsumer<String, String> consumer;

    public ManualCommitConsumer(String groupId) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "30000");
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, "3000");

        this.consumer = new KafkaConsumer<>(props);
    }

    public void consumeMessages(String topic) {
        consumer.subscribe(Arrays.asList(topic));

        try {
            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                for (ConsumerRecord<String, String> record : records) {
                    try {
                        processMessage(record);

                        // Коммит offset только после успешной обработки
                        consumer.commitSync(Collections.singletonMap(
                            new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset() + 1)
                        ));

                    } catch (Exception e) {
                        System.err.println("Error processing: " + e.getMessage());
                        // Не коммитим — сообщение будет перечитано
                    }
                }
            }
        } finally {
            consumer.close();
        }
    }

    private void processMessage(ConsumerRecord<String, String> record) {
        System.out.println("Processing: " + record.value());
    }
}
```

### 3. Consumer с обработкой по партициям

```java
public class PartitionAwareConsumer {
    private final KafkaConsumer<String, String> consumer;

    public PartitionAwareConsumer(String groupId) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        this.consumer = new KafkaConsumer<>(props);
    }

    public void consumeMessages(String topic) {
        consumer.subscribe(Arrays.asList(topic));

        try {
            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                // Группировка по партициям
                for (Map.Entry<TopicPartition, List<ConsumerRecord<String, String>>> entry :
                     records.partitions().entrySet()) {

                    TopicPartition partition = entry.getKey();
                    List<ConsumerRecord<String, String>> partitionRecords = entry.getValue();

                    for (ConsumerRecord<String, String> record : partitionRecords) {
                        processMessage(record);
                    }
                }

                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }

    private void processMessage(ConsumerRecord<String, String> record) {
        System.out.printf("Partition: %d, Offset: %d, Key: %s, Value: %s%n",
                         record.partition(), record.offset(),
                         record.key(), record.value());
    }
}
```

## Транзакции

### 1. Producer с транзакциями

```java
public class TransactionalProducer {
    private final KafkaProducer<String, String> producer;

    public TransactionalProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-transactional-producer");

        this.producer = new KafkaProducer<>(props);
        this.producer.initTransactions();
    }

    public void sendTransactional(String topic1, String topic2) {
        try {
            producer.beginTransaction();

            producer.send(new ProducerRecord<>(topic1, "key1", "value1"));
            producer.send(new ProducerRecord<>(topic1, "key2", "value2"));
            producer.send(new ProducerRecord<>(topic2, "key3", "value3"));

            producer.commitTransaction();

        } catch (Exception e) {
            producer.abortTransaction();
            throw new RuntimeException("Transaction failed", e);
        }
    }

    public void close() { producer.close(); }
}
```

> [!WARNING] Для exactly-once семантики нужны три условия одновременно:
> 1. `enable.idempotence=true` на producer
> 2. `transactional.id` задан уникальным для каждого инстанса producer
> 3. `isolation.level=read_committed` на consumer
>
> Без последнего consumer может читать незакоммиченные (впоследствии отменённые) транзакции.

### 2. Consumer с транзакциями

```java
public class TransactionalConsumer {
    private final KafkaConsumer<String, String> consumer;

    public TransactionalConsumer(String groupId) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");  // Только подтверждённые транзакции

        this.consumer = new KafkaConsumer<>(props);
    }

    public void consumeTransactional(String topic) {
        consumer.subscribe(Arrays.asList(topic));

        try {
            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                for (ConsumerRecord<String, String> record : records) {
                    processMessage(record);
                }

                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }

    private void processMessage(ConsumerRecord<String, String> record) {
        System.out.println("Transactional message: " + record.value());
    }
}
```

## Сериализация

### 1. Кастомный сериализатор/десериализатор

```java
public class UserSerializer implements Serializer<User> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public byte[] serialize(String topic, User user) {
        try {
            return objectMapper.writeValueAsBytes(user);
        } catch (Exception e) {
            throw new SerializationException("Error serializing User", e);
        }
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {}
}

public class UserDeserializer implements Deserializer<User> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public User deserialize(String topic, byte[] data) {
        try {
            return objectMapper.readValue(data, User.class);
        } catch (Exception e) {
            throw new SerializationException("Error deserializing User", e);
        }
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {}
}

// Модель данных
public class User {
    private String id;
    private String name;
    private String email;
    // конструкторы, геттеры, сеттеры
}
```

### 2. Использование кастомных сериализаторов

```java
public class CustomSerializerProducer {
    private final KafkaProducer<String, User> producer;

    public CustomSerializerProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                 UserSerializer.class.getName());

        this.producer = new KafkaProducer<>(props);
    }

    public void sendUser(String topic, String key, User user) {
        producer.send(new ProducerRecord<>(topic, key, user), (metadata, exception) -> {
            if (exception != null) {
                System.err.println("Error: " + exception.getMessage());
            } else {
                System.out.println("User sent successfully");
            }
        });
    }

    public void close() { producer.close(); }
}
```

## Обработка ошибок

### 1. Retry механизм

```java
public class RetryProducer {
    private final KafkaProducer<String, String> producer;
    private final int maxRetries;

    public RetryProducer(int maxRetries) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, maxRetries);
        props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 1000);

        this.producer = new KafkaProducer<>(props);
        this.maxRetries = maxRetries;
    }

    public void sendWithRetry(String topic, String key, String value) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

        int attempts = 0;
        while (attempts < maxRetries) {
            try {
                producer.send(record).get(5, TimeUnit.SECONDS);
                return;
            } catch (Exception e) {
                attempts++;
                System.err.println("Attempt " + attempts + " failed: " + e.getMessage());

                if (attempts >= maxRetries) {
                    throw new RuntimeException("Failed after " + maxRetries + " attempts", e);
                }

                try {
                    Thread.sleep(1000L * attempts);  // экспоненциальная задержка
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Interrupted during retry", ie);
                }
            }
        }
    }

    public void close() { producer.close(); }
}
```

### 2. Dead Letter Queue

```java
public class DeadLetterQueueConsumer {
    private final KafkaConsumer<String, String> consumer;
    private final KafkaProducer<String, String> dlqProducer;

    public DeadLetterQueueConsumer(String groupId) {
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                         "org.apache.kafka.common.serialization.StringDeserializer");
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                         "org.apache.kafka.common.serialization.StringDeserializer");
        consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        consumerProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                         "org.apache.kafka.common.serialization.StringSerializer");
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                         "org.apache.kafka.common.serialization.StringSerializer");

        this.consumer = new KafkaConsumer<>(consumerProps);
        this.dlqProducer = new KafkaProducer<>(producerProps);
    }

    public void consumeWithDLQ(String topic, String dlqTopic) {
        consumer.subscribe(Arrays.asList(topic));

        try {
            while (true) {
                ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(100));

                for (ConsumerRecord<String, String> record : records) {
                    try {
                        processMessage(record);
                        consumer.commitSync(Collections.singletonMap(
                            new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset() + 1)
                        ));
                    } catch (Exception e) {
                        sendToDLQ(dlqTopic, record, e);
                        consumer.commitSync(Collections.singletonMap(
                            new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset() + 1)
                        ));
                    }
                }
            }
        } finally {
            consumer.close();
            dlqProducer.close();
        }
    }

    private void processMessage(ConsumerRecord<String, String> record) {
        if (record.value().contains("error")) {
            throw new RuntimeException("Simulated processing error");
        }
        System.out.println("Processing: " + record.value());
    }

    private void sendToDLQ(String dlqTopic, ConsumerRecord<String, String> original, Exception error) {
        String dlqMessage = String.format(
            "{\"original\": \"%s\", \"error\": \"%s\", \"timestamp\": \"%s\"}",
            original.value(), error.getMessage(), Instant.now());

        dlqProducer.send(new ProducerRecord<>(dlqTopic, original.key(), dlqMessage));
    }
}
```

## Мониторинг

### 1. Метрики Producer через Micrometer

```java
public class MetricsProducer {
    private final KafkaProducer<String, String> producer;
    private final MeterRegistry meterRegistry;

    public MetricsProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.METRICS_RECORDING_LEVEL_CONFIG, "INFO");

        this.producer = new KafkaProducer<>(props);
        this.meterRegistry = new SimpleMeterRegistry();
    }

    public void sendMessage(String topic, String key, String value) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);
        Timer.Sample sample = Timer.start(meterRegistry);

        producer.send(record, (metadata, exception) -> {
            sample.stop(Timer.builder("kafka.send.latency").register(meterRegistry));

            if (exception != null) {
                Counter.builder("kafka.send.errors").register(meterRegistry).increment();
            } else {
                Counter.builder("kafka.messages.sent").register(meterRegistry).increment();
            }
        });
    }

    public void close() { producer.close(); }
}
```

## Лучшие практики

### 1. Конфигурация Producer (оптимальная)

```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
         "org.apache.kafka.common.serialization.StringSerializer");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
         "org.apache.kafka.common.serialization.StringSerializer");

// Производительность
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
props.put(ProducerConfig.LINGER_MS_CONFIG, 5);
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

// Надёжность
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 1000);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");

// Мониторинг
props.put(ProducerConfig.METRICS_RECORDING_LEVEL_CONFIG, "INFO");
```

### 2. Конфигурация Consumer (оптимальная)

```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
         "org.apache.kafka.common.serialization.StringDeserializer");
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
         "org.apache.kafka.common.serialization.StringDeserializer");

// Производительность
props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1);
props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);

// Надёжность
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);
```

> [!WARNING] `MAX_POLL_INTERVAL_MS_CONFIG` — если consumer не вызывает `poll()` в течение этого времени, Kafka считает его мёртвым и запускает rebalance. Убедитесь, что обработка батча умещается в этот интервал. При долгой обработке увеличьте значение или уменьшите `MAX_POLL_RECORDS_CONFIG`.

## Вопросы для собеседования

### Базовые вопросы
1. **Как создать Producer в Java?**
   ```java
   Properties props = new Properties();
   props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
   props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
   props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
   KafkaProducer<String, String> producer = new KafkaProducer<>(props);
   ```

2. **Как создать Consumer в Java?**
   ```java
   Properties props = new Properties();
   props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
   props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-group");
   props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
   props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
   KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
   ```

3. **Что такое callback в Producer?**
   - Асинхронная обработка результата отправки
   - Вызывается после подтверждения от брокера (или при ошибке)
   - Не блокирует основной поток — улучшает пропускную способность

### Продвинутые вопросы
4. **Как реализовать exactly-once семантику?**
   - `enable.idempotence=true` + `transactional.id` на producer
   - `isolation.level=read_committed` на consumer
   - `beginTransaction()` / `commitTransaction()` / `abortTransaction()`

5. **Как обрабатывать ошибки в Consumer?**
   - Dead Letter Queue (DLQ) для сообщений, которые не удалось обработать
   - Ручной retry с экспоненциальной задержкой
   - `ENABLE_AUTO_COMMIT=false` + ручной `commitSync` только после успеха

6. **Как оптимизировать производительность?**
   - Увеличить `batch.size` и `linger.ms` (больше батч = меньше запросов)
   - Включить сжатие: `compression.type=snappy` или `lz4`
   - Правильно выбрать число партиций для параллелизма

### Практические вопросы
7. **Как реализовать retry механизм?**
   ```java
   int maxRetries = 3;
   for (int attempt = 0; attempt < maxRetries; attempt++) {
       try {
           producer.send(record).get();
           break;
       } catch (Exception e) {
           if (attempt == maxRetries - 1) throw e;
           Thread.sleep(1000L * (attempt + 1));
       }
   }
   ```

8. **Как мониторить производительность?**
   - JMX метрики (доступны из коробки)
   - Micrometer + Prometheus + Grafana

9. **Как обеспечить порядок сообщений?**
   - Использовать ключ (один ключ — одна партиция)
   - `max.in.flight.requests.per.connection=1` без idempotence
   - Или `enable.idempotence=true` (допускает 5 in-flight при сохранении порядка)
