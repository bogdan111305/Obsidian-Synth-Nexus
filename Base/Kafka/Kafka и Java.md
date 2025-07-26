# Kafka и Java: полное руководство по разработке

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

### 1. Maven зависимости

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
    
    <!-- Тестирование -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>kafka</artifactId>
        <version>1.19.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 2. Gradle зависимости

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
import java.util.concurrent.Future;

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
        ProducerRecord<String, String> record = 
            new ProducerRecord<>(topic, key, value);
        
        producer.send(record, new Callback() {
            @Override
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                if (exception != null) {
                    System.err.println("Error sending message: " + exception.getMessage());
                } else {
                    System.out.println("Message sent to " + metadata.topic() + 
                                     " partition " + metadata.partition() + 
                                     " offset " + metadata.offset());
                }
            }
        });
    }
    
    public void close() {
        producer.close();
    }
}
```

### 2. Асинхронный Producer

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
        props.put(ProducerConfig.RETRIES_CONFIG, 0);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        
        this.producer = new KafkaProducer<>(props);
    }
    
    public CompletableFuture<RecordMetadata> sendMessageAsync(String topic, String key, String value) {
        ProducerRecord<String, String> record = 
            new ProducerRecord<>(topic, key, value);
        
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
    
    public void close() {
        producer.close();
    }
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
        
        // Простая хеш-функция для ключа
        int hash = Arrays.hashCode(keyBytes);
        return Math.abs(hash) % numPartitions;
    }
    
    @Override
    public void close() {}
    
    @Override
    public void configure(Map<String, ?> configs) {}
}

// Использование
Properties props = new Properties();
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
                    System.out.printf("offset = %d, key = %s, value = %s%n",
                                    record.offset(), record.key(), record.value());
                    
                    // Обработка сообщения
                    processMessage(record);
                }
            }
        } finally {
            consumer.close();
        }
    }
    
    private void processMessage(ConsumerRecord<String, String> record) {
        // Логика обработки сообщения
        System.out.println("Processing message: " + record.value());
    }
    
    public void close() {
        consumer.close();
    }
}
```

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
                        // Обработка сообщения
                        processMessage(record);
                        
                        // Ручной commit после успешной обработки
                        consumer.commitSync(Collections.singletonMap(
                            new TopicPartition(record.topic(), record.partition()),
                            new OffsetAndMetadata(record.offset() + 1)
                        ));
                        
                    } catch (Exception e) {
                        System.err.println("Error processing message: " + e.getMessage());
                        // Не делаем commit при ошибке
                    }
                }
            }
        } finally {
            consumer.close();
        }
    }
    
    private void processMessage(ConsumerRecord<String, String> record) {
        // Логика обработки сообщения
        System.out.println("Processing message: " + record.value());
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
                
                // Группировка записей по партициям
                Map<TopicPartition, List<ConsumerRecord<String, String>>> recordsByPartition = 
                    records.partitions();
                
                for (Map.Entry<TopicPartition, List<ConsumerRecord<String, String>>> entry : 
                     recordsByPartition.entrySet()) {
                    
                    TopicPartition partition = entry.getKey();
                    List<ConsumerRecord<String, String>> partitionRecords = entry.getValue();
                    
                    System.out.println("Processing partition: " + partition.partition() + 
                                     " with " + partitionRecords.size() + " records");
                    
                    for (ConsumerRecord<String, String> record : partitionRecords) {
                        processMessage(record);
                    }
                }
                
                // Commit всех обработанных сообщений
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
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-transactional-producer");
        
        this.producer = new KafkaProducer<>(props);
        this.producer.initTransactions();
    }
    
    public void sendTransactionalMessages(String topic1, String topic2) {
        try {
            producer.beginTransaction();
            
            // Отправка сообщений в первый топик
            producer.send(new ProducerRecord<>(topic1, "key1", "value1"));
            producer.send(new ProducerRecord<>(topic1, "key2", "value2"));
            
            // Отправка сообщений во второй топик
            producer.send(new ProducerRecord<>(topic2, "key3", "value3"));
            producer.send(new ProducerRecord<>(topic2, "key4", "value4"));
            
            // Подтверждение транзакции
            producer.commitTransaction();
            
        } catch (Exception e) {
            // Откат транзакции при ошибке
            producer.abortTransaction();
            throw new RuntimeException("Transaction failed", e);
        }
    }
    
    public void close() {
        producer.close();
    }
}
```

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
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        
        this.consumer = new KafkaConsumer<>(props);
    }
    
    public void consumeTransactionalMessages(String topic) {
        consumer.subscribe(Arrays.asList(topic));
        
        try {
            while (true) {
                ConsumerRecords<String, String> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, String> record : records) {
                    // Обработка только подтверждённых транзакций
                    processMessage(record);
                }
                
                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }
    
    private void processMessage(ConsumerRecord<String, String> record) {
        System.out.println("Processing transactional message: " + record.value());
    }
}
```

## Сериализация

