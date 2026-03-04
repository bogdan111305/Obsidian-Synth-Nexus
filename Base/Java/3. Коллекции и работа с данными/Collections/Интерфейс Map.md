---
title: "Java — Интерфейс Map (HashMap, LinkedHashMap, TreeMap)"
tags: [java, collections, map, hashmap, treemap, concurrenthashmap]
updated: 2026-03-04
---

# Интерфейс Map в Java

`Map` — это интерфейс в Java из пакета `java.util`, предназначенный для хранения пар **ключ-значение**, где каждый ключ уникален, а значения могут повторяться. Он широко используется для быстрого доступа к данным по ключу, кэширования, конфигураций и многопоточных приложений. Эта статья подробно рассматривает интерфейс `Map`, его реализации (`HashMap`, `LinkedHashMap`, `TreeMap`, `ConcurrentHashMap`, `WeakHashMap`), внутренние механизмы, производительность, Big-O операций, примеры применения и современные возможности (Java 21+).

## 1. Основные характеристики `Map`

- **Уникальность ключей**: Каждый ключ уникален, дублирование перезаписывает значение.
- **Основные операции**:
    - `put(K key, V value)`: Добавить или обновить пару.
    - `get(K key)`: Получить значение по ключу.
    - `remove(K key)`: Удалить пару по ключу.
    - `containsKey(K key)`: Проверяет наличие ключа.
    - `containsValue(V value)`: Проверяет наличие значения.
    - `size()`, `isEmpty()`, `clear()`: Управление размером и содержимым.

**Иерархия реализаций**:

```
java.util.Map<K,V> (интерфейс)
├─ java.util.HashMap<K,V>
├─ java.util.LinkedHashMap<K,V>
├─ java.util.TreeMap<K,V>
├─ java.util.concurrent.ConcurrentHashMap<K,V>
└─ java.util.WeakHashMap<K,V>
```

## 2. Сравнение реализаций

|Класс|Структура хранения|Порядок элементов|Потокобезопасность|Ключи отсортированы?|Особенности|
|---|---|---|---|---|---|
|`HashMap`|Хеш-таблица (массив + списки/деревья)|Нет|Нет|Нет|Быстрая работа (O(1) аморт.), неупорядоченный|
|`LinkedHashMap`|Хеш-таблица + двусвязный список|В порядке вставки/доступа|Нет|Нет|Итерация в порядке вставки или доступа|
|`TreeMap`|Красно-чёрное дерево|Отсортированы|Нет|Да|Ключи упорядочены, операции O(log n)|
|`ConcurrentHashMap`|Сегментированная хеш-таблица|Нет|Да|Нет|Потокобезопасный, без блокировок|
|`WeakHashMap`|Хеш-таблица с `WeakReference` ключами|Нет|Нет|Нет|Ключи удаляются сборщиком мусора при отсутствии ссылок|

## 3. `HashMap`

### 3.1. Внутреннее устройство

- **Структура**: Хеш-таблица, основанная на массиве бакетов (`Node<K,V>[] table`).
- **Узел (`Node`)**:
    - Поля: `hash`, `key`, `value`, `next` (для цепочки коллизий).
    - Хранится в бакете, определяемом индексом: `index = (n - 1) & hash`, где `n` — длина массива.
- **Хеширование**:
    - Вычисляется `hashCode()` ключа, затем улучшается функцией `hash` (XOR с битовым сдвигом).
    - Пример: `hash = key.hashCode() ^ (key.hashCode() >>> 16)`.
- **Коллизии**:
    - До Java 8: Цепочка (связный список).
    - После Java 8: Если в бакете ≥ 8 элементов (`TREEIFY_THRESHOLD`), цепочка преобразуется в красно-чёрное дерево, улучшая поиск до O(log n).
    - Обратное преобразование в список при ≤ 6 элементов (`UNTREEIFY_THRESHOLD`).

### 3.2. Хранение и расширение

- **Хранение**: Массив `table` хранит указатели на `Node` или `TreeNode` (для деревьев).
- **Начальная ёмкость**: По умолчанию 16, `loadFactor = 0.75`.
- **Расширение**:
    - При достижении `size > capacity * loadFactor` массив удваивается (O(n)).
    - Элементы перераспределяются: новый индекс = `(newCapacity - 1) & hash`.
    - Деревья могут преобразоваться обратно в списки, если коллизий стало меньше.
- **JVM**: AQS не используется, но хеширование и CAS применяются для оптимизации.

### 3.3. Big-O операций

