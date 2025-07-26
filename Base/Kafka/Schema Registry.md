# Schema Registry в Kafka: подробный справочник

## Оглавление
1. [Что такое Schema Registry](#что-это)
2. [Зачем нужен Schema Registry](#зачем)
3. [Архитектура и компоненты](#архитектура)
4. [Установка и запуск](#установка)
5. [Работа с Avro, Protobuf, JSON Schema](#форматы)
6. [Интеграция с Kafka Producers/Consumers](#интеграция)
7. [Эволюция схем и совместимость](#эволюция)
8. [REST API Schema Registry](#rest-api)
9. [Best practices](#best-practices)
10. [Вопросы для собеседования](#вопросы)

---

## 1. Что такое Schema Registry <a name="что-это"></a>

**Schema Registry** — это отдельный сервис, который хранит схемы (описания структуры сообщений) для топиков Kafka и обеспечивает их согласованное использование между продюсерами и консьюмерами.

- Поддерживает форматы: Avro, Protobuf, JSON Schema
- Позволяет централизованно управлять эволюцией схем
- Гарантирует совместимость данных между сервисами

## 2. Зачем нужен Schema Registry <a name="зачем"></a>

- **Безопасная эволюция схем**: можно менять структуру сообщений без потери совместимости
- **Валидация данных**: сообщения, не соответствующие схеме, не попадут в Kafka
- **Меньший размер сообщений**: вместо передачи полной схемы — только id схемы
- **Интеграция с Kafka Connect, Streams, REST Proxy**

## 3. Архитектура и компоненты <a name="архитектура"></a>

- **Schema Registry Server**: отдельный сервис (обычно на Java)
- **Kafka**: для хранения схем используется специальный топик (`_schemas`)
- **Клиенты**: producer/consumer, Kafka Connect, Streams, REST Proxy
- **REST API**: для регистрации, получения и управления схемами

## 4. Установка и запуск <a name="установка"></a>

### 1. Docker Compose

```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka:9092
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
```

### 2. Проверка запуска

```bash
curl http://localhost:8081/subjects
```

## 5. Работа с Avro, Protobuf, JSON Schema <a name="форматы"></a>

### 1. Avro
- Описание схемы в формате JSON
- Пример:
```json
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "email", "type": "string"}
  ]
}
```

### 2. Protobuf
- Описание схемы в формате .proto
- Пример:
```proto
syntax = "proto3";
message User {
  string id = 1;
  string email = 2;
}
```

### 3. JSON Schema
- Описание схемы в формате JSON Schema
- Пример:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "User",
  "type": "object",
  "properties": {
    "id": {"type": "string"},
    "email": {"type": "string"}
  },
  "required": ["id", "email"]
}
```

## 6. Интеграция с Kafka Producers/Consumers <a name="интеграция"></a>

### 1. Maven зависимости
```xml
<dependency>
  <groupId>io.confluent</groupId>
  <artifactId>kafka-avro-serializer</artifactId>
  <version>7.4.0</version>
</dependency>
```

### 2. Producer с Avro
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("value.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("schema.registry.url", "http://localhost:8081");

GenericRecord user = new GenericData.Record(schema);
user.put("id", "user-1");
user.put("email", "user@example.com");

ProducerRecord<String, GenericRecord> record = new ProducerRecord<>("users", user);
producer.send(record);
```

### 3. Consumer с Avro
```java
props.put("key.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("value.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("schema.registry.url", "http://localhost:8081");
props.put("specific.avro.reader", "true");
```

## 7. Эволюция схем и совместимость <a name="эволюция"></a>

### Типы совместимости:
- **BACKWARD**: новые схемы могут читать старые данные
- **FORWARD**: старые схемы могут читать новые данные
- **FULL**: обе стороны совместимы
- **NONE**: любая схема разрешена

### Примеры эволюции:
- Добавление нового поля с default-значением — совместимо
- Удаление обязательного поля — несовместимо

### Управление через REST API:
```bash
# Получить текущий режим совместимости
curl http://localhost:8081/config/users-value

# Установить режим совместимости
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "BACKWARD"}' \
  http://localhost:8081/config/users-value
```

## 8. REST API Schema Registry <a name="rest-api"></a>

- **POST /subjects/{subject}/versions** — регистрация новой схемы
- **GET /subjects** — список всех subjects
- **GET /subjects/{subject}/versions/latest** — получить последнюю версию схемы
- **GET /schemas/ids/{id}** — получить схему по id
- **DELETE /subjects/{subject}** — удалить все версии схемы

Пример регистрации схемы:
```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{...}"}' \
  http://localhost:8081/subjects/users-value/versions
```

## 9. Best practices <a name="best-practices"></a>

- Всегда используйте Schema Registry для всех критичных топиков
- Настраивайте совместимость схем (лучше всего — BACKWARD или FULL)
- Документируйте схемы и их эволюцию
- Используйте Avro/Protobuf для строгой типизации
- Не используйте NONE-совместимость в production
- Тестируйте эволюцию схем на тестовых данных
- Используйте versioning в названиях полей при необходимости

## 10. Вопросы для собеседования <a name="вопросы"></a>

### Базовые вопросы
1. **Что такое Schema Registry и зачем он нужен?**
2. **Какие форматы поддерживает Schema Registry?**
3. **Что такое subject в Schema Registry?**
4. **Как работает эволюция схем?**
5. **Какие типы совместимости схем бывают?**

### Продвинутые вопросы
6. **Как интегрировать Schema Registry с Kafka Connect?**
7. **Как работает сериализация и десериализация Avro/Protobuf/JSON Schema?**
8. **Как управлять схемами через REST API?**
9. **Как обеспечить совместимость схем при CI/CD?**
10. **Как мигрировать топики с одной схемой на другую?**

---

**Дополнительные ресурсы:**
- [Confluent Schema Registry Docs](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Avro Schema Evolution](https://avro.apache.org/docs/current/spec.html#Schema+Resolution)
- [Protobuf Schema Evolution](https://developers.google.com/protocol-buffers/docs/proto3#updating)
- [JSON Schema](https://json-schema.org/) 