### 1. Кастомный сериализатор

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
    
    // Конструкторы, геттеры, сеттеры
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
        ProducerRecord<String, User> record = 
            new ProducerRecord<>(topic, key, user);
        
        producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                System.err.println("Error sending user: " + exception.getMessage());
            } else {
                System.out.println("User sent successfully");
            }
        });
    }
    
    public void close() {
        producer.close();
    }
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
    
    public void sendMessageWithRetry(String topic, String key, String value) {
        ProducerRecord<String, String> record = 
            new ProducerRecord<>(topic, key, value);
        
        int attempts = 0;
        while (attempts < maxRetries) {
            try {
                Future<RecordMetadata> future = producer.send(record);
                RecordMetadata metadata = future.get(5, TimeUnit.SECONDS);
                System.out.println("Message sent successfully");
                return;
            } catch (Exception e) {
                attempts++;
                System.err.println("Attempt " + attempts + " failed: " + e.getMessage());
                
                if (attempts >= maxRetries) {
                    throw new RuntimeException("Failed to send message after " + maxRetries + " attempts", e);
                }
                
                // Экспоненциальная задержка
                try {
                    Thread.sleep(1000 * attempts);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Interrupted during retry", ie);
                }
            }
        }
    }
    
    public void close() {
        producer.close();
    }
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
                        // Отправка в Dead Letter Queue
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
        // Логика обработки сообщения
        if (record.value().contains("error")) {
            throw new RuntimeException("Simulated processing error");
        }
        System.out.println("Processing message: " + record.value());
    }
    
    private void sendToDLQ(String dlqTopic, ConsumerRecord<String, String> originalRecord, Exception error) {
        String dlqMessage = String.format("{\"original_message\": \"%s\", \"error\": \"%s\", \"timestamp\": \"%s\"}",
                                        originalRecord.value(), error.getMessage(), 
                                        Instant.now().toString());
        
        ProducerRecord<String, String> dlqRecord = 
            new ProducerRecord<>(dlqTopic, originalRecord.key(), dlqMessage);
        
        dlqProducer.send(dlqRecord, (metadata, exception) -> {
            if (exception != null) {
                System.err.println("Failed to send to DLQ: " + exception.getMessage());
            } else {
                System.out.println("Message sent to DLQ successfully");
            }
        });
    }
}
```

## Мониторинг

### 1. Метрики Producer

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
        
        // Регистрация метрик
        registerMetrics();
    }
    
    private void registerMetrics() {
        // Метрики для отслеживания отправленных сообщений
        Counter messagesSent = Counter.builder("kafka.messages.sent")
            .description("Number of messages sent")
            .register(meterRegistry);
        
        // Метрики для отслеживания ошибок
        Counter sendErrors = Counter.builder("kafka.send.errors")
            .description("Number of send errors")
            .register(meterRegistry);
        
        // Метрики для отслеживания латентности
        Timer sendLatency = Timer.builder("kafka.send.latency")
            .description("Message send latency")
            .register(meterRegistry);
    }
    
    public void sendMessage(String topic, String key, String value) {
        ProducerRecord<String, String> record = 
            new ProducerRecord<>(topic, key, value);
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        producer.send(record, (metadata, exception) -> {
            sample.stop(Timer.builder("kafka.send.latency").register(meterRegistry));
            
            if (exception != null) {
                Counter.builder("kafka.send.errors").register(meterRegistry).increment();
                System.err.println("Error sending message: " + exception.getMessage());
            } else {
                Counter.builder("kafka.messages.sent").register(meterRegistry).increment();
                System.out.println("Message sent successfully");
            }
        });
    }
    
    public void close() {
        producer.close();
    }
}
```

### 2. Метрики Consumer

```java
public class MetricsConsumer {
    private final KafkaConsumer<String, String> consumer;
    private final MeterRegistry meterRegistry;
    
    public MetricsConsumer(String groupId) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.METRICS_RECORDING_LEVEL_CONFIG, "INFO");
        
        this.consumer = new KafkaConsumer<>(props);
        this.meterRegistry = new SimpleMeterRegistry();
        
        registerMetrics();
    }
    
    private void registerMetrics() {
        // Метрики для отслеживания обработанных сообщений
        Counter messagesProcessed = Counter.builder("kafka.messages.processed")
            .description("Number of messages processed")
            .register(meterRegistry);
        
        // Метрики для отслеживания ошибок обработки
        Counter processingErrors = Counter.builder("kafka.processing.errors")
            .description("Number of processing errors")
            .register(meterRegistry);
        
        // Метрики для отслеживания времени обработки
        Timer processingTime = Timer.builder("kafka.processing.time")
            .description("Message processing time")
            .register(meterRegistry);
    }
    
    public void consumeMessages(String topic) {
        consumer.subscribe(Arrays.asList(topic));
        
        try {
            while (true) {
                ConsumerRecords<String, String> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, String> record : records) {
                    Timer.Sample sample = Timer.start(meterRegistry);
                    
                    try {
                        processMessage(record);
                        Counter.builder("kafka.messages.processed").register(meterRegistry).increment();
                    } catch (Exception e) {
                        Counter.builder("kafka.processing.errors").register(meterRegistry).increment();
                        System.err.println("Error processing message: " + e.getMessage());
                    } finally {
                        sample.stop(Timer.builder("kafka.processing.time").register(meterRegistry));
                    }
                }
                
                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }
    
    private void processMessage(ConsumerRecord<String, String> record) {
        System.out.println("Processing message: " + record.value());
    }
}
```