|Операция|Big-O (среднее)|Big-O (худший случай)|Пояснение|
|---|---|---|---|
|`get(key)`|O(1) амортизированное|O(log n) / O(n)|Поиск по хешу; O(log n) при дереве, O(n) при плохом `hashCode`|
|`put(key, value)`|O(1) амортизированное|O(log n) / O(n)|Вставка в бакет; resize O(n) редок, дерево O(log n)|
|`remove(key)`|O(1) амортизированное|O(log n) / O(n)|Удаление из бакета; аналогично `get`|
|`containsKey(key)`|O(1) амортизированное|O(log n) / O(n)|Поиск ключа; аналогично `get`|
|`containsValue(value)`|O(n)|O(n)|Линейный перебор всех элементов|
|`iteration`|O(n)|O(n)|Обход всех бакетов и узлов|
|`resize`|O(n)|O(n)|Удвоение массива и перераспределение всех элементов|

**Примечание**: Худший случай (O(n)) возникает при плохом `hashCode`, вызывающем коллизии. Деревья (Java 8+) снижают его до O(log n).

### 3.4. Пример

```java
Map<String, Integer> map = new HashMap<>();
map.put("key1", 1);
map.put("key2", 2);
System.out.println(map.get("key1")); // 1
```

## 4. `LinkedHashMap`

### 4.1. Внутреннее устройство

- **Наследование**: Расширяет `HashMap`, добавляя двусвязный список.
- **Узел (`Entry`)**:
    - Наследует `HashMap.Node`, добавляет поля `before` и `after` для двусвязного списка.
- **Хранение**:
    - Хеш-таблица: Как в `HashMap` (`Node<K,V>[] table`).
    - Двусвязный список: Хранит порядок вставки или доступа (`accessOrder = true`).
- **Порядок**:
    - По умолчанию: Порядок вставки.
    - При `accessOrder = true`: Порядок последнего доступа (LRU-кэш).

### 4.2. Расширение

- **Расширение**: Как в `HashMap`, массив бакетов удваивается при `size > capacity * loadFactor` (O(n)).
- **Двусвязный список**: Не требует перестройки, только обновляются ссылки в хеш-таблице.
- **JVM**: Ссылки `before`/`after` управляются через прямые указатели, без AQS.

### 4.3. Big-O операций

|Операция|Big-O (среднее)|Big-O (худший случай)|Пояснение|
|---|---|---|---|
|`get(key)`|O(1) амортизированное|O(log n) / O(n)|Поиск по хешу; обновление порядка при `accessOrder = true`|
|`put(key, value)`|O(1) амортизированное|O(log n) / O(n)|Вставка в хеш-таблицу и двусвязный список; resize O(n) редок|
|`remove(key)`|O(1) амортизированное|O(log n) / O(n)|Удаление из хеш-таблицы и списка|
|`containsKey(key)`|O(1) амортизированное|O(log n) / O(n)|Аналогично `get`|
|`containsValue(value)`|O(n)|O(n)|Линейный перебор всех элементов|
|`iteration`|O(n)|O(n)|Обход двусвязного списка в порядке вставки/доступа|
|`resize`|O(n)|O(n)|Удвоение массива и перераспределение|

### 4.4. Пример (LRU-кэш)

```java
LinkedHashMap<String, Integer> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
        return size() > 3; // Ограничиваем до 3 элементов
    }
};
lruCache.put("A", 1);
lruCache.put("B", 2);
lruCache.put("C", 3);
lruCache.get("A"); // Перемещает "A" в конец
lruCache.put("D", 4); // Удаляет "B" (старейший)
```

## 5. `TreeMap`

### 5.1. Внутреннее устройство

- **Структура**: Красно-чёрное дерево (самобалансирующееся бинарное дерево поиска).
- **Узел (`Entry`)**:
    - Поля: `key`, `value`, `left`, `right`, `parent`, `color` (красный/чёрный).
- **Сортировка**:
    - Ключи упорядочены по `Comparable` или `Comparator`.
    - Поиск, вставка и удаление требуют балансировки (повороты, перекраска).
- **JVM**: Не использует AQS, работает через прямое управление указателями.

### 5.2. Хранение и масштабирование

- **Хранение**: Динамическое дерево, растёт с добавлением узлов.
- **Масштабирование**: Нет массива, поэтому нет `resize`. Балансировка O(log n).
- **Операции по позициям**:
    - Нет прямого доступа по индексу.
    - Методы `firstKey()`, `lastKey()`, `pollFirstEntry()`, `pollLastEntry()` — O(log n).
    - Итерация: В порядке ключей (in-order traversal).

### 5.3. Big-O операций

|Операция|Big-O (среднее)|Big-O (худший случай)|Пояснение|
|---|---|---|---|
|`get(key)`|O(log n)|O(log n)|Поиск по дереву|
|`put(key, value)`|O(log n)|O(log n)|Вставка с балансировкой|
|`remove(key)`|O(log n)|O(log n)|Удаление с балансировкой|
|`containsKey(key)`|O(log n)|O(log n)|Аналогично `get`|
|`containsValue(value)`|O(n)|O(n)|Линейный перебор всех узлов|
|`iteration`|O(n)|O(n)|Обход дерева в порядке ключей|
|`firstKey/lastKey`|O(log n)|O(log n)|Доступ к минимальному/максимальному ключу|

