# ThreadLocal

> `ThreadLocal<T>` — переменная с отдельной копией значения для каждого потока: `get()`/`set()` в потоке A не видны потоку B. Хранится не в самом `ThreadLocal`, а в `ThreadLocalMap` внутри объекта `Thread`.
> На интервью: как устроена `ThreadLocalMap`, почему это классический источник утечек памяти в пулах потоков, чем `InheritableThreadLocal` опасен, почему с виртуальными потоками картина меняется и когда вместо него нужен `ScopedValue`.

## Связанные темы
[[Scoped Values (Java 21, JEP 446)]], [[Virtual Threads — модель и архитектура]], [[Happens-Before в контексте Virtual Threads]], [[ThreadPool, Future, Callable, Executors, CompletableFuture]], [[Java Memory Structure]]

---

## Устройство: ThreadLocalMap живёт в Thread, не в ThreadLocal

Интуиция «данные хранятся внутри объекта `ThreadLocal`» — неверна. На самом деле:

```
Thread
  └── ThreadLocalMap threadLocals   (создаётся лениво, при первом set())
        Entry[] table
          Entry(key = WeakReference<ThreadLocal<?>>, value = Object)  // value — СИЛЬНАЯ ссылка
          Entry(key = WeakReference<ThreadLocal<?>>, value = Object)
          ...
```

- Каждый `Thread` содержит собственную `ThreadLocalMap` — отдельную хеш-таблицу с open addressing (не `HashMap`).
- Ключ записи — `WeakReference` на сам объект `ThreadLocal`. Значение — обычная (сильная) ссылка на данные.
- `threadLocal.get()`/`set(v)` под капотом — это `Thread.currentThread().threadLocals.get(this)` / `.set(this, v)`, где `this` — объект `ThreadLocal`, используемый как ключ.

```java
public class ThreadLocal<T> {
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);           // map лежит внутри Thread
        if (map != null) {
            Entry e = map.getEntry(this);          // this = ThreadLocal-объект как ключ
            if (e != null) return (T) e.value;
        }
        return setInitialValue();                  // initialValue() или null
    }

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) map.set(this, value);
        else createMap(t, value);
    }
}
```

---

## API

```java
// Базовое объявление — как static final, одно на всё приложение:
private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd")); // ленивая инициализация на поток

DATE_FORMAT.get();      // получить значение текущего потока (или initialValue())
DATE_FORMAT.set(value); // задать значение для текущего потока
DATE_FORMAT.remove();   // удалить Entry — ОБЯЗАТЕЛЬНО перед возвратом потока в пул
```

**Классический пример — non-thread-safe объект, переиспользуемый на поток:**
```java
class DateUtils {
    private static final ThreadLocal<SimpleDateFormat> FORMATTER =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("dd.MM.yyyy"));

    static String format(Date date) {
        return FORMATTER.get().format(date); // свой SimpleDateFormat на каждый поток — без синхронизации
    }
}
```

---

## Утечки памяти: почему это опасно именно в пулах потоков

Ключ `Entry` — `WeakReference<ThreadLocal<?>>`, значение — сильная ссылка. Если внешняя ссылка на `ThreadLocal` пропадает (например, локальная переменная вышла из области видимости), GC соберёт ключ — `Entry.key` станет `null`. Но **значение** (`Entry.value`) всё ещё сильно связано с `Entry`, которая живёт в `table` до следующей операции над картой (`get`/`set`/`remove` с очисткой stale-записей).

```
Пока поток жив → ThreadLocalMap жив → все "осиротевшие" (key=null) Entry
держат value в памяти, пока их не вычистит рехеширование или явный remove()
```

**Почему это особенно опасно в `ExecutorService`-пуле:** платформенные потоки в пуле переживают множество задач. Если задача делает `set()` и не делает `remove()`, значение (и весь граф объектов, на которые оно ссылается — коннекшн, сессия, `ClassLoader` в app-сервере) утекает на весь срок жизни потока в пуле, то есть фактически навсегда.

```java
// УТЕЧКА:
void handleRequest(Connection conn) {
    CONNECTION_TL.set(conn);
    process();
    // забыли remove() → conn "жив" в потоке пула до следующего использования этого ThreadLocal в этом потоке
}

// ПРАВИЛЬНО:
void handleRequest(Connection conn) {
    CONNECTION_TL.set(conn);
    try {
        process();
    } finally {
        CONNECTION_TL.remove(); // обязательно, даже при исключении
    }
}
```

