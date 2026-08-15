# Foreign Function and Memory API

> **FFM API** (Foreign Function & Memory API, JEP 454, Java 22 finalized) — официальный способ взаимодействия Java с нативным кодом и нативной памятью. Заменяет JNI и `sun.misc.Unsafe`. Безопасный, удобный, поддерживает zero-copy работу с off-heap memory.

## Связанные темы
[[Panama vs Unsafe vs JNI]], [[Vector API — SIMD в Java]], [[Finalization и Cleaner API]], [[Структура памяти JVM]]

---

## Ключевые абстракции

```
FFM API:
  ├── MemorySegment    — непрерывный блок памяти (heap или off-heap)
  ├── Arena            — управление временем жизни MemorySegment
  ├── MemoryLayout     — описание структуры данных в памяти
  ├── SegmentAllocator — аллокатор для MemorySegment
  ├── Linker           — создание вызовов к нативным функциям
  ├── FunctionDescriptor — описание сигнатуры нативной функции
  └── SymbolLookup     — поиск нативных символов в библиотеках
```

## MemorySegment — работа с памятью

```java
import java.lang.foreign.*;
import java.lang.foreign.MemoryLayout.PathElement;

// --- Off-heap аллокация ---
try (Arena arena = Arena.ofConfined()) {  // ArenaScope — владелец памяти
    
    // 1. Аллокация простого значения
    MemorySegment intSeg = arena.allocate(ValueLayout.JAVA_INT);
    intSeg.set(ValueLayout.JAVA_INT, 0, 42);
    int value = intSeg.get(ValueLayout.JAVA_INT, 0);  // 42
    
    // 2. Аллокация массива
    MemorySegment array = arena.allocate(ValueLayout.JAVA_INT, 100);
    for (int i = 0; i < 100; i++) {
        array.setAtIndex(ValueLayout.JAVA_INT, i, i * 2);
    }
    
    // 3. Аллокация структуры
    MemoryLayout pointLayout = MemoryLayout.structLayout(
        ValueLayout.JAVA_INT.withName("x"),
        ValueLayout.JAVA_INT.withName("y")
    );
    MemorySegment point = arena.allocate(pointLayout);
    
    VarHandle xHandle = pointLayout.varHandle(PathElement.groupElement("x"));
    VarHandle yHandle = pointLayout.varHandle(PathElement.groupElement("y"));
    xHandle.set(point, 0L, 10);
    yHandle.set(point, 0L, 20);
    
}  // Arena.close() → free(memory) автоматически
```

## Arena — управление временем жизни

```java
// Confined Arena — доступна только создавшему потоку, автозакрытие
try (Arena arena = Arena.ofConfined()) {
    MemorySegment seg = arena.allocate(1024);
    // seg доступен только этому потоку
}  // память освобождена

// Shared Arena — доступна нескольким потокам
try (Arena arena = Arena.ofShared()) {
    MemorySegment seg = arena.allocate(1024);
    // thread-safe доступ
}

// Auto Arena — GC-managed (освобождается когда Arena недостижима)
Arena auto = Arena.ofAuto();
MemorySegment seg = auto.allocate(1024);
// Освобождается GC — не для performance-critical кода

// Global Arena — никогда не освобождается (для static буферов)
MemorySegment staticBuf = Arena.global().allocate(1024);
```

## Вызов нативных функций

```java
import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;

// --- Пример: вызов strlen из libc ---
Linker linker = Linker.nativeLinker();

// Найти символ strlen в системных библиотеках
SymbolLookup stdlib = linker.defaultLookup();
MemorySegment strlenAddr = stdlib.find("strlen").orElseThrow();

// Описать сигнатуру: size_t strlen(const char* s)
FunctionDescriptor strlenDescriptor = FunctionDescriptor.of(
    ValueLayout.JAVA_LONG,   // return type: size_t (long)
    ValueLayout.ADDRESS      // parameter: const char* (pointer)
);

// Создать MethodHandle для вызова
MethodHandle strlen = linker.downcallHandle(strlenAddr, strlenDescriptor);

// Использование
try (Arena arena = Arena.ofConfined()) {
    MemorySegment str = arena.allocateFrom("Hello, World!");  // C-string с \0
    long len = (long) strlen.invoke(str);  // → 13
    System.out.println("strlen = " + len);
}
```