### 5.4. Пример

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(3, "Three");
map.put(1, "One");
map.put(2, "Two");
System.out.println(map.firstKey()); // 1
System.out.println(map.lastEntry()); // Entry(3, "Three")
```

## 6. `ConcurrentHashMap`

### 6.1. Внутреннее устройство

- **Структура**: Сегментированная хеш-таблица.
- **Узел (`Node`)**: Аналогичен `HashMap`, но с volatile полями для потокобезопасности.
- **Сегменты**: Массив `Node<K,V>[] table`, разделённый на сегменты.
- **Потокобезопасность**:
    - Использует CAS (`compareAndSet`) и `synchronized` на уровне бакетов.
    - AQS для управления конкуренцией.
- **Хранение**:
    - Как `HashMap`, но с разделением на сегменты.
    - При коллизиях: Списки или деревья (Java 8+).

### 6.2. Расширение

- **Расширение**: Удвоение массива бакетов (O(n)), но сегментировано для минимизации блокировок.
- **Механизм**: Потоки помогают в перераспределении (`transfer`).
- **JVM**: Использует `sun.misc.Unsafe` для CAS и volatile-поля.

### 6.3. Big-O операций

|Операция|Big-O (среднее)|Big-O (худший случай)|Пояснение|
|---|---|---|---|
|`get(key)`|O(1) амортизированное|O(log n) / O(n)|Поиск без блокировок, аналогично `HashMap`|
|`put(key, value)`|O(1) амортизированное|O(log n) / O(n)|Вставка с CAS или локальной синхронизацией|
|`remove(key)`|O(1) амортизированное|O(log n) / O(n)|Удаление с синхронизацией|
|`containsKey(key)`|O(1) амортизированное|O(log n) / O(n)|Аналогично `get`|
|`containsValue(value)`|O(n)|O(n)|Линейный перебор всех элементов|
|`iteration`|O(n)|O(n)|Обход всех бакетов, возможна конкуренция|
|`resize`|O(n)|O(n)|Сегментированное перераспределение|

### 6.4. Пример

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("key1", 1);
map.computeIfAbsent("key2", k -> 2);
System.out.println(map.get("key1")); // 1
```

### 6.5. ConcurrentHashMap: Java 7 vs Java 8+ (детали реализации)

**Java 7: Segment-based locking**

В Java 7 `ConcurrentHashMap` делил таблицу на `Segment[]` — каждый сегмент был независимым `ReentrantLock`. По умолчанию 16 сегментов → параллелизм 16 потоков.

```java
// Java 7: упрощённая схема внутреннего устройства
static final class Segment<K,V> extends ReentrantLock {
    volatile HashEntry<K,V>[] table;  // под-таблица сегмента
    int count;       // число элементов в сегменте
    int threshold;   // порог для resize сегмента
    float loadFactor;
}

// put() в Java 7: блокировка на весь сегмент
public V put(K key, V value) {
    int hash = hash(key);
    Segment<K,V> s = segmentFor(hash);  // hash >> segmentShift
    s.lock();                            // блокируем весь сегмент (~1/16 таблицы)
    try {
        return s.put(key, hash, value, false);
    } finally {
        s.unlock();
    }
}
```

**Проблемы Java 7:**
- `size()` блокировала все 16 сегментов последовательно
- Конкуренция на одном сегменте блокирует 1/16 таблицы целиком
- `Segment[]` — дополнительная индерекция и объекты в heap
- Уровень параллелизма фиксирован (`concurrencyLevel`, по умолчанию 16)

---

**Java 8+: CAS + synchronized на первом узле бакета**

В Java 8 сегменты полностью убраны. Синхронизация происходит на уровне **отдельного бакета**. Для пустых бакетов — CAS, для непустых — `synchronized` на `f` (первый `Node`).

```java
// Java 8: упрощённая схема putVal()
final V putVal(K key, V value, boolean onlyIfAbsent) {
    int hash = spread(key.hashCode());  // (h ^ (h >>> 16)) & HASH_BITS
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0) {
            tab = initTable();          // CAS-инициализация таблицы
        } else if ((f = tabAt(tab, i = (n-1) & hash)) == null) {
            // Бакет пуст — CAS без блокировки, lock-free!
            if (casTabAt(tab, i, null, new Node<>(hash, key, value, null)))
                break;
        } else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f); // кооперативный resize
        } else {
            synchronized (f) {          // блокируем ТОЛЬКО первый узел бакета
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {      // обычный список
                        // добавляем/обновляем в связном списке
                    } else if (f instanceof TreeBin) {
                        // добавляем в красно-чёрное дерево
                    }
                }
            }
            if (binCount >= TREEIFY_THRESHOLD) treeifyBin(tab, i);
        }
    }
    addCount(1L, binCount);  // LongAdder-style счётчик
}
```

