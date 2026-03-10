---
title: "Java Reference Types — Strong, Soft, Weak, Phantom"
tags: [java, gc, memory, weakreference, softreference, phantomreference, referencequeue]
updated: 2026-03-04
---

# Типы ссылок в Java (Reference Types)

> [!QUOTE] Суть
> 4 типа ссылок: **Strong** (обычная, GC никогда не соберёт) → **Soft** (собирается при нехватке памяти, для кэшей) → **Weak** (собирается при следующем GC, для WeakHashMap) → **Phantom** (объект уже finalized, для cleanup). `ReferenceQueue` — механизм нотификации о сборке.

> [!WARNING] Ловушка: WeakHashMap и значения-ссылки на ключ
> В `WeakHashMap<Key, Value>` если Value содержит ссылку на Key — ключ никогда не будет собран GC! Цикличная сильная ссылка Value→Key удерживает ключ. Убедись что значения не ссылаются на ключи.

Java предоставляет четыре типа ссылок с разной "силой" удержания объектов от GC. Выбор правильного типа критичен для управления памятью: избежания утечек и эффективного кэширования.

## 1. Иерархия типов ссылок

```
java.lang.ref.Reference<T>  (абстрактный)
├── SoftReference<T>
├── WeakReference<T>
└── PhantomReference<T>

StrongReference — не класс, а обычная ссылка через переменную
```

Все три `Reference` классы принимают необязательный `ReferenceQueue<T>` в конструкторе — очередь, в которую GC помещает ссылку после того, как объект собран.

## 2. Strong Reference (сильная ссылка)

- **Обычная ссылка** — `Object obj = new Object();`
- Объект **никогда не собирается GC**, пока существует хотя бы одна strong reference
- `obj = null;` — разрываем ссылку, объект становится кандидатом для GC

```java
Object strong = new Object();   // GC не тронет, пока strong != null
strong = null;                  // объект теперь eligible for GC
```

## 3. SoftReference (мягкая ссылка)

- GC **сохраняет объект пока есть свободная память**
- Удаляет только при **нехватке памяти** (перед `OutOfMemoryError`)
- Идеально для **кэшей**: объект живёт пока не нужна память

```java
import java.lang.ref.SoftReference;

SoftReference<byte[]> cache = new SoftReference<>(new byte[1024 * 1024]);

// Получить объект (может вернуть null если GC собрал его):
byte[] data = cache.get();
if (data == null) {
    data = loadData();           // перезагрузить
    cache = new SoftReference<>(data);
}
```

**Алгоритм GC для SoftReference:**
- JVM гарантирует: SoftReference будет очищена **до** `OutOfMemoryError`
- HotSpot политика: очищает SoftReference если объект не использовался >= `(-XX:SoftRefLRUPolicyMSPerMB * freeHeapMB)` миллисекунд
- По умолчанию `-XX:SoftRefLRUPolicyMSPerMB=1000` (1 секунда на каждый MB свободного heap)

**Практика: SoftReference-кэш:**

```java
public class SoftCache<K, V> {
    private final Map<K, SoftReference<V>> map = new ConcurrentHashMap<>();

    public V get(K key) {
        SoftReference<V> ref = map.get(key);
        if (ref == null) return null;
        V value = ref.get();
        if (value == null) map.remove(key); // GC собрал → убираем ключ
        return value;
    }

    public void put(K key, V value) {
        map.put(key, new SoftReference<>(value));
    }
}
```

## 4. WeakReference (слабая ссылка)

- GC собирает объект **при следующем же проходе**, если нет strong/soft ссылок
- `ref.get()` может вернуть `null` в любой момент после GC
- Используется для: `WeakHashMap`, слушателей событий, ассоциаций без владения

```java
import java.lang.ref.WeakReference;
import java.lang.ref.ReferenceQueue;

ReferenceQueue<String> queue = new ReferenceQueue<>();
WeakReference<String> weakRef = new WeakReference<>(new String("Hello"), queue);

System.out.println(weakRef.get()); // "Hello"
System.gc();
System.out.println(weakRef.get()); // null (скорее всего)

// После GC ссылка попадает в queue:
Reference<?> polled = queue.poll(); // WeakReference объект
```

**WeakHashMap — как работает:**

```java
WeakHashMap<String, Integer> map = new WeakHashMap<>();
String key = new String("key");  // не интернирован! нужна heap-копия
map.put(key, 42);

key = null;  // убираем strong reference на ключ
System.gc();
// GC собрал ключ → WeakHashMap автоматически удаляет entry
System.out.println(map.isEmpty()); // true (после GC)
```

**Внимание**: `WeakHashMap` не работает с интернированными строками (`String.intern()`), так как они живут в String Pool — всегда есть strong reference.

**Паттерн: слабые слушатели (listeners без утечек):**

