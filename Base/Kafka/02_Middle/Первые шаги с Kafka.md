# Первые шаги с Apache Kafka: практическое руководство

> [!QUOTE] Consumer Group — группа консьюмеров, которые совместно читают топик. Каждая партиция назначается ровно одному консьюмеру в группе. Это обеспечивает параллельную обработку и балансировку нагрузки.

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
```

```bash
docker-compose up -d
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
docker exec -it kafka kafka-topics --create \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Проверка
docker exec -it kafka kafka-topics --describe \
  --topic my-first-topic \
  --bootstrap-server localhost:9092
```

### 2. Создание топика с настройками

```bash
# С retention policy
docker exec -it kafka kafka-topics --create \
  --topic events-topic \
  --bootstrap-server localhost:9092 \
  --partitions 5 \
  --replication-factor 1 \
  --config retention.ms=86400000 \
  --config retention.bytes=1073741824

# С compact cleanup policy
docker exec -it kafka kafka-topics --create \
  --topic compacted-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1 \
  --config cleanup.policy=compact \
  --config delete.retention.ms=86400000
```

> [!INFO] `cleanup.policy=compact` — Kafka хранит только последнее сообщение для каждого ключа. Используется для хранения актуального состояния (например, профили пользователей).

### 3. Управление топиками

```bash
# Список всех топиков
docker exec -it kafka kafka-topics --list \
  --bootstrap-server localhost:9092

# Увеличение количества партиций (уменьшить нельзя!)
docker exec -it kafka kafka-topics --alter \
  --topic my-first-topic \
  --partitions 6 \
  --bootstrap-server localhost:9092

# Удаление топика
docker exec -it kafka kafka-topics --delete \
  --topic my-first-topic \
  --bootstrap-server localhost:9092
```

> [!WARNING] Количество партиций можно только увеличивать, но не уменьшать. После увеличения перераспределение сообщений по партициям меняется — существующие данные остаются в старых партициях.

## Простой Producer

### 1. Консольный Producer

```bash
docker exec -it kafka kafka-console-producer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092
# Вводим сообщения построчно, Enter — отправить
```

### 2. Producer с ключами

```bash
docker exec -it kafka kafka-console-producer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Формат ввода:
# user-123:{"action": "login", "timestamp": "2024-01-01T10:00:00Z"}
# user-456:{"action": "logout", "timestamp": "2024-01-01T10:05:00Z"}
```

### 3. Producer с настройками надёжности

```bash
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
# Читать с начала
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --from-beginning

# С метаданными
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --property "print.timestamp=true" \
  --property "print.key=true" \
  --property "print.value=true"
```

### 2. Consumer с настройками группы

```bash
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
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --partition 0 \
  --offset 0
```

## Consumer Groups

### 1. Создание Consumer Group

```bash
docker exec -it kafka kafka-console-consumer \
  --topic my-first-topic \
  --bootstrap-server localhost:9092 \
  --group my-group \
  --from-beginning
```

### 2. Управление Consumer Groups

```bash
# Список групп
docker exec -it kafka kafka-consumer-groups --list \
  --bootstrap-server localhost:9092

# Детали группы: партиции, offset, lag
docker exec -it kafka kafka-consumer-groups --describe \
  --group my-group \
  --bootstrap-server localhost:9092

# Сброс offset (для перечитывания сообщений)
docker exec -it kafka kafka-consumer-groups --reset-offsets \
  --group my-group \
  --topic my-first-topic \
  --to-earliest \
  --execute \
  --bootstrap-server localhost:9092
```

### 3. Масштабирование Consumer Groups

```bash
# Запустить несколько консьюмеров в одной группе (в разных терминалах)
# Kafka автоматически распределит партиции между ними

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
```

> [!WARNING] Количество активных консьюмеров в группе не должно превышать количество партиций в топике. Лишние консьюмеры будут простаивать без назначенных партиций.

## Практические примеры

### 1. Система логирования

```bash
# Топик для логов
docker exec -it kafka kafka-topics --create \
  --topic application-logs \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Producer (ключ — уровень лога)
docker exec -it kafka kafka-console-producer \
  --topic application-logs \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Ввод:
# ERROR:{"level": "ERROR", "message": "Database connection failed", "timestamp": "2024-01-01T10:00:00Z"}
# INFO:{"level": "INFO", "message": "User logged in", "timestamp": "2024-01-01T10:01:00Z"}

# Consumer
docker exec -it kafka kafka-console-consumer \
  --topic application-logs \
  --bootstrap-server localhost:9092 \
  --group log-processor \
  --property "print.key=true" \
  --property "print.value=true"
```

### 2. Система событий пользователей

```bash
docker exec -it kafka kafka-topics --create \
  --topic user-events \
  --bootstrap-server localhost:9092 \
  --partitions 5 \
  --replication-factor 1

# Ключ = user-id, так все события одного пользователя попадают в одну партицию
docker exec -it kafka kafka-console-producer \
  --topic user-events \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Ввод:
# user-123:{"event": "login", "timestamp": "2024-01-01T10:00:00Z", "ip": "192.168.1.1"}
# user-456:{"event": "purchase", "timestamp": "2024-01-01T10:05:00Z", "amount": 99.99}
```

### 3. Система уведомлений

```bash
docker exec -it kafka kafka-topics --create \
  --topic notifications \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Ключ = тип уведомления
docker exec -it kafka kafka-console-producer \
  --topic notifications \
  --bootstrap-server localhost:9092 \
  --property "parse.key=true" \
  --property "key.separator=:"

# Ввод:
# email:{"type": "email", "to": "user@example.com", "subject": "Welcome"}
# sms:{"type": "sms", "to": "+1234567890", "message": "Your order has been shipped"}
# push:{"type": "push", "device_id": "device-123", "title": "New message"}
```

## Отладка и мониторинг

### 1. Проверка состояния топиков

```bash
# Детальная информация
docker exec -it kafka kafka-topics --describe \
  --topic my-first-topic \
  --bootstrap-server localhost:9092

# Партиции с отставанием репликации
docker exec -it kafka kafka-topics --describe \
  --under-replicated-partitions \
  --bootstrap-server localhost:9092

# Недоступные партиции
docker exec -it kafka kafka-topics --describe \
  --unavailable-partitions \
  --bootstrap-server localhost:9092
```

### 2. Мониторинг Consumer Groups

```bash
# Список групп
docker exec -it kafka kafka-consumer-groups --list \
  --bootstrap-server localhost:9092

# Детали группы: CURRENT-OFFSET, LOG-END-OFFSET, LAG
docker exec -it kafka kafka-consumer-groups --describe \
  --group my-group \
  --bootstrap-server localhost:9092
```

> [!INFO] **Consumer lag** — разница между LOG-END-OFFSET (последнее сообщение в партиции) и CURRENT-OFFSET (последнее обработанное). Большой lag означает, что консьюмер не справляется с нагрузкой.

### 3. Тестирование производительности

```bash
# Тест producer (100 000 сообщений по 1KB, 10K/сек)
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
docker logs kafka
docker logs zookeeper
docker logs -f kafka  # в реальном времени
```

## Вопросы для собеседования

### Базовые вопросы
1. **Как создать топик?**
   ```bash
   kafka-topics --create --topic my-topic \
     --bootstrap-server localhost:9092 \
     --partitions 3 --replication-factor 1
   ```

2. **Что такое consumer group?**
   - Группа консьюмеров, совместно читающих топик
   - Каждая партиция назначается одному консьюмеру в группе
   - Обеспечивает масштабируемость и отказоустойчивость

3. **Как работает партиционирование?**
   - По ключу: `hash(key) % num_partitions`
   - Без ключа: round-robin по партициям
   - Одинаковый ключ — всегда одна партиция

### Продвинутые вопросы
4. **Как обеспечить порядок сообщений?**
   - Использовать ключ для партиционирования
   - Порядок гарантирован только внутри одной партиции

5. **Что такое offset и как он работает?**
   - Уникальный идентификатор сообщения в партиции
   - Consumer хранит offset прочитанных сообщений
   - Автоматический commit (`enable.auto.commit=true`) или ручной

6. **Как масштабировать консьюмеры?**
   - Добавить консьюмеров в группу (не более числа партиций)
   - Kafka автоматически перераспределит партиции (rebalance)

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
   - По времени: `retention.ms` (например, 86400000 = 1 день)
   - По размеру: `retention.bytes`
   - Применяется на уровне топика или брокера