**Ключевые детали Java 8+:**
- `get()` — полностью без блокировок (поля `Node.val` и `Node.next` объявлены `volatile`)
- `size()` использует `baseCount` (CAS) + `counterCells[]` (аналог `LongAdder`), не является атомарной
- При resize потоки-помощники подключаются через `helpTransfer` — cooperative resizing
- Уровень параллелизма = количество бакетов (растёт при resize: 16 → 32 → ...)

**Сравнение Java 7 vs Java 8+:**

|Характеристика|Java 7 (Segments)|Java 8+ (CAS + Node lock)|
|---|---|---|
|Синхронизация|`ReentrantLock` на сегменте (1/16 таблицы)|CAS для пустых бакетов + `synchronized` на Node|
|Параллелизм записи|≤ 16 потоков одновременно|≤ capacity потоков одновременно|
|`get()`|Без блокировки (volatile)|Без блокировки (volatile)|
|`put()` в пустой бакет|Блокировка сегмента|CAS — lock-free|
|`put()` при коллизии|Блокировка сегмента|`synchronized` только на первом Node|
|`size()`|Блокирует все сегменты (точно)|`baseCount + sum(counterCells)` (приблизительно)|
|Memory overhead|`Segment[]` + доп. объекты|Минимальный (только `Node[]`)|
|Коллизии|Только списки|Списки → деревья при ≥8 коллизий|

## 7. `WeakHashMap`

### 7.1. Внутреннее устройство

- **Структура**: Хеш-таблица, аналогичная `HashMap`.
- **Узел**: Ключи хранятся как `WeakReference`, значения — как обычные ссылки.
- **Сборка мусора**:
    - Ключи, на которые нет сильных ссылок, удаляются сборщиком мусора.
    - Используется `ReferenceQueue` для очистки удалённых ключей.
- **JVM**: Не использует AQS, но зависит от GC.

### 7.2. Расширение

- **Расширение**: Как в `HashMap`, удвоение массива (O(n)).
- **Очистка**: Периодически вызывается `expungeStaleEntries()` для удаления устаревших ключей.

### 7.3. Big-O операций

|Операция|Big-O (среднее)|Big-O (худший случай)|Пояснение|
|---|---|---|---|
|`get(key)`|O(1) амортизированное|O(n)|Поиск по хешу, аналогично `HashMap`|
|`put(key, value)`|O(1) амортизированное|O(n)|Вставка с очисткой устаревших ключей|
|`remove(key)`|O(1) амортизированное|O(n)|Удаление с возможной очисткой|
|`containsKey(key)`|O(1) амортизированное|O(n)|Аналогично `get`|
|`containsValue(value)`|O(n)|O(n)|Линейный перебор всех элементов|
|`iteration`|O(n)|O(n)|Обход всех бакетов, возможна очистка|
|`resize`|O(n)|O(n)|Удвоение массива и перераспределение|

### 7.4. Пример

```java
WeakHashMap<String, Integer> map = new WeakHashMap<>();
String key = new String("key"); // WeakReference
map.put(key, 1);
key = null; // Удаляем сильную ссылку
System.gc(); // Ключ может быть удалён
```

## 8. Практическое применение

- **Java EE / Jakarta EE**:
    - Хранение конфигураций (`HashMap`).
    - Кэширование сессий (`ConcurrentHashMap`).
- **Spring**:
    - Кэширование в `@Cacheable` (`ConcurrentHashMap`).
    - Сохранение порядка обработки (`LinkedHashMap`).
- **Кэши**:
    - LRU-кэш с `LinkedHashMap`.
    - Слабые ссылки с `WeakHashMap` для автоматической очистки.
- **Тестирование**:
    - Хранение результатов тестов по ключам.

**Пример (Spring)**:

```java
@Service
public class CacheService {
    private final ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();

    @Cacheable("data")
    public String getData(String key) {
        return cache.computeIfAbsent(key, k -> fetchData(k));
    }

    private String fetchData(String key) {
        return "Data for " + key;
    }
}
```

## 9. Современные возможности (Java 21+)

### 9.1. Виртуальные потоки

`ConcurrentHashMap` эффективен с виртуальными потоками (Java 21, Project Loom).