```java
public class EventEmitter {
    private final List<WeakReference<EventListener>> listeners = new CopyOnWriteArrayList<>();

    public void addListener(EventListener listener) {
        listeners.add(new WeakReference<>(listener));
    }

    public void emit(Event event) {
        listeners.removeIf(ref -> {
            EventListener l = ref.get();
            if (l == null) return true;  // удаляем мёртвые ссылки
            l.onEvent(event);
            return false;
        });
    }
}
```

## 5. PhantomReference (фантомная ссылка)

- **Самая слабая** — `get()` всегда возвращает `null`
- Объект помещается в `ReferenceQueue` **после финализации**, но **до** освобождения памяти
- Цель: детектировать момент, когда объект **полностью собран** GC (для cleanup)

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;

ReferenceQueue<Object> refQueue = new ReferenceQueue<>();
Object resource = new Object();
PhantomReference<Object> phantom = new PhantomReference<>(resource, refQueue);

System.out.println(phantom.get()); // ВСЕГДА null

resource = null;  // strong reference устранена
System.gc();

// Ждём уведомления GC:
Reference<?> ref = refQueue.poll(); // PhantomReference объект
if (ref != null) {
    System.out.println("Object collected — release native resources!");
    ref.clear(); // освобождаем phantom reference
}
```

**Практика: управление нативными ресурсами (Cleaner API, Java 9+):**

```java
import java.lang.ref.Cleaner;

public class NativeResource implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    private final Cleaner.Cleanable cleanable;
    private final long nativeHandle;

    public NativeResource() {
        this.nativeHandle = allocateNativeMemory();
        // Регистрируем cleanup action (не должна захватывать this!)
        this.cleanable = cleaner.register(this, () -> freeNativeMemory(nativeHandle));
    }

    @Override
    public void close() {
        cleanable.clean(); // детерминированный cleanup
    }

    private static long allocateNativeMemory() { return 42L; }
    private static void freeNativeMemory(long handle) {
        System.out.println("Freeing native memory: " + handle);
    }
}

// Использование:
try (NativeResource res = new NativeResource()) {
    // работаем
} // автоматически вызовет clean()
```

`Cleaner` внутри использует `PhantomReference` + daemon-поток для обработки `ReferenceQueue`.

## 6. ReferenceQueue: механика

`ReferenceQueue<T>` — очередь, в которую GC помещает `Reference` объекты после сборки их референта:

- **WeakReference/SoftReference**: помещаются в queue **после** сборки референта (референт `null`)
- **PhantomReference**: помещается после **финализации**, поле `referent` уже очищено

```java
// Polling (не блокирует):
Reference<? extends MyObject> ref = queue.poll();

// Blocking (ждёт до timeout):
ref = queue.remove(1000); // ждать до 1 секунды

// Infinite blocking:
ref = queue.remove(); // блокирует навсегда
```

**Паттерн: фоновый поток для cleanup:**

```java
public class ResourceTracker {
    private static final ReferenceQueue<Object> refQueue = new ReferenceQueue<>();
    private static final Set<CleanupRef> refs = Collections.synchronizedSet(new HashSet<>());