## Пример: вызов кастомной нативной библиотеки

```java
// native.h:
// int add(int a, int b);
// void fill_array(int* arr, int size, int value);

System.loadLibrary("mynative");  // или System.load("/path/to/lib.so")
SymbolLookup lib = SymbolLookup.loaderLookup();

// Вызов add(3, 4)
MethodHandle add = linker.downcallHandle(
    lib.find("add").orElseThrow(),
    FunctionDescriptor.of(ValueLayout.JAVA_INT, ValueLayout.JAVA_INT, ValueLayout.JAVA_INT)
);
int result = (int) add.invoke(3, 4);  // → 7

// Вызов fill_array(arr, 100, 42)
MethodHandle fillArray = linker.downcallHandle(
    lib.find("fill_array").orElseThrow(),
    FunctionDescriptor.ofVoid(ValueLayout.ADDRESS, ValueLayout.JAVA_INT, ValueLayout.JAVA_INT)
);

try (Arena arena = Arena.ofConfined()) {
    MemorySegment arr = arena.allocate(ValueLayout.JAVA_INT, 100);
    fillArray.invoke(arr, 100, 42);
    // arr теперь заполнен 42
}
```

## jextract — автогенерация биндингов

```bash
# Инструмент для генерации Java биндингов из C заголовочных файлов
# Доступен отдельно: https://github.com/openjdk/jextract

jextract --output src/main/java \
         -t com.example.native \
         /usr/include/sqlite3.h

# Генерирует Java классы для всех функций, структур, констант SQLite
# Использование:
import static com.example.native.sqlite3_h.*;

try (Arena arena = Arena.ofConfined()) {
    MemorySegment db = arena.allocate(ValueLayout.ADDRESS);
    sqlite3_open(arena.allocateFrom(":memory:"), db);
    // ...
}
```

## Upcalls — callback от нативного кода в Java

```java
// Нативная функция принимает callback:
// void forEach(int* arr, int size, void(*callback)(int))

// Создаём Java callback
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle javaCb = lookup.findStatic(MyClass.class, "onElement",
    MethodType.methodType(void.class, int.class));

FunctionDescriptor cbDesc = FunctionDescriptor.ofVoid(ValueLayout.JAVA_INT);

try (Arena arena = Arena.ofConfined()) {
    // Оборачиваем Java метод в нативный function pointer
    MemorySegment callback = linker.upcallStub(javaCb, cbDesc, arena);
    
    // Передаём в нативную функцию
    forEachHandle.invoke(arraySegment, 100, callback);
}

static void onElement(int value) {
    System.out.println("Element: " + value);
}
```

## Производительность

```
Операция              | JNI      | FFM API  | Unsafe
----------------------|----------|----------|-------
Off-heap read/write   | N/A      | ~ns      | ~ns
Native function call  | ~100 ns  | ~10 ns   | N/A
Bounds checking       | Нет      | Да (safe)| Нет
GC visibility         | Нет      | Да       | Нет
```

FFM API быстрее JNI за счёт отсутствия JNI frame overhead и лучшей интеграции с JIT.

---

## Вопросы на интервью
- Что такое MemorySegment и Arena?
- Как Arena управляет временем жизни нативной памяти?
- Как вызвать нативную функцию без JNI через FFM API?
- Что такое upcall stub?
- Чем FFM API лучше JNI и Unsafe?

## Подводные камни
- `Arena.ofAuto()` зависит от GC — непредсказуемое время освобождения. Для performance-critical кода используй `ofConfined()` с try-with-resources
- MemorySegment хранит bounds — доступ за пределы → `IndexOutOfBoundsException` (в отличие от Unsafe который просто corrupts memory)
- Confined Arena: доступ из другого потока → `WrongThreadException`. Для shared — `Arena.ofShared()`
- `jextract` генерирует иногда нечитаемый код — для сложных C API лучше писать биндинги вручную