**Пример**:

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> map.put("key", 1));
```

### 9.2. Интеграция с `CompletableFuture`

```java
CompletableFuture.supplyAsync(() -> {
    map.put("key", 1);
    return map.get("key");
}).thenAccept(System.out::println);
```

## 10. Подводные камни

1. **Плохой `hashCode`**:
    - Много коллизий ухудшает производительность до O(n).
    - **Решение**: Реализуйте качественный `hashCode` и `equals`.
2. **Потокобезопасность**:
    - `HashMap`, `LinkedHashMap`, `TreeMap`, `WeakHashMap` не потокобезопасны.
    - **Решение**: Используйте `ConcurrentHashMap` или `Collections.synchronizedMap`.
3. **Сборка мусора в `WeakHashMap`**:
    - Ключи могут быть удалены неожиданно.
    - **Решение**: Проверяйте наличие ключа перед использованием.
4. **Переполнение `HashMap`**:
    - Неограниченный рост вызывает `resize` и O(n).
    - **Решение**: Устанавливайте начальную ёмкость.

## 11. Производительность

- **`HashMap`**: Быстрейшая для однопоточных приложений (O(1) амортизированное).
- **`LinkedHashMap`**: Чуть медленнее из-за двусвязного списка.
- **`TreeMap`**: O(log n), подходит для сортировки.
- **`ConcurrentHashMap`**: Высокая производительность в многопоточных сценариях.
- **`WeakHashMap`**: Скорость как у `HashMap`, но с накладными расходами на GC.

## 12. Лучшие практики

1. **Выбирайте правильную реализацию**:
    - `HashMap` для общего использования.
    - `LinkedHashMap` для порядка вставки/доступа.
    - `TreeMap` для сортировки.
    - `ConcurrentHashMap` для многопоточности.
    - `WeakHashMap` для кэшей с автоматической очисткой.
2. **Настройте начальную ёмкость**:
    
    ```java
    HashMap<String, Integer> map = new HashMap<>(1000);
    ```
    
3. **Реализуйте качественный `hashCode`/`equals`**:
    
    ```java
    @Override
    public int hashCode() {
        return Objects.hash(field1, field2);
    }
    ```
    
4. **Обрабатывайте исключения в многопоточности**:
    
    ```java
    try {
        concurrentMap.computeIfAbsent(key, k -> computeValue(k));
    } catch (Exception e) {
        handleError(e);
    }
    ```
    
5. **Тестируйте производительность**:
    
    ```java
    @Test
    void testHashMap() {
        Map<String, Integer> map = new HashMap<>();
        map.put("key", 1);
        assertEquals(1, map.get("key"));
    }
    ```
    

## 13. Пример: Кэширование с `ConcurrentHashMap`

```java
import java.util.concurrent.ConcurrentHashMap;

public class DataCache {
    private final ConcurrentHashMap<String, String> cache = new ConcurrentHashMap<>();

    public String getData(String key) {
        return cache.computeIfAbsent(key, k -> {
            try {
                Thread.sleep(1000); // Симуляция запроса
                return "Data for " + k;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return null;
            }
        });
    }
}
```

## 14. Современные возможности Java 21+

### Pattern Matching для Map

Java 21 улучшил pattern matching, что полезно при работе с Map:

```java
Map<String, Object> data = Map.of("name", "John", "age", 30, "city", "NYC");

// Pattern matching в switch expressions
String result = switch (data.get("type")) {
    case String s -> "String: " + s;
    case Integer i -> "Number: " + i;
    case null -> "Null value";
    default -> "Unknown type";
};

// Pattern matching в instanceof
data.forEach((key, value) -> {
    if (value instanceof String s && s.length() > 3) {
        System.out.println("Long string: " + s);
    } else if (value instanceof Integer i && i > 25) {
        System.out.println("Adult age: " + i);
    }
});
```

### Structured Concurrency с Map

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Map<String, Future<String>> futures = Map.of(
        "user", scope.fork(() -> fetchUserData()),
        "profile", scope.fork(() -> fetchProfileData()),
        "settings", scope.fork(() -> fetchSettingsData())
    );
    
    scope.join();
    scope.throwIfFailed();
    
    Map<String, String> results = futures.entrySet().stream()
        .collect(Collectors.toMap(
            Map.Entry::getKey,
            entry -> entry.getValue().resultNow()
        ));
}
```

### Виртуальные потоки и Map

```java
Map<String, CompletableFuture<String>> asyncMap = Map.of(
    "key1", CompletableFuture.supplyAsync(() -> "value1"),
    "key2", CompletableFuture.supplyAsync(() -> "value2")
);

ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
CompletableFuture<Map<String, String>> result = CompletableFuture.supplyAsync(() -> {
    return asyncMap.entrySet().stream()
        .collect(Collectors.toMap(
            Map.Entry::getKey,
            entry -> entry.getValue().join()
        ));
}, executor);
```

## 15. Расширенные примеры использования

### Кэширование с TTL (Time To Live)

