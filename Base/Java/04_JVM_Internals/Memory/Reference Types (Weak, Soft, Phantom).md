# Типы ссылок в Java (Reference Types)

> 4 типа ссылок: **Strong** (обычная, GC никогда не соберёт) → **Soft** (собирается при нехватке памяти, для кэшей) → **Weak** (собирается при следующем GC, для WeakHashMap) → **Phantom** (объект уже finalized, для cleanup). `ReferenceQueue` — механизм нотификации о сборке.
> На интервью: разница Soft vs Weak, почему `PhantomReference.get()` всегда null, `Cleaner` API vs `finalize()`, ловушки WeakHashMap.

## Связанные темы
[[Java Memory Structure]], [[Java Exceptions]]

---

## Иерархия и достижимость

```
java.lang.ref.Reference<T>
├── SoftReference<T>
├── WeakReference<T>
└── PhantomReference<T>
```

GC обрабатывает в порядке достижимости:
```
Strongly reachable   → есть strong ref от GC Root → не трогать
Softly reachable     → нет strong, есть SoftReference → при нехватке памяти
Weakly reachable     → нет strong/soft, есть WeakReference → при следующем GC
Phantom reachable    → финализирован, есть PhantomReference → для cleanup
Unreachable          → нет ничего → сразу
```

---

## Strong Reference

Обычная ссылка. Объект живёт пока есть хотя бы одна strong ссылка.

```java
Object strong = new Object();  // GC не тронет
strong = null;                 // теперь eligible for GC
```

---

## SoftReference — кэши

Собирается только при нехватке памяти (перед `OutOfMemoryError`). Идеально для кэшей.

```java
SoftReference<byte[]> cache = new SoftReference<>(new byte[1024 * 1024]);

byte[] data = cache.get();     // null если GC собрал
if (data == null) {
    data = loadData();
    cache = new SoftReference<>(data);
}
```

**HotSpot политика:** очищает SoftReference если объект не использовался >= `SoftRefLRUPolicyMSPerMB * freeHeapMB` мс. По умолчанию `-XX:SoftRefLRUPolicyMSPerMB=1000`.

```java
// Паттерн: SoftReference-кэш
public class SoftCache<K, V> {
    private final Map<K, SoftReference<V>> map = new ConcurrentHashMap<>();

    public V get(K key) {
        SoftReference<V> ref = map.get(key);
        if (ref == null) return null;
        V value = ref.get();
        if (value == null) map.remove(key); // GC собрал → убрать ключ
        return value;
    }

    public void put(K key, V value) {
        map.put(key, new SoftReference<>(value));
    }
}
```

---

## WeakReference — ассоциации без владения

Собирается при следующем GC проходе, даже если памяти достаточно.

```java
ReferenceQueue<String> queue = new ReferenceQueue<>();
WeakReference<String> weakRef = new WeakReference<>(new String("Hello"), queue);

System.out.println(weakRef.get()); // "Hello"
System.gc();
System.out.println(weakRef.get()); // null (скорее всего)
```

**WeakHashMap:**
```java
WeakHashMap<String, Integer> map = new WeakHashMap<>();
String key = new String("key");    // heap-копия, не intern()!
map.put(key, 42);

key = null;   // strong ref устранена
System.gc();
System.out.println(map.isEmpty()); // true — entry автоматически удалена
```

**Внимание:** `WeakHashMap` не работает с интернированными строками — String Pool держит strong ref.

**Паттерн: listeners без утечек:**
```java
class EventEmitter {
    private final List<WeakReference<EventListener>> listeners = new CopyOnWriteArrayList<>();

    public void addListener(EventListener listener) {
        listeners.add(new WeakReference<>(listener));
    }

    public void emit(Event event) {
        listeners.removeIf(ref -> {
            EventListener l = ref.get();
            if (l == null) return true;  // мёртвая ссылка — удалить
            l.onEvent(event);
            return false;
        });
    }
}
```

---

## PhantomReference — cleanup после финализации

`get()` **всегда** возвращает `null`. Нотификация после финализации объекта, до освобождения памяти. Используется для cleanup нативных ресурсов.

