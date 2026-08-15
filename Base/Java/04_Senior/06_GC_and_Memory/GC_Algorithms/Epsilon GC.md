# Epsilon GC

> **Epsilon GC** (JEP 318, Java 11, experimental → Java 17 production) — "no-op" GC: аллоцирует память, но **никогда не собирает мусор**. При исчерпании heap → `OutOfMemoryError`. Не для продакшена — для performance testing, short-lived jobs, latency benchmarking.

## Связанные темы
[[G1GC — архитектура и tuning]], [[ZGC и Generational ZGC]], [[JVM флаги для GC]], [[Структура памяти JVM]]

---

## Включение

```bash
-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC  # Java 11-16
-XX:+UseEpsilonGC  # Java 17+ (без UnlockExperimental)
```

## Как это работает

Epsilon реализует только один механизм — **аллокацию** через bump pointer:

```
Heap: [──────────────────────────────────────────────]
       ↑                    ↑
       start          current_top = последний аллоцированный объект + size

Новый объект:
  ptr = current_top
  current_top += obj_size
  return ptr

Когда current_top >= heap_end → OutOfMemoryError
```

Никаких write barriers, marking, sweeping, compacting. Нулевой GC overhead.

## Зачем это нужно

### 1. Baseline для performance benchmarks

```bash
# Измерить чистую throughput без GC:
java -XX:+UseEpsilonGC -Xmx8g -Xms8g MyBenchmark

# Сравнить с G1:
java -XX:+UseG1GC -Xmx8g -Xms8g MyBenchmark

# Разница = GC overhead
```

### 2. Ultra-short-lived processes

```bash
# Lambda функция / batch job который живёт < 1 секунды
# и никогда не успевает исчерпать heap:
java -XX:+UseEpsilonGC -Xmx512m ProcessOneRequest
# Zero GC pauses — если memory в пределах heap
```

### 3. GC latency elimination

```java
// Trading system: 99.99% latency target < 1 мс
// Запускаем в коротких сессиях (restart после Xmx исчерпания)
// Epsilon гарантирует zero GC паузы в пределах сессии
```

### 4. Memory leak detection

```bash
# С Epsilon любая утечка сразу → OOM
# Быстро найти "держит ли этот код память"
java -XX:+UseEpsilonGC -Xmx256m -verbose:gc MyApp
```

## Настройка

```bash
# Размер heap (весь доступен для аллокации)
-Xms512m -Xmx512m  # фиксируй — Epsilon не понимает heap resize

# TLAB (Thread-Local Allocation Buffer) tuning
-XX:TLABSize=1m        # размер TLAB на поток
-XX:+ResizeTLAB        # авторегулировка (default)

# Логирование аллокации (для диагностики)
-XX:+EpsilonPrintHeapSteps  # периодически логировать состояние heap
-XX:EpsilonUpdateCountAtPollInterval=1000  # каждые N аллокаций
```

## Пример вывода

```
[gc,heap       ] Heap: 512M reserved, 512M (100.00%) committed, 312M (60.94%) used
[gc,heap       ] Heap: 512M reserved, 512M (100.00%) committed, 412M (80.47%) used
...
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

## Когда точно не использовать

- Любой production сервис с неограниченным временем работы
- Сервисы под нагрузкой (короткоживущие объекты накапливаются)
- Когда не можешь доказать, что heap не будет исчерпан

## Epsilon + GraalVM Native Image

В GraalVM Native Image можно использовать Epsilon для "hello world"-приложений или CLI tools где startup latency критична и GC не нужен.

---

## Вопросы на интервью
- Зачем нужен GC который ничего не собирает?
- Как Epsilon GC выделяет память?
- В каких production сценариях оправдан Epsilon GC?
- Как использовать Epsilon для измерения GC overhead?

## Подводные камни
- Epsilon + утечки памяти = быстрый OOM. Это особенность, а не баг — используй для обнаружения утечек
- `Xms != Xmx` с Epsilon не имеет смысла — heap не shrink/grow
- Если приложение живёт дольше чем heap хватает → crash без recovery. Мониторь heap usage