```java
public class TTLCache<K, V> {
    private final Map<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
    private final long ttlMillis;
    
    public TTLCache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
    }
    
    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry != null && !entry.isExpired()) {
            return entry.getValue();
        }
        cache.remove(key);
        return null;
    }
    
    public void put(K key, V value) {
        cache.put(key, new CacheEntry<>(value, System.currentTimeMillis()));
    }
    
    public void cleanup() {
        cache.entrySet().removeIf(entry -> entry.getValue().isExpired());
    }
    
    private static class CacheEntry<V> {
        private final V value;
        private final long timestamp;
        
        public CacheEntry(V value, long timestamp) {
            this.value = value;
            this.timestamp = timestamp;
        }
        
        public V getValue() { return value; }
        
        public boolean isExpired() {
            return System.currentTimeMillis() - timestamp > ttlMillis;
        }
    }
}
```

### Потокобезопасный кэш с ограничением размера

```java
public class BoundedCache<K, V> {
    private final Map<K, V> cache;
    private final int maxSize;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    
    public BoundedCache(int maxSize) {
        this.maxSize = maxSize;
        this.cache = new LinkedHashMap<>(maxSize, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > maxSize;
            }
        };
    }
    
    public V get(K key) {
        lock.readLock().lock();
        try {
            return cache.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public void put(K key, V value) {
        lock.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public Map<K, V> snapshot() {
        lock.readLock().lock();
        try {
            return new HashMap<>(cache);
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

### Функциональные операции с Map

```java
public class FunctionalMapOperations {
    
    // Трансформация значений
    public static <K, V, R> Map<K, R> transformValues(
            Map<K, V> map, Function<V, R> transformer) {
        return map.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                entry -> transformer.apply(entry.getValue())
            ));
    }
    
    // Фильтрация по ключам и значениям
    public static <K, V> Map<K, V> filterMap(
            Map<K, V> map, 
            Predicate<K> keyFilter, 
            Predicate<V> valueFilter) {
        return map.entrySet().stream()
            .filter(entry -> keyFilter.test(entry.getKey()) && 
                           valueFilter.test(entry.getValue()))
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue
            ));
    }
    
    // Группировка с агрегацией
    public static <K, V> Map<K, List<V>> groupByAndAggregate(
            List<V> items, 
            Function<V, K> keyExtractor,
            Function<V, V> valueTransformer) {
        return items.stream()
            .collect(Collectors.groupingBy(
                keyExtractor,
                Collectors.mapping(valueTransformer, Collectors.toList())
            ));
    }
    
    // Слияние Map с разрешением конфликтов
    public static <K, V> Map<K, V> mergeMaps(
            Map<K, V> map1, 
            Map<K, V> map2, 
            BiFunction<V, V, V> conflictResolver) {
        Map<K, V> result = new HashMap<>(map1);
        map2.forEach((key, value) -> 
            result.merge(key, value, conflictResolver));
        return result;
    }
}
```

### Интеграция с внешними системами

```java
public class MapIntegration {
    
    // Загрузка конфигурации из файла
    public static Map<String, String> loadConfig(String filename) {
        try (Stream<String> lines = Files.lines(Paths.get(filename))) {
            return lines
                .filter(line -> !line.trim().isEmpty() && !line.startsWith("#"))
                .map(line -> line.split("=", 2))
                .filter(parts -> parts.length == 2)
                .collect(Collectors.toMap(
                    parts -> parts[0].trim(),
                    parts -> parts[1].trim()
                ));
        } catch (IOException e) {
            throw new RuntimeException("Error loading config", e);
        }
    }
    
    // Экспорт в JSON
    public static String toJson(Map<String, Object> map) {
        return map.entrySet().stream()
            .map(entry -> String.format("\"%s\": \"%s\"", 
                entry.getKey(), entry.getValue()))
            .collect(Collectors.joining(", ", "{", "}"));
    }
    