> [!WARNING] Утечка ClassLoader в app-серверах (Tomcat и т.п.)
> Если `ThreadLocal` объявлен в классе, загруженном web-приложением, а поток принадлежит контейнеру (переживает redeploy), `Entry.value` может держать весь `ClassLoader` приложения — классическая причина `OutOfMemoryError: Metaspace` после нескольких redeploy без перезапуска сервера.

---

## InheritableThreadLocal

Копирует значения родительского потока в дочерний **в момент создания** дочернего потока (не «живая» связь):

```java
InheritableThreadLocal<String> CONTEXT = new InheritableThreadLocal<>();

CONTEXT.set("parent-value");
new Thread(() -> {
    System.out.println(CONTEXT.get()); // "parent-value" — скопировано при Thread.start()
}).start();

CONTEXT.set("changed"); // дочерний поток этого уже не увидит — копия сделана один раз
```

- Копирование — `childValue(parentValue)` (по умолчанию просто возвращает ту же ссылку; можно переопределить для глубокого копирования).
- **Дорого при массовом создании потоков**: каждый новый поток копирует всю `ThreadLocalMap` родителя. Для `Thread.ofVirtual()` — это O(n) на каждый virtual thread, при миллионах VT деградирует ощутимо (подробнее — [[Virtual Threads — модель и архитектура]], [[Happens-Before в контексте Virtual Threads]]).

---

## ThreadLocal и потокобезопасность значения

`ThreadLocal` гарантирует, что *ссылка* на значение не расшарена между потоками — но если само значение мутабельно и кто-то передаст ссылку на него в другой поток вручную, потокобезопасность объекта из-под `ThreadLocal` теряется. `ThreadLocal` — про изоляцию по потокам, не про immutability данных внутри.

---

## ThreadLocal и Virtual Threads

`ThreadLocal` продолжает работать корректно с виртуальными потоками — он привязан к объекту `Thread` (в том числе VT), а не к carrier thread, поэтому смена carrier при перемонтировании не нарушает видимость (детали — [[Happens-Before в контексте Virtual Threads]]). Проблема не в корректности, а в стоимости:

- Каждый VT — отдельный `Thread` со своей `ThreadLocalMap`. При платформенных потоках их были тысячи — при VT могут быть миллионы. Если каждый ThreadLocal хранит нетривиальный объект (буфер, соединение), это миллионы одновременно живущих объектов.
- `InheritableThreadLocal` умножает проблему — O(n)-копирование карты на каждый форк VT.

Рекомендуемая замена для VT-кода — [[Scoped Values (Java 21, JEP 446)]]: иммутабельный, без `remove()`, с O(1) наследованием в `StructuredTaskScope.fork()`. Полная сравнительная таблица `ThreadLocal` vs `ScopedValue` — там же.

---

## Вопросы на интервью

- Где физически хранятся значения `ThreadLocal`? Почему не в самом объекте `ThreadLocal`?
- Почему ключ в `ThreadLocalMap` — `WeakReference`, а значение — нет? Что это значит для утечек?
- Почему `ThreadLocal` в пуле потоков особенно опасен? Как правильно его использовать с `ExecutorService`?
- Чем `InheritableThreadLocal` отличается от обычного `ThreadLocal`? Когда копируется значение?
- Работает ли `ThreadLocal` с виртуальными потоками? В чём тогда проблема?
- Когда вместо `ThreadLocal` стоит использовать `ScopedValue`, а когда — нет?

## Подводные камни

- **Забытый `remove()` в пуле потоков** — главный источник утечек. Всегда `set()` → `try { ... } finally { remove(); }`.
- **`WeakReference` на ключ не спасает от утечки value** — пока поток жив, `Entry.value` держится сильной ссылкой, даже если сам `ThreadLocal` уже недостижим извне.
- **`static final ThreadLocal`, но нестатичный populate** — если `ThreadLocal` создаётся не как `static final`, а как поле объекта, каждый новый экземпляр объекта создаёт новый ключ — старые записи в `ThreadLocalMap` потоков превращаются в мусор с той же WeakReference-проблемой.
- **`InheritableThreadLocal` в пуле потоков** — копирование происходит один раз при `Thread.start()`, а не при каждой задаче. В `ExecutorService` (поток переиспользуется) значения из первой задачи могут "утечь" в следующую, если явно не переустановлены.
- **1M virtual threads + тяжёлый `ThreadLocal`** — работает корректно, но по памяти может быть хуже, чем ожидалось. Профилируй, прежде чем переносить blocking-код на VT без ревизии `ThreadLocal`-использования.
