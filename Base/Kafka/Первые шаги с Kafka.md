# Первые шаги с Apache Kafka: практическое руководство

## Оглавление
1. [Подготовка окружения](#подготовка)
2. [Создание топиков](#топики)
3. [Простой Producer](#producer)
4. [Простой Consumer](#consumer)
5. [Consumer Groups](#consumer-groups)
6. [Практические примеры](#примеры)
7. [Отладка и мониторинг](#отладка)
8. [Вопросы для собеседования](#вопросы)

## Подготовка окружения

### 1. Запуск Kafka через Docker

```bash
# Создание docker-compose.yml
cat > docker-compose.yml << EOF
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

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
EOF

# Запуск сервисов
docker-compose up -d

# Проверка статуса
docker-compose ps
```

### 2. Проверка работоспособности

```bash
# Проверка доступности Kafka
docker exec -it kafka kafka-broker-api-versions \
  --bootstrap-server localhost:9092

# Список топиков (изначально пустой)
docker exec -it kafka kafka-topics --list \
  --bootstrap-server localhost:9092
```

## Создание топиков

### 1. Базовое создание топика

```bash
# Создание простого топика
docker exec -it kafka kafka-topics --create \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Проверка созданного топика
docker exec -it kafka kafka-topics --describe \
  --topic my-first-topic \
  --bootstrap-server localhost:9092
```

### 2. Создание топика с настройками

```bash
# Создание топика с retention policy
docker exec -it kafka kafka-topics --create \
  --topic events-topic \
  --bootstrap-server localhost:9092 \
  --partitions 5 \
  --replication-factor 1 \
  --config retention.ms=86400000 \
  --config retention.bytes=1073741824

# Создание топика с cleanup policy
docker exec -it kafka kafka-topics --create \
  --topic compacted-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1 \
  --config cleanup.policy=compact \
  --config delete.retention.ms=86400000
```

### 3. Управление топиками

```bash
# Список всех топиков
docker exec -it kafka kafka-topics --list \
  --bootstrap-server localhost:9092

# Детальная информация о топике
docker exec -it kafka kafka-topics --describe \
  --topic my-first-topic \
  --bootstrap-server localhost:9092

# Увеличение количества партиций
docker exec -it kafka kafka-topics --alter \
  --topic my-first-topic \
  --partitions 6 \
  --bootstrap-server localhost:9092

# Удаление топика
docker exec -it kafka kafka-topics --delete \
  --topic my-first-topic \
  --bootstrap-server localhost:9092
```

## Простой Producer

### 1. Консольный Producer

```bash
# Запуск консольного producer
docker exec -it kafka kafka-console-producer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092

# В консоли можно вводить сообщения:
# {"user": "alice", "action": "login", "timestamp": "2024-01-01T10:00:00Z"}
# {"user": "bob", "action": "logout", "timestamp": "2024-01-01T10:05:00Z"}
```

### 2. Producer с ключами

```bash
# Producer с указанием ключей
docker exec -it kafka kafka-console-producer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Ввод сообщений с ключами:
# user-123:{"action": "login", "timestamp": "2024-01-01T10:00:00Z"}
# user-456:{"action": "logout", "timestamp": "2024-01-01T10:05:00Z"}
```

### 3. Producer с настройками

```bash
# Producer с настройками производительности
docker exec -it kafka kafka-console-producer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --property "acks=all" \
  --property "retries=3" \
  --property "batch.size=16384" \
  --property "linger.ms=1"
```

## Простой Consumer

### 1. Консольный Consumer

```bash
# Запуск консольного consumer
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --from-beginning

# Просмотр сообщений с метаданными
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --property "print.timestamp=true" \
  --property "print.key=true" \
  --property "print.value=true"
```

### 2. Consumer с настройками

```bash
# Consumer с настройками группы
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group \
  --property "auto.offset.reset=earliest" \
  --property "enable.auto.commit=true" \
  --property "auto.commit.interval.ms=1000"
```

### 3. Consumer для конкретной партиции

```bash
# Чтение из конкретной партиции
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --partition 0 \
  --offset 0
```

## Consumer Groups

### 1. Создание Consumer Group

```bash
# Запуск consumer в группе
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --group my-group \
  --from-beginning
```

### 2. Управление Consumer Groups

```bash
# Список consumer groups
docker exec -it kafka kafka-consumer-groups --list \
  --bootstrap-server localhost:9092

# Детальная информация о группе
docker exec -it kafka kafka-consumer-groups --describe \
  --group my-group \
  --bootstrap-server localhost:9092

# Сброс offset для группы
docker exec -it kafka kafka-consumer-groups --reset-offsets \
  --group my-group \
  --topic my-first-topic \
  --to-earliest \
  --execute \
  --bootstrap-server localhost:9092
```

### 3. Масштабирование Consumer Groups

```bash
# Запуск нескольких consumer'ов в одной группе
# Терминал 1:
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --group my-group

# Терминал 2:
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --group my-group

# Проверка распределения партиций
docker exec -it kafka kafka-consumer-groups --describe \
  --group my-group \
  --bootstrap-server localhost:9092
```

## Практические примеры

### 1. Система логирования

```bash
# Создание топика для логов
docker exec -it kafka kafka-topics --create \
  --topic application-logs \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Producer для логов
docker exec -it kafka kafka-console-producer \
  --topic application-logs \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Ввод логов:
# ERROR:{"level": "ERROR", "message": "Database connection failed", "timestamp": "2024-01-01T10:00:00Z"}
# INFO:{"level": "INFO", "message": "User logged in", "timestamp": "2024-01-01T10:01:00Z"}
# WARN:{"level": "WARN", "message": "High memory usage", "timestamp": "2024-01-01T10:02:00Z"}

# Consumer для обработки логов
docker exec -it kafka kafka-console-consumer \
  --topic application-logs \
  --bootstrap-server localhost:9092 \
  --group log-processor \
  --property "print.key=true" \
  --property "print.value=true"
```

### 2. Система событий пользователей

```bash
# Создание топика для событий
docker exec -it kafka kafka-topics --create \
  --topic user-events \
  --bootstrap-server localhost:9092 \
  --partitions 5 \
  --replication-factor 1

# Producer для событий
docker exec -it kafka kafka-console-producer \
  --topic user-events \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Ввод событий:
# user-123:{"event": "login", "timestamp": "2024-01-01T10:00:00Z", "ip": "192.168.1.1"}
# user-456:{"event": "purchase", "timestamp": "2024-01-01T10:05:00Z", "amount": 99.99}
# user-123:{"event": "logout", "timestamp": "2024-01-01T10:10:00Z", "session_duration": 600}

# Consumer для аналитики
docker exec -it kafka kafka-console-consumer \
  --topic user-events \
  --bootstrap-server localhost:9092 \
  --group analytics-processor \
  --property "print.key=true" \
  --property "print.value=true"
```

### 3. Система уведомлений

```bash
# Создание топика для уведомлений
docker exec -it kafka kafka-topics --create \
  --topic notifications \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Producer для уведомлений
docker exec -it kafka kafka-console-producer \
  --topic notifications \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Ввод уведомлений:
# email:{"type": "email", "to": "user@example.com", "subject": "Welcome", "body": "Welcome to our service"}
# sms:{"type": "sms", "to": "+1234567890", "message": "Your order has been shipped"}
# push:{"type": "push", "device_id": "device-123", "title": "New message", "body": "You have a new message"}

# Consumer для отправки уведомлений
docker exec -it kafka kafka-console-consumer \
  --topic notifications \
  --bootstrap-server localhost:9092 \
  --group notification-sender \
  --property "print.key=true" \
  --property "print.value=true"
```

## Отладка и мониторинг

### 1. Проверка состояния топиков

```bash
# Детальная информация о топике
docker exec -it kafka kafka-topics --describe \
  --topic my-first-topic \
  --bootstrap-server localhost:9092

# Проверка under-replicated партиций
docker exec -it kafka kafka-topics --describe \
  --under-replicated-partitions \
  --bootstrap-server localhost:9092

# Проверка недоступных партиций
docker exec -it kafka kafka-topics --describe \
  --unavailable-partitions \
  --bootstrap-server localhost:9092
```

### 2. Мониторинг Consumer Groups

```bash
# Список всех групп
docker exec -it kafka kafka-consumer-groups --list \
  --bootstrap-server localhost:9092

# Детальная информация о группе
docker exec -it kafka kafka-consumer-groups --describe \
  --group my-group \
  --bootstrap-server localhost:9092

# Проверка lag (отставания)
docker exec -it kafka kafka-consumer-groups --describe \
  --group my-group \
  --bootstrap-server localhost:9092 \
  --members
```

### 3. Тестирование производительности

```bash
# Тест producer
docker exec -it kafka kafka-producer-perf-test \
  --topic performance-test \
  --num-records 100000 \
  --record-size 1000 \
  --throughput 10000 \
  --bootstrap-server localhost:9092

# Тест consumer
docker exec -it kafka kafka-consumer-perf-test \
  --topic performance-test \
  --bootstrap-server localhost:9092 \
  --messages 100000
```

### 4. Просмотр логов

```bash
# Логи Kafka
docker logs kafka

# Логи ZooKeeper
docker logs zookeeper

# Логи в реальном времени
docker logs -f kafka
```

## Вопросы для собеседования

### Базовые вопросы
1. **Как создать топик в Kafka?**
   ```bash
   kafka-topics --create --topic my-topic \
     --bootstrap-server localhost:9092 \
     --partitions 3 --replication-factor 1
   ```

2. **Что такое consumer group?**
   - Группа consumer'ов, которые совместно обрабатывают сообщения
   - Обеспечивает масштабируемость и отказоустойчивость
   - Каждая партиция обрабатывается только одним consumer'ом в группе

3. **Как работает партиционирование?**
   - Сообщения распределяются по партициям на основе ключа
   - Сообщения с одинаковым ключом попадают в одну партицию
   - Обеспечивает параллельную обработку

### Продвинутые вопросы
4. **Как обеспечить порядок сообщений?**
   - Использование ключей для партиционирования
   - Одна партиция = гарантированный порядок
   - Consumer group с одним consumer'ом

5. **Что такое offset и как он работает?**
   - Уникальный идентификатор сообщения в партиции
   - Consumer отслеживает свой прогресс через offset
   - Автоматический или ручной commit

6. **Как масштабировать consumer'ы?**
   - Добавление consumer'ов в группу
   - Автоматическое перераспределение партиций
   - Количество consumer'ов не должно превышать количество партиций

### Практические вопросы
7. **Как отладить проблемы с consumer group?**
   ```bash
   kafka-consumer-groups --describe --group my-group \
     --bootstrap-server localhost:9092
   ```

8. **Как проверить производительность Kafka?**
   ```bash
   kafka-producer-perf-test --topic test-topic \
     --num-records 100000 --record-size 1000 \
     --bootstrap-server localhost:9092
   ```

9. **Как настроить retention policy?**
   - По времени: retention.ms
   - По размеру: retention.bytes
   - Комбинированный подход

---

**Дополнительные ресурсы:**
- [Kafka Quick Start](https://kafka.apache.org/quickstart)
- [Kafka Console Producer/Consumer](https://kafka.apache.org/documentation/#basic_ops_console_producer_consumer)
- [Kafka Consumer Groups](https://kafka.apache.org/documentation/#consumerconfigs) 