    // Асинхронная загрузка данных
    public static CompletableFuture<Map<String, String>> loadDataAsync(
            List<String> urls) {
        List<CompletableFuture<Map.Entry<String, String>>> futures = 
            urls.stream()
                .map(url -> CompletableFuture.supplyAsync(() -> 
                    Map.entry(url, fetchData(url))))
                .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    Map.Entry::getValue
                )));
    }
    
    private static String fetchData(String url) {
        // Реализация HTTP запроса
        return "data from " + url;
    }
}
```

## 16. Вопросы для собеседования

### Базовые вопросы

1. **Объясните разницу между HashMap и TreeMap**
   - HashMap: O(1) среднее время, неупорядоченный
   - TreeMap: O(log n), отсортированный по ключам
   - HashMap использует хеш-таблицу, TreeMap - красно-черное дерево

2. **Что такое ConcurrentHashMap и как он работает?**
   - Потокобезопасная реализация Map
   - Использует сегментированную структуру для уменьшения блокировок
   - CAS операции для атомарных операций

3. **Объясните механизм хеширования в HashMap**
   - `hashCode()` вычисляет хеш ключа
   - Индекс = (capacity - 1) & hash
   - Коллизии разрешаются списками или деревьями (Java 8+)

### Продвинутые вопросы

4. **Как работает WeakHashMap?**
   - Ключи хранятся как WeakReference
   - Автоматически удаляются сборщиком мусора
   - Полезен для кэшей без утечек памяти

5. **Объясните производительность различных реализаций Map**
   - HashMap: O(1) среднее, O(n) худшее
   - TreeMap: O(log n) гарантированное
   - LinkedHashMap: O(1) среднее + порядок

6. **Как реализовать LRU кэш с помощью LinkedHashMap?**
```java
LinkedHashMap<K, V> lruCache = new LinkedHashMap<>(capacity, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
};
```

### Практические вопросы

7. **Напишите код для нахождения наиболее частого элемента**
```java
public static <T> T findMostFrequent(List<T> list) {
    return list.stream()
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
        .entrySet().stream()
        .max(Map.Entry.comparingByValue())
        .map(Map.Entry::getKey)
        .orElse(null);
}
```

8. **Реализуйте потокобезопасный кэш с TTL**
```java
public class ThreadSafeCache<K, V> {
    private final Map<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
    
    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry != null && !entry.isExpired()) {
            return entry.getValue();
        }
        cache.remove(key);
        return null;
    }
    
    public void put(K key, V value, long ttlMillis) {
        cache.put(key, new CacheEntry<>(value, System.currentTimeMillis() + ttlMillis));
    }
}
```

9. **Как оптимизировать производительность HashMap?**
   - Задавайте начальную емкость
   - Реализуйте качественный hashCode() и equals()
   - Используйте примитивные типы где возможно
   - Избегайте частого изменения размера

## Load Factor 0.75: математическое обоснование

Значение `loadFactor = 0.75` в `HashMap` — не магическое число, а вывод из теории вероятностей. Объяснение приведено прямо в [javadoc JDK](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/HashMap.java):

> «Идеальное значение load factor при случайном хешировании следует из распределения Пуассона: при среднем числе элементов в бакете (λ) = 0.5 вероятность 8 или более элементов в одном бакете составляет 0.00000006.»

**Вывод через распределение Пуассона:**

При `loadFactor = 0.75` и случайном хешировании среднее число элементов в одном бакете:

```
λ = n / capacity = capacity * 0.75 / capacity = 0.75
```

Но реальная нагрузка на бакеты — это λ с учётом `tableSizeFor(n)`, и при средней заполненности ожидаемое λ ≈ **0.5**.

Вероятность ровно k элементов в бакете по формуле Пуассона:

```
P(k) = (λ^k * e^(-λ)) / k!

