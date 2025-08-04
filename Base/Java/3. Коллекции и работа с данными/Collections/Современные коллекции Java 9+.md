# Современные коллекции Java 9+

Java 9 и последующие версии принесли значительные улучшения в фреймворк коллекций, включая неизменяемые коллекции, новые методы и улучшенную производительность. Эта статья рассматривает современные возможности коллекций Java.

## 1. Неизменяемые коллекции (Immutable Collections)

### 1.1. Создание неизменяемых коллекций

Java 9+ предоставляет фабричные методы для создания неизменяемых коллекций:

```java
// Неизменяемые списки
List<String> immutableList = List.of("a", "b", "c");
List<String> emptyList = List.of();

// Неизменяемые множества
Set<String> immutableSet = Set.of("a", "b", "c");
Set<Integer> numbers = Set.of(1, 2, 3, 4, 5);

// Неизменяемые карты
Map<String, Integer> immutableMap = Map.of("a", 1, "b", 2, "c", 3);
Map<String, String> emptyMap = Map.of();

// Карты с большим количеством элементов
Map<String, Integer> largeMap = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3),
    Map.entry("d", 4),
    Map.entry("e", 5)
);
```

### 1.2. Преимущества неизменяемых коллекций

```java
// Потокобезопасность
List<String> threadSafe = List.of("a", "b", "c");
// Можно безопасно передавать между потоками

// Предсказуемость
Set<String> predictable = Set.of("a", "b", "c");
// Состояние никогда не изменится

// Производительность
List<String> optimized = List.of("a", "b", "c");
// Нет накладных расходов на синхронизацию
```

### 1.3. Ограничения неизменяемых коллекций

```java
List<String> immutable = List.of("a", "b", "c");

// ❌ Нельзя добавлять элементы
// immutable.add("d"); // UnsupportedOperationException

// ❌ Нельзя удалять элементы
// immutable.remove("a"); // UnsupportedOperationException

// ❌ Нельзя изменять элементы
// immutable.set(0, "x"); // UnsupportedOperationException

// ✅ Можно создавать изменяемые копии
List<String> mutable = new ArrayList<>(immutable);
mutable.add("d"); // Работает
```

## 2. Новые методы в существующих коллекциях

### 2.1. Методы для работы с коллекциями

#### `List.of()` и `Set.of()`

```java
// Создание неизменяемых коллекций
List<String> list = List.of("a", "b", "c");
Set<Integer> set = Set.of(1, 2, 3);

// Создание пустых коллекций
List<String> emptyList = List.of();
Set<String> emptySet = Set.of();
```

#### `Map.of()` и `Map.ofEntries()`

```java
// Создание неизменяемых карт
Map<String, Integer> map = Map.of("a", 1, "b", 2, "c", 3);

// Для большего количества элементов
Map<String, Integer> largeMap = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    Map.entry("c", 3),
    Map.entry("d", 4),
    Map.entry("e", 5),
    Map.entry("f", 6),
    Map.entry("g", 7),
    Map.entry("h", 8),
    Map.entry("i", 9),
    Map.entry("j", 10)
);
```

### 2.2. Новые методы в изменяемых коллекциях

#### `List.copyOf()` и `Set.copyOf()`

```java
// Создание неизменяемых копий
List<String> original = Arrays.asList("a", "b", "c");
List<String> immutableCopy = List.copyOf(original);

Set<String> originalSet = new HashSet<>(Arrays.asList("a", "b", "c"));
Set<String> immutableSetCopy = Set.copyOf(originalSet);
```

#### `Map.copyOf()`

```java
Map<String, Integer> original = new HashMap<>();
original.put("a", 1);
original.put("b", 2);

Map<String, Integer> immutableCopy = Map.copyOf(original);
```

## 3. Улучшенные методы в существующих коллекциях

### 3.1. Новые методы в `List`

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c", "d"));

// Java 9: of() - создание неизменяемого списка
List<String> immutable = List.of("a", "b", "c");

// Java 10: copyOf() - создание неизменяемой копии
List<String> copy = List.copyOf(list);

// Java 11: isEmpty() - проверка на пустоту
if (list.isEmpty()) {
    System.out.println("Список пуст");
}
```

### 3.2. Новые методы в `Set`

```java
Set<String> set = new HashSet<>(Arrays.asList("a", "b", "c"));

// Java 9: of() - создание неизменяемого множества
Set<String> immutable = Set.of("a", "b", "c");