    // Запускаем daemon-поток для обработки очереди
    static {
        Thread cleaner = new Thread(() -> {
            while (true) {
                try {
                    CleanupRef ref = (CleanupRef) refQueue.remove();
                    ref.cleanup();  // вызываем cleanup action
                    refs.remove(ref);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        cleaner.setDaemon(true);
        cleaner.start();
    }

    static class CleanupRef extends PhantomReference<Object> {
        private final Runnable action;
        CleanupRef(Object referent, Runnable action) {
            super(referent, refQueue);
            this.action = action;
        }
        void cleanup() { action.run(); }
    }

    public static void register(Object obj, Runnable cleanup) {
        refs.add(new CleanupRef(obj, cleanup)); // держим CleanupRef в Set!
    }
}
```

**Важно**: `PhantomReference`/`WeakReference` объекты нужно держать в `Set` или другой структуре — иначе сами станут мусором до того, как попадут в queue.

## 7. Сравнение всех типов

|Тип|GC очищает когда|`get()` возвращает|В ReferenceQueue после|Применение|
|---|---|---|---|---|
|**Strong**|Никогда (пока есть ссылка)|Объект всегда|N/A|Обычные переменные|
|**Soft**|При нехватке памяти|Объект или `null`|После очистки объекта|Кэши (image cache, data cache)|
|**Weak**|При следующем Minor/Major GC|Объект или `null`|После очистки объекта|WeakHashMap, listeners, каноническое отображение|
|**Phantom**|После финализации|Всегда `null`|После финализации (до free)|Cleanup нативных ресурсов|

## 8. Достижимость объекта (Reachability)

GC определяет "жизнь" объекта по его достижимости:

```
Strongly reachable   → есть strong reference от GC Root
Softly reachable     → нет strong, но есть SoftReference
Weakly reachable     → нет strong/soft, но есть WeakReference
Phantom reachable    → финализирован, есть PhantomReference
Unreachable          → нет никаких ссылок → сразу для освобождения
```

GC обрабатывает в этом порядке: сначала softly reachable (при нужде), потом weakly reachable (всегда), потом phantom reachable (для cleanup).

## 9. Подводные камни

**Проблема 1: `WeakHashMap` с примитивными обёртками**
```java
WeakHashMap<Integer, String> map = new WeakHashMap<>();
map.put(1000, "value");   // Integer.valueOf(1000) — не кэшируется JVM
// Если никто не держит ссылку на этот Integer — GC соберёт ключ!
// Integer.valueOf(-128..127) кэшируются → safe, но это деталь JVM
```

**Проблема 2: PhantomReference без ReferenceQueue**
```java
PhantomReference<Object> ref = new PhantomReference<>(obj, null);
// Бесполезно! Без queue нельзя получить уведомление о сборке
```

**Проблема 3: Захват `this` в cleanup action**
```java
// НЕПРАВИЛЬНО — lambda захватывает this, мешая GC собрать объект:
cleaner.register(this, () -> this.cleanup());

// ПРАВИЛЬНО — lambda без захвата this:
long handle = this.nativeHandle;
cleaner.register(this, () -> NativeResource.freeHandle(handle));
```

## Interview Q&A (Senior Level)

**Q1: Чем SoftReference отличается от WeakReference с точки зрения поведения GC?**

> **WeakReference**: GC очищает при следующем проходе, как только нет strong/soft ссылок. Это происходит даже при достаточном количестве памяти — GC не щадит weak references. **SoftReference**: GC очищает только при нехватке памяти, перед `OutOfMemoryError`. HotSpot использует политику: SoftReference не очищается, если объект был доступен в последние `SoftRefLRUPolicyMSPerMB * freeMB` миллисекунд. Итог: SoftReference для кэшей (объект живёт пока есть память), WeakReference для ассоциаций без владения (очищается при первой возможности).

**Q2: Почему `PhantomReference.get()` всегда возвращает `null`? Как тогда выполнять cleanup?**

> По дизайну: PhantomReference сигнализирует о факте сборки объекта, не предоставляя доступа к нему (объект уже финализирован). Если бы `get()` возвращал ссылку, можно было бы "воскресить" объект через strong reference — JVM пришлось бы запускать финализацию заново. Cleanup выполняется через `ReferenceQueue`: фоновый поток вызывает `queue.remove()`, получает `PhantomReference` объект (не референт!), и выполняет заранее сохранённое cleanup action. Java 9+ `Cleaner` API делает это прозрачно.

**Q3: Что произойдёт если не хранить `PhantomReference` в переменной после регистрации?**

> `PhantomReference` — это просто Java-объект. Если на него нет strong reference, GC соберёт **сам объект `PhantomReference`** до того, как референт будет собран. В результате cleanup не произойдёт — ссылка никогда не попадёт в `ReferenceQueue`. Всегда храните `Reference` объекты в `Set`, `List` или поле — пока жив мониторируемый объект или пока не выполнен cleanup.

**Q4: Как WeakHashMap очищает устаревшие entry? Является ли это атомарной операцией?**

> `WeakHashMap` хранит ключи как `WeakReference`. Когда GC собирает ключ, соответствующий `WeakReference` помещается в `ReferenceQueue`, связанную с картой. При следующем вызове любого метода карты (`get`, `put`, `size`, `containsKey`) вызывается внутренний метод `expungeStaleEntries()`, который опрашивает queue и удаляет соответствующие entry. Это **не атомарно и не потокобезопасно** — `WeakHashMap` не является thread-safe. Для потокобезопасного варианта нужен `Collections.synchronizedMap(new WeakHashMap<>())` или Caffeine/Guava `CacheBuilder.weakKeys()`.

**Q5: В чём преимущество `Cleaner` API (Java 9+) перед `finalize()`?**

> `finalize()` имеет критические проблемы: (1) непредсказуемый вызов — может не вызваться вообще; (2) "воскрешение" объекта через strong reference в `finalize()` задерживает GC на дополнительный цикл; (3) GC обрабатывает финализаторы через специальный `FinalizerThread` — если он перегружен, объекты накапливаются. `Cleaner` API (основан на PhantomReference): (1) cleanup вызывается **после** финализации, не может воскресить объект; (2) детерминированный вызов через `Cleanable.clean()` (аналог `close()`); (3) не зависит от FinalizerThread — использует собственный daemon-поток; (4) `finalize()` deprecated с Java 9, removed в Java 21 (`--enable-preview`).

## Связанные темы

- [[Java Memory Structure]] — GC алгоритмы, достижимость, Metaspace
- [[Интерфейс Map]] — WeakHashMap детали
- [[Java Exceptions]] — OutOfMemoryError
