# Kafka Connect: интеграция с внешними системами

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

**Kafka Connect** — это инструмент для масштабируемой и надёжной передачи данных между Apache Kafka и другими системами. Он позволяет легко интегрировать Kafka с базами данных, файловыми системами, облачными сервисами и другими источниками данных.

### Основные преимущества:
- **Масштабируемость**: горизонтальное масштабирование через распределённую архитектуру
- **Надёжность**: автоматическое восстановление после сбоев
- **Простота**: декларативная конфигурация через REST API
- **Экосистема**: множество готовых коннекторов

## Архитектура

### Компоненты Kafka Connect

#### 1. Connect Workers
```bash
# Запуск standalone worker
bin/connect-standalone.sh config/connect-standalone.properties \
  config/connect-file-source.properties \
  config/connect-file-sink.properties

# Запуск distributed worker
bin/connect-distributed.sh config/connect-distributed.properties
```

#### 2. Connectors
- **Source Connectors**: читают данные из внешних систем в Kafka
- **Sink Connectors**: записывают данные из Kafka во внешние системы

#### 3. Tasks
- Каждый коннектор может создавать несколько задач
- Задачи выполняются параллельно для масштабирования

#### 4. Converters
- Преобразуют данные между форматами (JSON, Avro, Protobuf)
- Обеспечивают совместимость схем

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
# Создание коннектора через REST API
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
# config/connect-jdbc-source.properties
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
# Создание JDBC коннектора
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

### 3. Debezium Source Connector (CDC)

```properties
# config/connect-debezium-source.properties
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
# Создание Debezium коннектора
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

## Sink Connectors

### 1. File Sink Connector

```properties
# config/connect-file-sink.properties
name=file-sink
connector.class=org.apache.kafka.connect.file.FileStreamSinkConnector
tasks.max=1
file=/tmp/test.sink.txt
topics=connect-test
```

```bash
# Создание file sink коннектора
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
# config/connect-jdbc-sink.properties
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
# Создание JDBC sink коннектора
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
# config/connect-elasticsearch-sink.properties
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
# Создание Elasticsearch sink коннектора
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
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log

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
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_DELETE_TOPIC_ENABLE: 'true'
    volumes:
      - kafka-data:/var/lib/kafka/data

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
      CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE: 'false'
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: 'false'
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
    volumes:
      - kafka-connect-data:/var/lib/kafka-connect

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
  kafka-connect-data:
```

### 4. Управление коннекторами через REST API

```bash
# Список коннекторов
curl -X GET http://localhost:8083/connectors

# Информация о коннекторе
curl -X GET http://localhost:8083/connectors/file-source

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
# Включение JMX метрик
JMX_PORT=9999
JMX_HOSTNAME=localhost
```

### 2. Prometheus метрики

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'kafka-connect'
    static_configs:
      - targets: ['kafka-connect:9999']
    metrics_path: /metrics
    scrape_interval: 5s
```

### 3. Grafana дашборды

```json
{
  "dashboard": {
    "title": "Kafka Connect Metrics",
    "panels": [
      {
        "title": "Records Processed",
        "type": "graph",
        "targets": [
          {
            "expr": "kafka_connect_sink_record_send_total"
          }
        ]
      },
      {
        "title": "Records Polled",
        "type": "graph",
        "targets": [
          {
            "expr": "kafka_connect_source_record_poll_total"
          }
        ]
      }
    ]
  }
}
```

### 4. Логирование

```properties
# Настройка логирования
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[%d] %p %m (%c)%n

# Логирование коннекторов
log4j.logger.org.apache.kafka.connect=INFO
log4j.logger.io.confluent.connect=INFO
```

## Лучшие практики

### 1. Конфигурация производительности

```properties
# Оптимизация для высокой пропускной способности
tasks.max=4
batch.size=1000
max.poll.records=500
poll.interval.ms=1000
```

### 2. Обработка ошибок

```properties
# Настройки retry
errors.retry.timeout=300000
errors.retry.delay.max.ms=60000
errors.tolerance=all
errors.log.enable=true
errors.log.include.messages=true
```

### 3. Схемы и конвертеры

```properties
# Использование Schema Registry
key.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=http://localhost:8081
value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=http://localhost:8081
```

### 4. Безопасность

```properties
# SSL настройки
security.protocol=SSL
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=password
ssl.keystore.location=/path/to/keystore.jks
ssl.keystore.password=password
ssl.key.password=password
```

### 5. Мониторинг и алертинг

```bash
#!/bin/bash
# Скрипт для проверки состояния коннекторов

CONNECT_URL="http://localhost:8083"
ALERT_EMAIL="admin@example.com"

# Проверка статуса коннекторов
connectors=$(curl -s "$CONNECT_URL/connectors" | jq -r '.[]')

for connector in $connectors; do
    status=$(curl -s "$CONNECT_URL/connectors/$connector/status" | jq -r '.connector.state')
    
    if [ "$status" != "RUNNING" ]; then
        echo "Connector $connector is in state: $status" | mail -s "Kafka Connect Alert" $ALERT_EMAIL
    fi
done
```

## Вопросы для собеседования

### Базовые вопросы
1. **Что такое Kafka Connect?**
   - Инструмент для интеграции Kafka с внешними системами
   - Поддерживает source и sink коннекторы
   - Обеспечивает масштабируемость и надёжность

2. **Какие режимы работы поддерживает Kafka Connect?**
   - Standalone mode: для разработки и тестирования
   - Distributed mode: для production с масштабированием

3. **Что такое Source и Sink коннекторы?**
   - Source: читают данные из внешних систем в Kafka
   - Sink: записывают данные из Kafka во внешние системы

### Продвинутые вопросы
4. **Как обеспечить exactly-once семантику в Kafka Connect?**
   - Использование транзакций
   - Идемпотентные операции
   - Правильная настройка offset management

5. **Как масштабировать Kafka Connect?**
   - Увеличение количества tasks
   - Добавление worker узлов
   - Партиционирование данных

6. **Как мониторить производительность Kafka Connect?**
   - JMX метрики
   - REST API для статуса
   - Логирование и алертинг

### Практические вопросы
7. **Как создать JDBC source коннектор?**
   ```bash
   curl -X POST -H "Content-Type: application/json" \
     --data '{
       "name": "jdbc-source",
       "config": {
         "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
         "connection.url": "jdbc:postgresql://localhost:5432/mydb",
         "table.whitelist": "users"
       }
     }' \
     http://localhost:8083/connectors
   ```

8. **Как обрабатывать ошибки в коннекторах?**
   - Настройка retry политик
   - Dead Letter Queue
   - Логирование ошибок

9. **Как обеспечить высокую доступность?**
   - Множественные worker узлы
   - Репликация топиков
   - Мониторинг и автоматическое восстановление

---

**Дополнительные ресурсы:**
- [Kafka Connect Documentation](https://kafka.apache.org/documentation/#connect)
- [Confluent Hub](https://www.confluent.io/hub/)
- [Debezium Documentation](https://debezium.io/documentation/) 