// Java 10: copyOf() - создание неизменяемой копии
Set<String> copy = Set.copyOf(set);
```

### 3.3. Новые методы в `Map`

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);

// Java 9: of() и ofEntries() - создание неизменяемых карт
Map<String, Integer> immutable = Map.of("a", 1, "b", 2);

// Java 10: copyOf() - создание неизменяемой копии
Map<String, Integer> copy = Map.copyOf(map);

// Java 8+: getOrDefault() - безопасное получение значения
int value = map.getOrDefault("c", 0);

// Java 8+: putIfAbsent() - добавление только если ключ отсутствует
map.putIfAbsent("c", 3);

// Java 8+: computeIfAbsent() - вычисление значения если ключ отсутствует
map.computeIfAbsent("d", k -> k.length());

// Java 8+: computeIfPresent() - вычисление значения если ключ присутствует
map.computeIfPresent("a", (k, v) -> v * 2);

// Java 8+: merge() - объединение значений
map.merge("a", 10, Integer::sum);
```

## 4. Специализированные коллекции

### 4.1. `Optional` в коллекциях

```java
// Безопасная работа с коллекциями
List<String> list = Arrays.asList("a", "b", "c");

// Java 8+: findFirst() с Optional
Optional<String> first = list.stream().findFirst();
first.ifPresent(System.out::println);

// Java 8+: max() с Optional
Optional<String> max = list.stream().max(String::compareTo);
max.ifPresent(System.out::println);
```

### 4.2. Специализированные потоки

```java
// IntStream для примитивов
IntStream.range(0, 10)
    .filter(i -> i % 2 == 0)
    .forEach(System.out::println);

// LongStream для больших чисел
LongStream.range(0, 1000000)
    .parallel()
    .filter(i -> i % 2 == 0)
    .count();

// DoubleStream для дробных чисел
DoubleStream.of(1.1, 2.2, 3.3, 4.4, 5.5)
    .average()
    .ifPresent(System.out::println);
```

## 5. Производительность и оптимизация

### 5.1. Оптимизации в Java 9+

```java
// Компактные строки (Java 9+)
String compact = "Hello, World!";
// Меньше памяти для строк с латинскими символами

// Улучшенная производительность G1GC
// Автоматическая оптимизация для коллекций

// Оптимизированные неизменяемые коллекции
List<String> optimized = List.of("a", "b", "c");
// Специальные реализации для небольших коллекций
```

### 5.2. Рекомендации по производительности

```java
// Используйте неизменяемые коллекции для констант
private static final List<String> CONSTANTS = List.of("a", "b", "c");

// Предварительное выделение емкости
List<String> list = new ArrayList<>(1000);

// Используйте специализированные потоки
IntStream.range(0, 1000)
    .filter(i -> i % 2 == 0)
    .sum(); // Быстрее чем Stream<Integer>
```

## 6. Практические примеры

### 6.1. Создание конфигурации

```java
// Неизменяемая конфигурация
public class AppConfig {
    private static final Map<String, String> CONFIG = Map.of(
        "db.url", "jdbc:postgresql://localhost:5432/mydb",
        "db.user", "admin",
        "db.password", "secret",
        "app.port", "8080",
        "app.host", "localhost"
    );
    
    public static String getConfig(String key) {
        return CONFIG.getOrDefault(key, "");
    }
    
    public static Set<String> getConfigKeys() {
        return Set.copyOf(CONFIG.keySet());
    }
}
```

### 6.2. Валидация данных

```java
public class DataValidator {
    private static final Set<String> VALID_STATUSES = Set.of(
        "ACTIVE", "INACTIVE", "PENDING", "SUSPENDED"
    );
    
    private static final List<String> REQUIRED_FIELDS = List.of(
        "id", "name", "email", "status"
    );
    
    public static boolean isValidStatus(String status) {
        return VALID_STATUSES.contains(status);
    }
    
    public static boolean hasAllRequiredFields(Map<String, Object> data) {
        return REQUIRED_FIELDS.stream()
            .allMatch(data::containsKey);
    }
}
```

### 6.3. Кэширование

```java
public class CacheManager {
    private static final Map<String, Object> CACHE = new ConcurrentHashMap<>();
    
    public static <T> T getOrCompute(String key, Supplier<T> supplier) {
        return (T) CACHE.computeIfAbsent(key, k -> supplier.get());
    }
    
    public static void clearCache() {
        CACHE.clear();
    }
    
    public static Set<String> getCacheKeys() {
        return Set.copyOf(CACHE.keySet());
    }
}
```