## Лучшие практики

### 1. Конфигурация Producer

```java
public class OptimizedProducer {
    private final KafkaProducer<String, String> producer;
    
    public OptimizedProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
                 "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
                 "org.apache.kafka.common.serialization.StringSerializer");
        
        // Настройки производительности
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        
        // Настройки надёжности
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 1000);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
        
        // Настройки мониторинга
        props.put(ProducerConfig.METRICS_RECORDING_LEVEL_CONFIG, "INFO");
        props.put(ProducerConfig.METRICS_NUM_SAMPLES_CONFIG, 2);
        props.put(ProducerConfig.METRICS_SAMPLE_WINDOW_MS_CONFIG, 30000);
        
        this.producer = new KafkaProducer<>(props);
    }
    
    public void sendMessage(String topic, String key, String value) {
        ProducerRecord<String, String> record = 
            new ProducerRecord<>(topic, key, value);
        
        producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                handleSendError(exception, record);
            } else {
                handleSendSuccess(metadata, record);
            }
        });
    }
    
    private void handleSendError(Exception exception, ProducerRecord<String, String> record) {
        if (exception instanceof RetriableException) {
            // Повторная попытка для временных ошибок
            System.err.println("Retriable error: " + exception.getMessage());
        } else {
            // Критическая ошибка
            System.err.println("Critical error: " + exception.getMessage());
        }
    }
    
    private void handleSendSuccess(RecordMetadata metadata, ProducerRecord<String, String> record) {
        System.out.println("Message sent to " + metadata.topic() + 
                         " partition " + metadata.partition() + 
                         " offset " + metadata.offset());
    }
    
    public void close() {
        producer.close();
    }
}
```

### 2. Конфигурация Consumer

```java
public class OptimizedConsumer {
    private final KafkaConsumer<String, String> consumer;
    
    public OptimizedConsumer(String groupId) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
                 "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
                 "org.apache.kafka.common.serialization.StringDeserializer");
        
        // Настройки производительности
        props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1);
        props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);
        
        // Настройки надёжности
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);
        
        // Настройки мониторинга
        props.put(ConsumerConfig.METRICS_RECORDING_LEVEL_CONFIG, "INFO");
        
        this.consumer = new KafkaConsumer<>(props);
    }
    
    public void consumeMessages(String topic) {
        consumer.subscribe(Arrays.asList(topic));
        
        try {
            while (true) {
                ConsumerRecords<String, String> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                if (!records.isEmpty()) {
                    processBatch(records);
                    consumer.commitSync();
                }
            }
        } catch (WakeupException e) {
            // Graceful shutdown
            System.out.println("Consumer shutting down...");
        } finally {
            consumer.close();
        }
    }
    
    private void processBatch(ConsumerRecords<String, String> records) {
        for (ConsumerRecord<String, String> record : records) {
            try {
                processMessage(record);
            } catch (Exception e) {
                handleProcessingError(e, record);
            }
        }
    }
    
    private void processMessage(ConsumerRecord<String, String> record) {
        System.out.println("Processing message: " + record.value());
    }
    
    private void handleProcessingError(Exception e, ConsumerRecord<String, String> record) {
        System.err.println("Error processing message: " + e.getMessage());
        // Логика обработки ошибок (DLQ, retry, etc.)
    }
    
    public void close() {
        consumer.wakeup();
    }
}
```

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
   - Позволяет обрабатывать успех/ошибку без блокировки
   - Улучшает производительность

### Продвинутые вопросы
4. **Как реализовать exactly-once семантику?**
   - Идемпотентные producer'ы
   - Транзакции
   - Consumer group координация

5. **Как обрабатывать ошибки в Consumer?**
   - Dead Letter Queue
   - Retry механизм
   - Ручной commit

6. **Как оптимизировать производительность?**
   - Настройка batch size
   - Сжатие данных
   - Оптимизация количества партиций

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
           Thread.sleep(1000 * (attempt + 1));
       }
   }
   ```

8. **Как мониторить производительность?**
   - JMX метрики
   - Micrometer
   - Кастомные метрики

9. **Как обеспечить порядок сообщений?**
   - Использование ключей
   - Одна партиция
   - Consumer group с одним consumer'ом

---

**Дополнительные ресурсы:**
- [Kafka Java API](https://kafka.apache.org/documentation/#api)
- [Kafka Producer API](https://kafka.apache.org/documentation/#producerapi)
- [Kafka Consumer API](https://kafka.apache.org/documentation/#consumerapi) 