P(0) = 0.6065  (бакет пуст)
P(1) = 0.3033
P(2) = 0.0758
P(3) = 0.0126
...
P(8) ≈ 6×10⁻⁸  ← именно это TREEIFY_THRESHOLD = 8
```

**Почему TREEIFY_THRESHOLD = 8?** Потому что при нормальном хешировании вероятность попасть в бакет с 8 элементами составляет ~0.00000006. Если такое происходит — это сигнал плохого `hashCode`, и дерево оправдано несмотря на overhead создания `TreeNode` (который вдвое тяжелее `Node`).

**Почему не 0.5 или 1.0?**

|Load Factor|Коллизии|Resize частота|Итог|
|---|---|---|---|
|0.5|Очень мало|Часто (много памяти)|Расточительно|
|0.75|Мало (λ≈0.5)|Умеренно|**Оптимально**|
|1.0|Больше|Редко|Деградация O(n)|

## Hash Collision DoS Attack

**Вектор атаки (2011–2012):**

До Java 8 все коллизии хранились в связном списке — поиск O(n). В 2011 году на конференции 28C3 исследователи продемонстрировали атаку:

1. Злоумышленник знает алгоритм хеширования (например, `String.hashCode()`)
2. Генерирует тысячи строк с одинаковым `hashCode()`: например, `"Aa"`, `"BB"`, `"C#"` — у всех одинаковый хеш
3. Отправляет HTTP POST-запрос с такими параметрами (они парсятся в `HashMap`)
4. Сервер тратит O(n²) время на обработку → 100% CPU, DoS

**Пример строк с коллизией hashCode:**

```java
// Строки с одинаковым hashCode:
System.out.println("Aa".hashCode());   // 2112
System.out.println("BB".hashCode());   // 2112
// Можно строить произвольно длинные цепочки:
// "AaAa", "AaBB", "BBAa", "BBBB" — все дают hashCode = 2031744
```

**Защита в Java 8+:**

1. **Treeification**: при ≥8 коллизий в бакете список → красно-чёрное дерево (O(log n) вместо O(n))
2. **`ALTERNATIVE_HASHING`** (Java 7u6+): дополнительное рандомизированное хеширование для `String` при больших таблицах
3. **`HashMap.TREEIFY_THRESHOLD = 8`**: порог, вычисленный из распределения Пуассона (нормальный hashCode никогда не достигнет 8 коллизий)

**Почему `ConcurrentHashMap` более устойчив?**
- Также использует treeification с Java 8+
- Синхронизация на уровне бакета: атака замедляет только один бакет, не весь map

## Interview Q&A (Senior Level)

**Q1: Почему в Java 8 убрали Segment из ConcurrentHashMap и чем заменили?**

> `Segment extends ReentrantLock` в Java 7 фиксировал параллелизм на уровне 16 сегментов: при конкуренции блокировался целый сегмент (1/16 таблицы). Java 8 заменил это на синхронизацию **на первом узле бакета**: пустой бакет — CAS (lock-free), непустой — `synchronized (firstNode)`. Уровень параллелизма вырос с 16 до размера capacity (по умолчанию 16, но растёт при resize). Дополнительный выигрыш: убран overhead от `Segment[]` объектов, кооперативный resize через `helpTransfer`, `size()` через `LongAdder`-style счётчик `counterCells`.

**Q2: Почему load factor HashMap равен 0.75, а не 0.5 или 1.0?**

> Из javadoc JDK: при случайном хешировании количество элементов в бакетах подчиняется распределению Пуассона. При `loadFactor = 0.75` ожидаемое среднее λ ≈ 0.5, и вероятность 8 коллизий в одном бакете — ~6×10⁻⁸. Именно это значение (P(k≥8) ≈ 0) задаёт `TREEIFY_THRESHOLD = 8`. При 0.5 слишком много resize (память), при 1.0 — много коллизий (O(n) деградация). 0.75 — математически оптимальный компромисс.

**Q3: Что такое Hash Collision DoS-атака и как Java 8 защищается от неё?**

> До Java 8 все коллизии хранились в связном списке — O(n) поиск. В 2011–2012 исследователи показали: можно сгенерировать строки с одинаковым `hashCode()` (например, `"Aa"` и `"BB"`), отправить их как HTTP-параметры, и сервер потратит O(n²) на обработку → CPU 100%. Java 8 защищается через treeification: при ≥8 коллизий список → красно-чёрное дерево (O(log n)). Вероятность достичь 8 коллизий при честном `hashCode` составляет 6×10⁻⁸ — если это происходит, значит хеш плохой, и дерево оправдано.

**Q4: Что произойдёт, если изменить мутабельный ключ HashMap после вставки?**

> После вставки `map.put(key, value)` хранится пара с индексом бакета = `(capacity-1) & hash(key)`. Если изменить поле ключа, влияющее на `hashCode()`, — `hashCode()` изменится, индекс бакета станет другим. `map.get(key)` вычислит новый индекс, не найдёт элемент — вернёт `null`. Элемент физически остаётся в старом бакете и будет обнаружен только при полном обходе. После resize ситуация станет непредсказуемой: элемент окажется в "неправильном" бакете нового массива. Решение: использовать только иммутабельные ключи (`String`, `Integer`, `record`).

**Q5: Как `ConcurrentHashMap.size()` работает в Java 8 без глобальной блокировки? Является ли результат точным?**

> Java 8 использует двухуровневый счётчик: `baseCount` (обновляется через CAS при низкой конкуренции) + `counterCells[]` (массив ячеек для распределения конкуренции — идея из `LongAdder`). При высокой конкуренции на `baseCount` CAS выбирает ячейку `counterCells[threadLocalRandomProbe % len]` и инкрементирует её. `size()` суммирует `baseCount + sum(counterCells)`. Результат **не является точным** в момент конкуренции — JavaDoc явно говорит: "размер может изменяться между вызовами". Для точного подсчёта нужна внешняя синхронизация. Аналогично работает `mappingCount()` (возвращает `long`, предпочтителен для больших map).

## Связанные темы

- [[Общая иерархия коллекций]] — архитектура Collection Framework
- [[Интерфейс Set]] — уникальные ключи как Set
- [[Производительность коллекций]] — Big-O для Map операций
- [[Java Stream API]] — обработка Map через stream
- [[Корпоративная социальная сеть - микросервисная архитектура, выбор языка и идеи для вовлечённости|Корпоративная социальная сеть]] — кэширование данных в микросервисах

## 17. Заключение

`Map` — универсальный интерфейс для хранения пар ключ-значение. Реализации (`HashMap`, `LinkedHashMap`, `TreeMap`, `ConcurrentHashMap`, `WeakHashMap`) покрывают различные сценарии: от быстрого доступа и порядка вставки до сортировки, потокобезопасности и автоматической очистки. Основанные на хеш-таблицах или деревьях, они используют JVM-механизмы (CAS, AQS, GC) для эффективности. Поддержка виртуальных потоков (Java 21+) и интеграция с `CompletableFuture` делают их ещё мощнее. Следование лучшим практикам минимизирует риски и оптимизирует производительность.