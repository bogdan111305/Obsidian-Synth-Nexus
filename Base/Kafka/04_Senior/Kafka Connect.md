# Kafka Connect: интеграция с внешними системами

> [!QUOTE] Kafka Connect — фреймворк для масштабируемой передачи данных между Kafka и внешними системами. Управляется через REST API. Source Connectors читают данные во Kafka, Sink Connectors — записывают данные из Kafka.

## Оглавление
1. [Введение в Kafka Connect](#введение)
2. [Архитектура](#архитектура)
3. [Source Connectors](#source)
4. [Sink Connectors](#sink)
5. [Настройка и развёртывание](#настройка)
6. [Мониторинг](#мониторинг)
7. [Лучшие практики](#практики)
8. [Вопросы для собеседования](#вопросы)

## Введение в Kafka Connect

**Kafka Connect** — инструмент для надёжной передачи данных между Kafka и внешними системами (БД, файловые системы, облачные сервисы). Работает на декларативной конфигурации через REST API — код писать не нужно.

### Ключевые свойства:
- **Масштабируемость**: горизонтальное масштабирование через распределённую архитектуру (distributed mode)
- **Надёжность**: автоматическое восстановление задач после сбоев
- **Экосистема**: сотни готовых коннекторов на Confluent Hub

## Архитектура

### Компоненты

- **Connect Worker** — процесс, запускающий коннекторы и задачи
- **Connector** — конфигурация интеграции (Source или Sink)
- **Task** — единица параллелизма; один коннектор создаёт несколько задач
- **Converter** — преобразует данные между форматами (JSON, Avro, Protobuf)

```bash
# Запуск standalone worker
bin/connect-standalone.sh config/connect-standalone.properties \
  config/connect-file-source.properties

# Запуск distributed worker
bin/connect-distributed.sh config/connect-distributed.properties
```

> [!INFO] **Standalone mode** — для разработки и тестирования. **Distributed mode** — для production: несколько worker'ов, автоматическое перераспределение задач при сбое.

## Source Connectors

### 1. File Source Connector

```properties
# config/connect-file-source.properties
name=file-source
connector.class=org.apache.kafka.connect.file.FileStreamSourceConnector
tasks.max=1
file=/tmp/test.txt
topic=connect-test
```

```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{
    "name": "file-source",
    "config": {
      "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
      "tasks.max": "1",
      "file": "/tmp/test.txt",
      "topic": "connect-test"
    }
  }' \
  http://localhost:8083/connectors
```

### 2. JDBC Source Connector

```properties
name=jdbc-source
connector.class=io.confluent.connect.jdbc.JdbcSourceConnector
tasks.max=1
connection.url=jdbc:postgresql://localhost:5432/mydb
connection.user=postgres
connection.password=password
topic.prefix=jdbc-
table.whitelist=users,orders
mode=incrementing
incrementing.column.name=id
```

```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{
    "name": "jdbc-source",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
      "tasks.max": "1",
      "connection.url": "jdbc:postgresql://localhost:5432/mydb",
      "connection.user": "postgres",
      "connection.password": "password",
      "topic.prefix": "jdbc-",
      "table.whitelist": "users,orders",
      "mode": "incrementing",
      "incrementing.column.name": "id"
    }
  }' \
  http://localhost:8083/connectors
```

> [!INFO] JDBC Source поддерживает несколько режимов отслеживания изменений:
> - `incrementing` — по возрастающему числовому id
> - `timestamp` — по полю timestamp
> - `timestamp+incrementing` — комбинация обоих
> - `bulk` — полная перезагрузка таблицы при каждом опросе

### 3. Debezium Source Connector (CDC)

```properties
name=debezium-source
connector.class=io.debezium.connector.postgresql.PostgresConnector
tasks.max=1
database.hostname=localhost
database.port=5432
database.user=postgres
database.password=password
database.dbname=mydb
database.server.name=my-server
table.include.list=public.users,public.orders
```

```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{
    "name": "debezium-source",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "tasks.max": "1",
      "database.hostname": "localhost",
      "database.port": "5432",
      "database.user": "postgres",
      "database.password": "password",
      "database.dbname": "mydb",
      "database.server.name": "my-server",
      "table.include.list": "public.users,public.orders"
    }
  }' \
  http://localhost:8083/connectors
```

> [!INFO] Debezium использует механизм Change Data Capture (CDC) — читает WAL (Write-Ahead Log) PostgreSQL. Это позволяет захватывать все операции INSERT/UPDATE/DELETE в реальном времени без дополнительной нагрузки на БД.

## Sink Connectors

### 1. File Sink Connector

```properties
name=file-sink
connector.class=org.apache.kafka.connect.file.FileStreamSinkConnector
tasks.max=1
file=/tmp/test.sink.txt
topics=connect-test
```

```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{
    "name": "file-sink",
    "config": {
      "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
      "tasks.max": "1",
      "file": "/tmp/test.sink.txt",
      "topics": "connect-test"
    }
  }' \
  http://localhost:8083/connectors
```

### 2. JDBC Sink Connector

```properties
name=jdbc-sink
connector.class=io.confluent.connect.jdbc.JdbcSinkConnector
tasks.max=1
connection.url=jdbc:postgresql://localhost:5432/mydb
connection.user=postgres
connection.password=password
topics=users,orders
auto.create=true
auto.evolve=true
insert.mode=upsert
pk.mode=record_key
```

```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{
    "name": "jdbc-sink",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      "tasks.max": "1",
      "connection.url": "jdbc:postgresql://localhost:5432/mydb",
      "connection.user": "postgres",
      "connection.password": "password",
      "topics": "users,orders",
      "auto.create": "true",
      "auto.evolve": "true",
      "insert.mode": "upsert",
      "pk.mode": "record_key"
    }
  }' \
  http://localhost:8083/connectors
```

### 3. Elasticsearch Sink Connector

```properties
name=elasticsearch-sink
connector.class=io.confluent.connect.elasticsearch.ElasticsearchSinkConnector
tasks.max=1
connection.url=http://localhost:9200
type.name=kafka-connect
topics=users,orders
key.ignore=true
schema.ignore=true
```

```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{
    "name": "elasticsearch-sink",
    "config": {
      "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
      "tasks.max": "1",
      "connection.url": "http://localhost:9200",
      "type.name": "kafka-connect",
      "topics": "users,orders",
      "key.ignore": "true",
      "schema.ignore": "true"
    }
  }' \
  http://localhost:8083/connectors
```

## Настройка и развёртывание

### 1. Standalone Mode

```properties
# config/connect-standalone.properties
bootstrap.servers=localhost:9092
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.storage.StringConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false
offset.storage.file.filename=/tmp/connect.offsets
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
config.storage.replication.factor=1
offset.storage.replication.factor=1
status.storage.replication.factor=1
```

### 2. Distributed Mode

```properties
# config/connect-distributed.properties
bootstrap.servers=localhost:9092
group.id=connect-cluster
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.storage.StringConverter
key.converter.schemas.enable=false
value.converter.schemas.enable=false
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
config.storage.replication.factor=1
offset.storage.replication.factor=1
status.storage.replication.factor=1
rest.port=8083
rest.host.name=localhost
rest.advertised.host.name=localhost
rest.advertised.port=8083
```

### 3. Docker Compose для Kafka Connect

```yaml
# docker-compose-connect.yml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.4.0
    hostname: kafka-connect
    container_name: kafka-connect
    depends_on:
      - kafka
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: 'kafka:29092'
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: connect-cluster
      CONNECT_CONFIG_STORAGE_TOPIC: connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: connect-status
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
```

### 4. Управление через REST API

```bash
# Список коннекторов
curl -X GET http://localhost:8083/connectors

# Статус коннектора
curl -X GET http://localhost:8083/connectors/file-source/status

# Конфигурация коннектора
curl -X GET http://localhost:8083/connectors/file-source/config

# Обновление конфигурации
curl -X PUT -H "Content-Type: application/json" \
  --data '{
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "2",
    "file": "/tmp/test.txt",
    "topic": "connect-test"
  }' \
  http://localhost:8083/connectors/file-source/config

# Удаление коннектора
curl -X DELETE http://localhost:8083/connectors/file-source

# Перезапуск коннектора
curl -X POST http://localhost:8083/connectors/file-source/restart

# Перезапуск задачи
curl -X POST http://localhost:8083/connectors/file-source/tasks/0/restart
```

## Мониторинг

### 1. JMX метрики

```properties
# Включение JMX
JMX_PORT=9999
JMX_HOSTNAME=localhost
```

### 2. Prometheus

```yaml
scrape_configs:
  - job_name: 'kafka-connect'
    static_configs:
      - targets: ['kafka-connect:9999']
    metrics_path: /metrics
    scrape_interval: 5s
```

### 3. Ключевые метрики

```json
{
  "dashboard": {
    "title": "Kafka Connect Metrics",
    "panels": [
      { "expr": "kafka_connect_sink_record_send_total" },
      { "expr": "kafka_connect_source_record_poll_total" }
    ]
  }
}
```

### 4. Скрипт мониторинга коннекторов

```bash
#!/bin/bash
CONNECT_URL="http://localhost:8083"

connectors=$(curl -s "$CONNECT_URL/connectors" | jq -r '.[]')

for connector in $connectors; do
    status=$(curl -s "$CONNECT_URL/connectors/$connector/status" | jq -r '.connector.state')
    if [ "$status" != "RUNNING" ]; then
        echo "ALERT: Connector $connector is in state: $status"
    fi
done
```

## Лучшие практики

### 1. Производительность

```properties
tasks.max=4
batch.size=1000
max.poll.records=500
poll.interval.ms=1000
```

### 2. Обработка ошибок

```properties
errors.retry.timeout=300000
errors.retry.delay.max.ms=60000
errors.tolerance=all
errors.log.enable=true
errors.log.include.messages=true
```

> [!WARNING] `errors.tolerance=all` — коннектор будет продолжать работу при ошибках обработки записей. Без этой настройки (`errors.tolerance=none`) коннектор останавливается при первой ошибке. Всегда включайте `errors.log.enable=true` и настройте DLQ (`errors.deadletterqueue.topic.name`), чтобы не терять данные.

### 3. Schema Registry интеграция

```properties
key.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://localhost:8081
value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=http://localhost:8081
```

### 4. Безопасность

```properties
security.protocol=SSL
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=password
ssl.keystore.location=/path/to/keystore.jks
ssl.keystore.password=password
ssl.key.password=password
```

## Вопросы для собеседования

### Базовые вопросы
1. **Что такое Kafka Connect?**
   - Фреймворк для интеграции Kafka с внешними системами без написания кода
   - Управляется через REST API
   - Source — данные в Kafka, Sink — данные из Kafka

2. **Какие режимы работы?**
   - Standalone: для разработки и тестирования, один процесс
   - Distributed: для production, несколько worker'ов, auto-failover

3. **Что такое Source и Sink коннекторы?**
   - Source: читают данные из внешних систем → Kafka
   - Sink: записывают данные из Kafka → внешние системы

### Продвинутые вопросы
4. **Как обеспечить exactly-once в Kafka Connect?**
   - Source: идемпотентные операции или транзакции
   - Sink: `insert.mode=upsert` + `pk.mode=record_key` для JDBC
   - Правильная настройка offset management

5. **Как масштабировать Kafka Connect?**
   - Увеличить `tasks.max` в конфигурации коннектора
   - Добавить worker'ы в distributed режиме
   - Задачи автоматически перераспределятся

6. **Как мониторить?**
   - REST API: `GET /connectors/{name}/status`
   - JMX метрики → Prometheus → Grafana
   - Проверка состояния `RUNNING` / `FAILED` / `PAUSED`

### Практические вопросы
7. **Как создать JDBC source коннектор?**
   ```bash
   curl -X POST -H "Content-Type: application/json" \
     --data '{
       "name": "jdbc-source",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
         "connection.url": "jdbc:postgresql://localhost:5432/mydb",
         "table.whitelist": "users",
         "mode": "incrementing",
         "incrementing.column.name": "id"
       }
     }' \
     http://localhost:8083/connectors
   ```

8. **Как обрабатывать ошибки?**
   - `errors.tolerance=all` — не останавливать коннектор при ошибках
   - `errors.deadletterqueue.topic.name` — DLQ для проблемных записей
   - `errors.log.enable=true` — логировать ошибки

9. **Как обеспечить высокую доступность?**
   - Distributed mode с несколькими worker'ами
   - Репликация internal topics (`connect-configs`, `connect-offsets`, `connect-status`)
   - Мониторинг состояния через REST API
