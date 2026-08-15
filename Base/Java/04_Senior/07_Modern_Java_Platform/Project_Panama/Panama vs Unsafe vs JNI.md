# Panama vs Unsafe vs JNI

> Три способа работать с нативным кодом/памятью в Java: **JNI** (стандартный, но дорогой), **Unsafe** (быстрый, но опасный и непубличный), **FFM API / Panama** (быстрый, безопасный, официальный). Panama — замена обоим в долгосрочной перспективе.

## Связанные темы
[[Foreign Function and Memory API]], [[Структура памяти JVM]], [[Java Agents & Instrumentation API]]

---

## JNI (Java Native Interface)

**JNI** — официальный механизм вызова C/C++ кода из Java (и наоборот).

```java
// Java side
public class NativeLib {
    static {
        System.loadLibrary("mylib");
    }
    public native int compute(int a, int b);
}

// C side (mylib.c)
#include <jni.h>

JNIEXPORT jint JNICALL
Java_NativeLib_compute(JNIEnv* env, jobject obj, jint a, jint b) {
    return a + b;
}

// Компиляция:
// javac NativeLib.java
// javah NativeLib  (генерирует заголовочный файл)
// gcc -shared -o libmylib.so mylib.c -I$JAVA_HOME/include
```

### JNI — проблемы

```
1. Overhead (~100 нс на вызов):
   - JNI frame setup
   - GC safepoint transition (Java → Native mode)
   - Параметры: boxing/unboxing, array pinning
   
2. Сложность разработки:
   - Ручное управление памятью
   - Error-prone API (JNIEnv, local/global refs)
   - Нет bounds checking
   
3. Переносимость:
   - .so / .dll / .dylib для каждой платформы
   - Разные ABI на разных ОС/архитектурах
```

## sun.misc.Unsafe

`Unsafe` — внутренний класс JDK для low-level операций. Используется внутри JDK (AtomicXxx, ConcurrentHashMap, NIO).

```java
// Получить экземпляр (только reflection или trusted caller)
Field f = Unsafe.class.getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe unsafe = (Unsafe) f.get(null);

// --- Операции ---

// Аллокация объекта без конструктора
MyClass obj = (MyClass) unsafe.allocateInstance(MyClass.class);

// Прямой доступ к полям (обходя private/final)
long offset = unsafe.objectFieldOffset(MyClass.class.getDeclaredField("value"));
unsafe.putInt(obj, offset, 42);
int val = unsafe.getInt(obj, offset);

// Off-heap память
long address = unsafe.allocateMemory(1024);
unsafe.putInt(address, 42);
int x = unsafe.getInt(address);
unsafe.freeMemory(address);  // обязательно! нет GC

// CAS
unsafe.compareAndSwapInt(obj, offset, expected, newValue);

// Memory barriers
unsafe.loadFence();   // = LoadLoad + LoadStore
unsafe.storeFence();  // = StoreStore + LoadStore  
unsafe.fullFence();   // = все
```

### Unsafe — проблемы

```
1. Нет bounds checking → JVM crash при неверном адресе
2. Memory leaks при allocateMemory без freeMemory
3. Приватный API: нет гарантий совместимости между версиями
4. Java 9+: доступ ограничен (предупреждения, будущее удаление)
5. Segfault убивает весь JVM процесс
```

## FFM API (Panama) — правильный путь

```java
// Замена JNI для вызова нативных функций
// Замена Unsafe для off-heap памяти

// Off-heap (замена Unsafe.allocateMemory):
try (Arena arena = Arena.ofConfined()) {
    MemorySegment seg = arena.allocate(1024);
    seg.set(ValueLayout.JAVA_INT, 0, 42);
    // Нет bounds violation — автоматическая проверка
    // Нет memory leak — Arena закрывается в try-with-resources
}

// Native function call (замена JNI):
Linker linker = Linker.nativeLinker();
MethodHandle printf = linker.downcallHandle(
    linker.defaultLookup().find("printf").orElseThrow(),
    FunctionDescriptor.of(ValueLayout.JAVA_INT, ValueLayout.ADDRESS)
);
```

## Сравнительная таблица

| Характеристика | JNI | Unsafe | FFM API |
|----------------|-----|--------|---------|
| Статус | Официальный | Internal, deprecated path | Официальный (Java 22+) |
| Безопасность | Unsafe (нет bounds check) | Опасен (crash JVM) | Safe (bounds check, lifetime) |
| Производительность | ~100 нс вызов | ~нс | ~10 нс вызов |
| Простота | Сложный (C + Java) | Простой но рискованный | Умеренный |
| Off-heap memory | Через GetPrimitiveArrayCritical | allocateMemory | MemorySegment + Arena |
| Garbage Collection | Не участвует | Не участвует | Интегрирован (bounds, lifetime) |
| Совместимость | Стабильный | Нет гарантий | Стабильный (finalized) |
| Инструменты | javah | Нет | jextract |

## Когда использовать каждый

### JNI — когда нет выбора
- Легаси C-библиотека с JNI биндингами (JDBC native drivers, OpenSSL bindings)
- Код на Java < 22
- Команда знает JNI

### Unsafe — никогда в новом коде
- Только если пишешь JDK internals или low-level framework
- Всё что делает Unsafe — делает FFM API безопаснее и официальнее

### FFM API — для нового кода
- Любое взаимодействие с нативным кодом
- Off-heap память в performance-critical коде
- Инструментация нативных библиотек
- Замена DirectByteBuffer для zero-copy IO

## Migration: JNI → FFM

```java
// JNI: было
public class OldLib {
    native int process(byte[] data, int len);
    
    // C: Java_OldLib_process(JNIEnv*, jobject, jbyteArray, jint)
    // Требует: pin array, GetByteArrayElements, ReleasePrimitiveArrayCritical
}

// FFM: стало
public class NewLib {
    private static final MethodHandle PROCESS;
    
    static {
        Linker linker = Linker.nativeLinker();
        System.loadLibrary("mylib");
        PROCESS = linker.downcallHandle(
            SymbolLookup.loaderLookup().find("process_bytes").orElseThrow(),
            FunctionDescriptor.of(ValueLayout.JAVA_INT, ValueLayout.ADDRESS, ValueLayout.JAVA_INT)
        );
    }
    
    int process(byte[] data) {
        try (Arena arena = Arena.ofConfined()) {
            MemorySegment seg = arena.allocateFrom(ValueLayout.JAVA_BYTE, data);
            return (int) PROCESS.invoke(seg, data.length);
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

## Restricted Methods

В Java 22+ некоторые методы FFM API помечены `@Restricted`:

```bash
# Для использования restricted methods нужно разрешение:
--enable-native-access=com.example.mymodule
# или для unnamed module:
--enable-native-access=ALL-UNNAMED
```

Без этого флага: `IllegalCallerException` при вызове `Linker.nativeLinker()` и других.

---

## Вопросы на интервью
- В чём проблемы JNI и почему Panama их решает?
- Что делает Unsafe и почему его нельзя использовать в новом коде?
- Как Arena обеспечивает безопасность при работе с нативной памятью?
- Чем FFM API отличается от DirectByteBuffer?
- Что такое restricted methods в FFM API?

## Подводные камни
- DirectByteBuffer (NIO) — тоже off-heap, но ограничен 2 GB (int offset). FFM API поддерживает произвольные размеры
- При вызове JNI из виртуального потока (Loom) — поток может быть pinned на carrier thread. FFM downcall не имеет этой проблемы (Java 21+)
- `--enable-native-access` нужен в module-path приложениях — добавь в JVM args или module-info