```java
ReferenceQueue<Object> refQueue = new ReferenceQueue<>();
Object resource = new Object();
PhantomReference<Object> phantom = new PhantomReference<>(resource, refQueue);

System.out.println(phantom.get()); // ВСЕГДА null

resource = null;
System.gc();

Reference<?> ref = refQueue.poll(); // нотификация от GC
if (ref != null) {
    System.out.println("Object collected — release native resources!");
}
```

**Cleaner API (Java 9+) — рекомендуемый способ:**
```java
public class NativeResource implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    private final Cleaner.Cleanable cleanable;
    private final long nativeHandle;

    public NativeResource() {
        this.nativeHandle = allocateNativeMemory();
        // cleanup action НЕ должна захватывать this!
        long handle = this.nativeHandle;
        this.cleanable = cleaner.register(this, () -> freeNativeMemory(handle));
    }

    @Override
    public void close() {
        cleanable.clean(); // детерминированный cleanup
    }

    private static void freeNativeMemory(long handle) { ... }
}
```

`Cleaner` внутри использует PhantomReference + daemon-поток для обработки ReferenceQueue.

---

## ReferenceQueue: механика

```java
// Polling (не блокирует):
Reference<? extends T> ref = queue.poll();

// Blocking (ждёт timeout):
ref = queue.remove(1000); // ms

// Infinite blocking:
ref = queue.remove();
```

**Фоновый cleanup поток:**
```java
static {
    Thread cleanerThread = new Thread(() -> {
        while (true) {
            try {
                CleanupRef ref = (CleanupRef) refQueue.remove();
                ref.cleanup();
                refs.remove(ref);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    });
    cleanerThread.setDaemon(true);
    cleanerThread.start();
}
```

**Важно:** `Reference` объекты нужно держать в `Set` — иначе сами станут мусором до того как попадут в queue.

---

## Сравнение

| Тип | GC очищает | `get()` | В ReferenceQueue | Применение |
|---|---|---|---|---|
| **Strong** | Никогда | Объект | — | Обычные переменные |
| **Soft** | При нехватке памяти | Объект или null | После очистки | Image/data кэши |
| **Weak** | При следующем GC | Объект или null | После очистки | WeakHashMap, listeners |
| **Phantom** | После финализации | Всегда null | После финализации | Cleanup нативных ресурсов |

---

## Вопросы на интервью

- Чем SoftReference отличается от WeakReference? Как GC решает когда очищать Soft?
- Почему `PhantomReference.get()` всегда возвращает `null`? Как тогда выполнять cleanup?
- Что будет если не хранить `PhantomReference` в переменной?
- Как WeakHashMap очищает устаревшие entry? Это thread-safe?
- Чем `Cleaner` API лучше `finalize()`? (3+ причины)
- Что такое "resurrection" объекта в `finalize()`? Почему это проблема?
- Почему WeakHashMap не работает с `String.intern()`?

---

## Подводные камни

- **WeakHashMap + значение ссылается на ключ** — `WeakHashMap<Key, Value>` где Value содержит strong ref на Key → ключ никогда не будет собран GC. Цикличная strong ссылка Value→Key.
- **WeakHashMap с интернированными строками** — `String.intern()` ключи в String Pool имеют strong ref из пула → entry никогда не удалится.
- **PhantomReference без ReferenceQueue** — бесполезен: `get()` всегда null, уведомление никогда не получишь.
- **Cleanup action захватывает `this`** — `cleaner.register(this, () -> this.cleanup())` — lambda держит strong ref на `this`, GC никогда не соберёт объект. Захватывай только final поля: `long h = nativeHandle; cleaner.register(this, () -> free(h))`.
- **Потеря Reference объекта** — если `WeakReference ref = new WeakReference<>(obj)` — локальная переменная вышла из scope → сам `WeakReference` стал мусором → никогда не попадёт в ReferenceQueue.
- **`finalize()` deprecated** — JDK 9 deprecated, JDK 21 — `@Deprecated(forRemoval = true)`. Используй `Cleaner` или `try-with-resources`.
- **SoftReference и GC overhead** — слишком много SoftReference объектов замедляет GC: при каждом Full GC нужно проверять все из них. Предпочти Caffeine `softValues()` кэш с лимитом по размеру.