## 7. Миграция с Java 8

### 7.1. Замена устаревших подходов

```java
// ❌ Старый способ создания списков
List<String> oldWay = Arrays.asList("a", "b", "c");

// ✅ Новый способ (Java 9+)
List<String> newWay = List.of("a", "b", "c");

// ❌ Старый способ создания множеств
Set<String> oldSet = new HashSet<>(Arrays.asList("a", "b", "c"));

// ✅ Новый способ (Java 9+)
Set<String> newSet = Set.of("a", "b", "c");

// ❌ Старый способ создания карт
Map<String, Integer> oldMap = new HashMap<>();
oldMap.put("a", 1);
oldMap.put("b", 2);

// ✅ Новый способ (Java 9+)
Map<String, Integer> newMap = Map.of("a", 1, "b", 2);
```

### 7.2. Обновление существующего кода

```java
// Обновление констант
public class Constants {
    // ❌ Старый способ
    public static final List<String> OLD_CONSTANTS = 
        Collections.unmodifiableList(Arrays.asList("a", "b", "c"));
    
    // ✅ Новый способ
    public static final List<String> NEW_CONSTANTS = List.of("a", "b", "c");
}

// Обновление фабричных методов
public class DataFactory {
    // ❌ Старый способ
    public static List<String> createList() {
        return Collections.unmodifiableList(Arrays.asList("a", "b", "c"));
    }
    
    // ✅ Новый способ
    public static List<String> createList() {
        return List.of("a", "b", "c");
    }
}
```

## 8. Лучшие практики

### 8.1. Выбор между изменяемыми и неизменяемыми коллекциями

```java
// Используйте неизменяемые коллекции для:
// - Констант и конфигурации
// - Возвращаемых значений API
// - Данных, передаваемых между потоками
// - Кэшированных данных

private static final Set<String> VALID_ROLES = Set.of("ADMIN", "USER", "GUEST");

// Используйте изменяемые коллекции для:
// - Временных данных
// - Коллекций, которые нужно модифицировать
// - Буферов и накопителей

List<String> buffer = new ArrayList<>();
buffer.add("item1");
buffer.add("item2");
```

### 8.2. Производительность

```java
// Предпочитайте неизменяемые коллекции для небольших данных
List<String> small = List.of("a", "b", "c"); // Оптимизировано

// Используйте изменяемые коллекции для больших данных
List<String> large = new ArrayList<>(10000);

// Используйте специализированные потоки для примитивов
IntStream.range(0, 1000).sum(); // Быстрее
// vs
Stream.of(1, 2, 3).mapToInt(Integer::intValue).sum(); // Медленнее
```

## 9. Вопросы для собеседования

### Базовые вопросы
1. **В чем разница между `List.of()` и `Arrays.asList()`?**
2. **Можно ли изменить коллекцию, созданную через `List.of()`?**
3. **Когда использовать неизменяемые коллекции?**
4. **Как создать неизменяемую копию существующей коллекции?**
5. **В чем преимущества `Map.of()` перед конструктором `HashMap`?**

### Продвинутые вопросы
1. **Как работает внутренняя реализация неизменяемых коллекций?**
2. **Какие оптимизации применяются к неизменяемым коллекциям?**
3. **Как мигрировать существующий код на новые коллекции?**
4. **Какие проблемы могут возникнуть при использовании неизменяемых коллекций?**
5. **Как тестировать код с неизменяемыми коллекциями?**

## 10. Дополнительные ресурсы

### Официальная документация
- [Java 9 Collections Factory Methods](https://docs.oracle.com/javase/9/core/collections-factory-methods.htm)
- [Java 10 Local Variable Type Inference](https://docs.oracle.com/javase/10/language/toc.htm)
- [Java 11 Features](https://docs.oracle.com/en/java/javase/11/)

### Книги
- "Modern Java in Action" by Raoul-Gabriel Urma
- "Effective Java" by Joshua Bloch (3rd Edition)
- "Java 9 Modularity" by Sander Mak

### Онлайн ресурсы
- [Baeldung - Java 9 Collections](https://www.baeldung.com/java-9-collections-factory-methods)
- [Oracle Blog - Immutable Collections](https://blogs.oracle.com/javamagazine/immutable-collections-in-java-9)
- [Java Code Geeks - Java 9 Features](https://www.javacodegeeks.com/java-9-features.html)

---

**Примечание**: Современные коллекции Java 9+ предоставляют более безопасный и эффективный способ работы с данными. Изучите эти возможности для написания более качественного кода. 