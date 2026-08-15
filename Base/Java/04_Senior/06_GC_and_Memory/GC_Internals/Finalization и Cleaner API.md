# Finalization и Cleaner API

> **`finalize()`** — метод Object, вызываемый GC перед сборкой объекта для освобождения нативных ресурсов. Deprecated Java 9, removed Java 18+. **Cleaner API** (Java 9, `java.lang.ref.Cleaner`) — замена: безопасный, non-blocking, без воскрешения объектов.

## Связанные темы
[[GC Roots и достижимость объектов]], [[Типы ссылок в Java (Reference Types)]], [[Write Barriers и Card Table]]

---

## Проблемы finalize()

```java
// ❌ Плохо: старый подход
public class NativeResource {
    private long nativePtr;
    
    @Override
    protected void finalize() throws Throwable {
        try {
            releaseNative(nativePtr);  // освобождаем нативный ресурс
        } finally {
            super.finalize();
        }
    }
}
```

**Проблемы:**

1. **Воскрешение (resurrection)**: в `finalize()` можно сохранить `this` в static поле → объект снова достижим → GC не соберёт → `finalize()` больше не будет вызван → утечка нативного ресурса

2. **Два GC-цикла**: объект с `finalize()` требует 2 цикла сборки:
   - Цикл 1: GC обнаруживает недостижимость → добавляет в Finalizer Queue → НЕ собирает
   - Finalizer Thread вызывает `finalize()`
   - Цикл 2: GC собирает объект (если не воскрес)

3. **Непредсказуемое время**: GC не гарантирует когда `finalize()` будет вызван. Может не вызваться вообще при System.exit().

4. **Блокировка Finalizer Thread**: если `finalize()` висит → очередь растёт → OOM. Finalizer Thread один и некоммутируемый.

5. **Performance**: объекты с `finalize()` всегда промоутятся в Old Gen (не могут быть собраны в Young).

## Cleaner API (Java 9+)

```java
import java.lang.ref.Cleaner;

public class NativeResource implements AutoCloseable {
    // Cleaner — статический, shared между instances
    private static final Cleaner CLEANER = Cleaner.create();
    
    private final long nativePtr;
    private final Cleaner.Cleanable cleanable;
    
    public NativeResource(long ptr) {
        this.nativePtr = ptr;
        // Регистрируем cleanup action
        // ВАЖНО: action НЕ должен держать ссылку на NativeResource (иначе утечка!)
        this.cleanable = CLEANER.register(this, new CleanupAction(ptr));
    }
    
    // Cleanup action — отдельный класс, не inner class (не держит ref на outer)
    private static class CleanupAction implements Runnable {
        private final long ptr;
        
        CleanupAction(long ptr) {
            this.ptr = ptr;
        }
        
        @Override
        public void run() {
            // Вызывается когда NativeResource стал недостижим
            releaseNative(ptr);
        }
    }
    
    @Override
    public void close() {
        cleanable.clean();  // явный вызов — немедленный и идемпотентный
    }
    
    private static native void releaseNative(long ptr);
}
```

## Как работает Cleaner под капотом

1. `Cleaner.register(obj, action)` создаёт **PhantomReference** на `obj`
2. PhantomReference помещается в `ReferenceQueue` Cleaner'а
3. Когда `obj` становится **phantom-reachable** (только через PhantomReference), GC помещает ref в очередь
4. Cleaner thread (daemon) опрашивает очередь и вызывает `action.run()`

```
NativeResource obj ──strong ref──> CleanupAction
     ↑
PhantomReference  ──added to──> ReferenceQueue
                                      ↑
                              Cleaner daemon thread polls
```

> [!INFO]
> `PhantomReference.get()` всегда возвращает `null` — это намеренно. Phantom reference не позволяет воскресить объект в отличие от `WeakReference` и `SoftReference`.

## Сравнение подходов

| Подход | Гарантия вызова | Воскрешение | Performance | Рекомендация |
|--------|----------------|-------------|-------------|--------------|
| `finalize()` | Нет | Возможно | Плохая | Не использовать |
| `Cleaner` | Нет (GC-dependent) | Невозможно | Хорошая | Safety net |
| `try-with-resources` | Да (при выходе из блока) | — | Отличная | Основной способ |
| `Runtime.addShutdownHook` | При нормальном завершении | — | — | Только для глобальных ресурсов |

## try-with-resources — правильный способ

```java
// ✅ Правильно: явное управление ресурсами
try (NativeResource r = new NativeResource(allocateNative())) {
    r.doWork();
}  // close() вызывается гарантированно, даже при исключении

// Cleaner — только safety net на случай если close() забыли вызвать
```

## DirectByteBuffer и Cleaner

`java.nio.DirectByteBuffer` использует Cleaner для освобождения off-heap памяти:

```java
// Внутри DirectByteBuffer (упрощённо):
cleaner = Cleaner.create(this, new Deallocator(base, size, cap));

// При сборке DirectByteBuffer → Cleaner вызывает Deallocator.run()
// → unsafe.freeMemory(address)
```

Именно поэтому `ByteBuffer.allocateDirect()` объекты должны быть GC'иться, иначе native memory растёт. `-XX:MaxDirectMemorySize` ограничивает этот off-heap регион.

## Диагностика утечек финализаторов

```bash
# Количество объектов в очереди финализации
jcmd <pid> VM.finalization_count

# JFR: jdk.FinalizerStatistics event
# Если queue > 1000 — проблема

# В JMX: java.lang:type=Memory → ObjectPendingFinalizationCount
```

---

## Вопросы на интервью
- Почему `finalize()` deprecated? Назовите 3 проблемы.
- Что такое "воскрешение объекта" в контексте finalization?
- Как Cleaner API работает под капотом (через какой тип Reference)?
- Почему в CleanupAction нельзя держать ссылку на сам объект?
- Как освобождается память DirectByteBuffer?

## Подводные камни
- Cleaner не гарантирует вызов cleanup — если JVM завершится без GC, action не вызовется. Основной cleanup должен быть через `close()` / try-with-resources
- `Cleaner.create()` создаёт daemon thread — нужно использовать shared instance, не создавать per-object
- Если `CleanupAction` держит ссылку на объект (например, через inner class) → объект никогда не будет phantom-reachable → Cleaner никогда не вызовется
- `System.gc()` не гарантирует финализацию → нельзя на него полагаться в тестах
