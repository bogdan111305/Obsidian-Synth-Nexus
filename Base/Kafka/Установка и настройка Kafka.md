# Установка и настройка Apache Kafka

## Оглавление
1. [Способы установки](#способы-установки)
2. [Установка через Docker](#docker)
3. [Установка standalone](#standalone)
4. [Настройка кластера](#кластер)
5. [Конфигурация брокера](#конфигурация)
6. [Безопасность](#безопасность)
7. [Мониторинг](#мониторинг)
8. [Troubleshooting](#troubleshooting)
9. [Вопросы для собеседования](#вопросы)

## Способы установки

### Доступные варианты:
1. **Docker** - самый простой способ для разработки
2. **Standalone** - установка на одном сервере
3. **Кластер** - production-ready установка
4. **Managed services** - AWS MSK, Confluent Cloud, Azure Event Hubs

## Установка через Docker

### 1. Простая установка с Docker Compose

```yaml
# docker-compose.yml
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
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_DELETE_TOPIC_ENABLE: 'true'
    volumes:
      - kafka-data:/var/lib/kafka/data

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
```

### 2. Запуск и проверка

```bash
# Запуск всех сервисов
docker-compose up -d

# Проверка статуса
docker-compose ps

# Просмотр логов
docker-compose logs kafka

# Создание тестового топика
docker exec -it kafka kafka-topics --create \
  --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Проверка топиков
docker exec -it kafka kafka-topics --list \
  --bootstrap-server localhost:9092
```

### 3. Kafka без ZooKeeper (KRaft mode)

```yaml
# docker-compose-kraft.yml
version: '3.8'
services:
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka
    container_name: kafka
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://0.0.0.0:9092,CONTROLLER://kafka:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_DELETE_TOPIC_ENABLE: 'true'
      CLUSTER_ID: '4L6g3nShT-eMCtK--X86sw'
    volumes:
      - kafka-data:/var/lib/kafka/data
    command: 
      - bash
      - -c
      - |
        echo "Formatting kafka storage..."
        kafka-storage format -t $CLUSTER_ID -c /etc/kafka/kafka.properties
        echo "Starting kafka..."
        /etc/confluent/docker/run

volumes:
  kafka-data:
```

## Установка standalone

### 1. Скачивание и распаковка

```bash
# Скачивание Kafka
wget https://downloads.apache.org/kafka/3.5.1/kafka_2.13-3.5.1.tgz

# Распаковка
tar -xzf kafka_2.13-3.5.1.tgz
cd kafka_2.13-3.5.1

# Создание директорий для логов
mkdir -p /tmp/kafka-logs
mkdir -p /tmp/zookeeper
```

### 2. Настройка ZooKeeper

```properties
# config/zookeeper.properties
dataDir=/tmp/zookeeper
clientPort=2181
maxClientCnxns=0
admin.enableServer=false
```

### 3. Настройка Kafka

```properties
# config/server.properties
# Основные настройки
broker.id=0
listeners=PLAINTEXT://localhost:9092
log.dirs=/tmp/kafka-logs
num.partitions=1
default.replication.factor=1

# Настройки производительности
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Настройки логов
log.segment.bytes=1073741824
log.retention.hours=168
log.retention.check.interval.ms=300000

# Настройки очистки
log.cleaner.enable=true
log.cleanup.policy=delete

# Настройки безопасности
delete.topic.enable=true
auto.create.topics.enable=true
```

### 4. Запуск сервисов

```bash
# Запуск ZooKeeper
bin/zookeeper-server-start.sh config/zookeeper.properties &

# Запуск Kafka
bin/kafka-server-start.sh config/server.properties &

# Проверка статуса
jps | grep -E "(Kafka|QuorumPeerMain)"

# Создание топика
bin/kafka-topics.sh --create \
  --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Проверка топиков
bin/kafka-topics.sh --list \
  --bootstrap-server localhost:9092
```

## Настройка кластера

### 1. Архитектура кластера

```bash
# Минимальная конфигурация для production:
# - 3 брокера Kafka
# - 3 узла ZooKeeper (или KRaft mode)
# - Отдельные серверы для мониторинга
```

### 2. Конфигурация для каждого брокера

```properties
# server-1.properties
broker.id=1
listeners=PLAINTEXT://kafka-1:9092
log.dirs=/var/lib/kafka/data
num.partitions=3
default.replication.factor=3
min.insync.replicas=2

# Настройки репликации
replica.lag.time.max.ms=10000
replica.lag.max.messages=4000

# Настройки производительности
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Настройки логов
log.segment.bytes=1073741824
log.retention.hours=168
log.retention.bytes=1073741824

# Настройки безопасности
delete.topic.enable=true
auto.create.topics.enable=false
```

### 3. Настройка ZooKeeper кластера

```properties
# zookeeper-1.properties
dataDir=/var/lib/zookeeper/data
clientPort=2181
initLimit=5
syncLimit=2
server.1=kafka-1:2888:3888
server.2=kafka-2:2888:3888
server.3=kafka-3:2888:3888
```

### 4. Скрипты для управления кластером

```bash
#!/bin/bash
# start-cluster.sh

echo "Starting ZooKeeper cluster..."
ssh kafka-1 "cd /opt/kafka && bin/zookeeper-server-start.sh config/zookeeper.properties &"
ssh kafka-2 "cd /opt/kafka && bin/zookeeper-server-start.sh config/zookeeper.properties &"
ssh kafka-3 "cd /opt/kafka && bin/zookeeper-server-start.sh config/zookeeper.properties &"

sleep 10

echo "Starting Kafka cluster..."
ssh kafka-1 "cd /opt/kafka && bin/kafka-server-start.sh config/server-1.properties &"
ssh kafka-2 "cd /opt/kafka && bin/kafka-server-start.sh config/server-2.properties &"
ssh kafka-3 "cd /opt/kafka && bin/kafka-server-start.sh config/server-3.properties &"

echo "Cluster started successfully!"
```

## Конфигурация брокера

### 1. Основные настройки

```properties
# Идентификация брокера
broker.id=1

# Сетевые настройки
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://kafka-1:9092

# Директории для логов
log.dirs=/var/lib/kafka/data

# Настройки партиций
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
```

### 2. Настройки производительности

```properties
# Сетевые потоки
num.network.threads=3
num.io.threads=8

# Буферы
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Настройки логов
log.segment.bytes=1073741824
log.retention.hours=168
log.retention.bytes=1073741824

# Настройки очистки
log.cleaner.enable=true
log.cleanup.policy=delete
log.cleaner.threads=2
```

### 3. Настройки репликации

```properties
# Репликация
replica.lag.time.max.ms=10000
replica.lag.max.messages=4000
replica.fetch.max.bytes=1048576
replica.fetch.wait.max.ms=500

# Настройки лидера
leader.imbalance.check.interval.seconds=300
leader.imbalance.per.broker.percentage=10
```

## Безопасность

### 1. SSL/TLS настройка

```properties
# SSL настройки
ssl.keystore.location=/path/to/kafka.server.keystore.jks
ssl.keystore.password=password
ssl.key.password=password
ssl.truststore.location=/path/to/kafka.server.truststore.jks
ssl.truststore.password=password
ssl.client.auth=required

# Слушатели
listeners=SSL://0.0.0.0:9093
advertised.listeners=SSL://kafka-1:9093
```

### 2. SASL настройка

```properties
# SASL настройки
sasl.enabled.mechanisms=PLAIN
sasl.mechanism.inter.broker.protocol=PLAIN
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
allow.everyone.if.no.acl.found=false

# Слушатели
listeners=SASL_PLAINTEXT://0.0.0.0:9092
advertised.listeners=SASL_PLAINTEXT://kafka-1:9092
```

### 3. ACL настройка

```bash
# Создание пользователя
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:alice \
  --operation Read \
  --topic test-topic

# Создание группы
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:alice \
  --operation Read \
  --group my-group
```

## Мониторинг

### 1. JMX метрики

```properties
# Включение JMX
JMX_PORT=9999
JMX_HOSTNAME=localhost
```

### 2. Prometheus метрики

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-1:9999', 'kafka-2:9999', 'kafka-3:9999']
    metrics_path: /metrics
    scrape_interval: 5s
```

### 3. Grafana дашборды

```json
{
  "dashboard": {
    "title": "Kafka Cluster Metrics",
    "panels": [
      {
        "title": "Messages per second",
        "type": "graph",
        "targets": [
          {
            "expr": "kafka_server_brokertopicmetrics_messagesinpersec_rate"
          }
        ]
      }
    ]
  }
}
```

## Troubleshooting

### 1. Частые проблемы

```bash
# Проверка статуса брокеров
kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# Проверка топиков
kafka-topics.sh --describe --topic test-topic --bootstrap-server localhost:9092

# Проверка consumer groups
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Проверка логов
tail -f /var/log/kafka/server.log
```

### 2. Диагностика производительности

```bash
# Тест производительности producer
kafka-producer-perf-test.sh \
  --topic test-topic \
  --num-records 1000000 \
  --record-size 1000 \
  --throughput 10000 \
  --bootstrap-server localhost:9092

# Тест производительности consumer
kafka-consumer-perf-test.sh \
  --topic test-topic \
  --bootstrap-server localhost:9092 \
  --messages 1000000
```

### 3. Восстановление после сбоев

```bash
# Перебалансировка партиций
kafka-leader-election.sh \
  --bootstrap-server localhost:9092 \
  --election-type PREFERRED \
  --all-topic-partitions

# Проверка репликации
kafka-topics.sh --describe --under-replicated-partitions \
  --bootstrap-server localhost:9092
```

## Вопросы для собеседования

### Базовые вопросы
1. **Какие способы установки Kafka вы знаете?**
   - Docker, standalone, кластер, managed services
   - Каждый имеет свои преимущества и недостатки

2. **Что такое KRaft mode?**
   - Режим работы Kafka без ZooKeeper
   - Использует встроенный consensus protocol
   - Доступен с версии 2.8

3. **Как настроить высокую доступность?**
   - Минимум 3 брокера
   - Replication factor = 3
   - Настройка min.insync.replicas

### Продвинутые вопросы
4. **Как настроить безопасность в Kafka?**
   - SSL/TLS для шифрования
   - SASL для аутентификации
   - ACL для авторизации

5. **Как мониторить производительность?**
   - JMX метрики
   - Prometheus + Grafana
   - Кастомные дашборды

6. **Как диагностировать проблемы в кластере?**
   - Проверка логов
   - Анализ метрик
   - Использование CLI инструментов

### Практические вопросы
7. **Как настроить retention policy?**
   - По времени: log.retention.hours
   - По размеру: log.retention.bytes
   - Комбинированный подход

8. **Как оптимизировать производительность?**
   - Настройка буферов
   - Оптимизация количества партиций
   - Настройка batch size

9. **Как обеспечить отказоустойчивость?**
   - Правильная настройка репликации
   - Мониторинг in-sync replicas
   - Автоматическое восстановление

---

**Дополнительные ресурсы:**
- [Kafka Configuration](https://kafka.apache.org/documentation/#configuration)
- [Kafka Security](https://kafka.apache.org/documentation/#security)
- [Kafka Monitoring](https://kafka.apache.org/documentation/#monitoring) 