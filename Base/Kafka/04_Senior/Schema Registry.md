# Schema Registry в Kafka: подробный справочник

> [!QUOTE] Schema Registry — отдельный сервис для хранения и управления схемами сообщений Kafka. Продюсер регистрирует схему и передаёт в сообщении только её ID. Консьюмер получает схему по ID и десериализует данные. Это уменьшает размер сообщений и гарантирует совместимость между сервисами.

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

## 1. Что такое Schema Registry

**Schema Registry** — отдельный сервис (обычно Confluent Schema Registry), который хранит схемы (описания структуры сообщений) для топиков Kafka и обеспечивает их согласованное использование между продюсерами и консьюмерами.

- Поддерживает форматы: **Avro**, **Protobuf**, **JSON Schema**
- Позволяет централизованно управлять эволюцией схем
- Гарантирует совместимость данных между сервисами

## 2. Зачем нужен Schema Registry

- **Безопасная эволюция схем**: менять структуру сообщений без потери совместимости
- **Валидация данных**: сообщения, не соответствующие схеме, не попадут в Kafka
- **Меньший размер сообщений**: передаётся только ID схемы (4 байта), а не полная схема
- **Интеграция с Kafka Connect, Streams, REST Proxy**

> [!INFO] Формат wire: `[magic byte (0x0)] [schema-id (4 bytes)] [serialized data]`. Продюсер регистрирует схему → получает ID → отправляет ID + данные. Консьюмер читает ID → запрашивает схему из Registry → десериализует данные.

## 3. Архитектура и компоненты

- **Schema Registry Server**: отдельный Java-сервис
- **Kafka**: схемы хранятся во внутреннем топике `_schemas` (replicated)
- **Клиенты**: producer/consumer, Kafka Connect, Streams, REST Proxy
- **REST API**: для регистрации, получения и управления схемами

> [!WARNING] Schema Registry — single point of failure в стандартной конфигурации. Запускайте несколько инстансов за балансировщиком. При недоступности Registry новые продюсеры не смогут регистрировать схемы, существующие будут работать (схемы кешируются локально).

## 4. Установка и запуск

### Docker Compose

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

### Проверка запуска

```bash
# Список зарегистрированных subjects (пустой при первом запуске)
curl http://localhost:8081/subjects
```

## 5. Работа с Avro, Protobuf, JSON Schema

### 1. Avro

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

```proto
syntax = "proto3";
message User {
  string id = 1;
  string email = 2;
}
```

### 3. JSON Schema

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

> [!INFO] Avro — наиболее распространённый формат для Kafka. Компактный бинарный формат, строгая типизация, хорошая поддержка Schema Registry. Protobuf — альтернатива с лучшей совместимостью для полиглотных сред.

## 6. Интеграция с Kafka Producers/Consumers

### Maven зависимости

```xml
<dependency>
  <groupId>io.confluent</groupId>
  <artifactId>kafka-avro-serializer</artifactId>
  <version>7.4.0</version>
</dependency>
```

### Producer с Avro

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

### Consumer с Avro

```java
props.put("key.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("value.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("schema.registry.url", "http://localhost:8081");
props.put("specific.avro.reader", "true");
```

## 7. Эволюция схем и совместимость

### Типы совместимости

| Тип | Описание |
|-----|----------|
| **BACKWARD** | Новая схема может читать данные, записанные старой |
| **FORWARD** | Старая схема может читать данные, записанные новой |
| **FULL** | Совместимость в обе стороны |
| **NONE** | Любые изменения разрешены (без проверок) |

### Примеры эволюции

- Добавить новое поле с `default`-значением — совместимо (BACKWARD)
- Удалить поле — совместимо для FORWARD, несовместимо для BACKWARD
- Удалить обязательное поле (без default) — несовместимо

### Управление через REST API

```bash
# Получить текущий режим совместимости
curl http://localhost:8081/config/users-value

# Установить режим совместимости
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "BACKWARD"}' \
  http://localhost:8081/config/users-value
```

> [!WARNING] `NONE`-совместимость в production — опасно. Продюсер может записать данные с несовместимой схемой, и консьюмер не сможет их десериализовать. Используйте `BACKWARD` или `FULL` и тестируйте эволюцию схем в CI/CD.

## 8. REST API Schema Registry

| Метод | Эндпоинт | Назначение |
|-------|----------|------------|
| `POST` | `/subjects/{subject}/versions` | Зарегистрировать схему |
| `GET` | `/subjects` | Список всех subjects |
| `GET` | `/subjects/{subject}/versions/latest` | Последняя версия схемы |
| `GET` | `/schemas/ids/{id}` | Получить схему по ID |
| `DELETE` | `/subjects/{subject}` | Удалить все версии схемы |

### Пример регистрации схемы

```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"}]}"}' \
  http://localhost:8081/subjects/users-value/versions
```

## 9. Best practices

- Использовать Schema Registry для всех критичных топиков в production
- Настроить `BACKWARD` или `FULL` совместимость, не `NONE`
- Не удалять обязательные поля без default — это breaking change
- Тестировать эволюцию схем в CI/CD до деплоя
- Запускать несколько инстансов Schema Registry за балансировщиком
- Использовать именование subjects: `{topic-name}-key` и `{topic-name}-value`

## 10. Вопросы для собеседования

### Базовые вопросы
1. **Что такое Schema Registry и зачем он нужен?**
   - Централизованное хранилище схем для Kafka
   - Уменьшает размер сообщений (передаётся ID, не сама схема)
   - Обеспечивает валидацию и совместимость между продюсерами и консьюмерами

2. **Какие форматы поддерживает Schema Registry?**
   - Avro, Protobuf, JSON Schema

3. **Что такое subject в Schema Registry?**
   - Именованный контейнер для версий схемы
   - Обычно соответствует топику: `{topic}-value` или `{topic}-key`

4. **Как работает эволюция схем?**
   - Каждая новая версия схемы проверяется на совместимость с предыдущими
   - Совместимость настраивается через REST API (`BACKWARD`, `FORWARD`, `FULL`, `NONE`)

5. **Какие типы совместимости бывают?**
   - BACKWARD, FORWARD, FULL, NONE

### Продвинутые вопросы
6. **Как интегрировать Schema Registry с Kafka Connect?**
   - Установить `value.converter=AvroConverter` или `ProtobufConverter`
   - Указать `schema.registry.url` в конфигурации worker'а

7. **Как работает сериализация Avro?**
   - Продюсер регистрирует схему → получает ID
   - В сообщение записывается: `magic byte (0x0) + schema_id (4 bytes) + avro_data`
   - Консьюмер читает ID → запрашивает схему из Registry → десериализует

8. **Как управлять схемами через REST API?**
   - `POST /subjects/{subject}/versions` — зарегистрировать
   - `GET /subjects/{subject}/versions/latest` — получить последнюю
   - `PUT /config/{subject}` — изменить совместимость

9. **Как обеспечить совместимость схем при CI/CD?**
   - Использовать плагин `kafka-schema-registry-maven-plugin` для проверки перед деплоем
   - Автоматически тестировать совместимость в pipeline

10. **Как мигрировать топик с одной схемой на другую?**
    - Создать новый топик с новой схемой
    - Параллельно читать из старого и записывать в новый (с трансформацией)
    - Переключить консьюмеров на новый топик
    - Удалить старый топик после полной миграции
