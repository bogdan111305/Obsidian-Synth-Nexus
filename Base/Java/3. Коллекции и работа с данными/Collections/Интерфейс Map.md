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

## 14. Заключение

`Map` — универсальный интерфейс для хранения пар ключ-значение. Реализации (`HashMap`, `LinkedHashMap`, `TreeMap`, `ConcurrentHashMap`, `WeakHashMap`) покрывают различные сценарии: от быстрого доступа и порядка вставки до сортировки, потокобезопасности и автоматической очистки. Основанные на хеш-таблицах или деревьях, они используют JVM-механизмы (CAS, AQS, GC) для эффективности. Поддержка виртуальных потоков (Java 21+) и интеграция с `CompletableFuture` делают их ещё мощнее. Следование лучшим практикам минимизирует риски и оптимизирует производительность.