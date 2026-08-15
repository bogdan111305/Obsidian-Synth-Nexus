# GC Roots и достижимость объектов

> **GC Root** — "якорный" объект, который GC считает живым по определению. Объект достижим (живой), если от любого GC Root существует цепочка ссылок до него. Всё недостижимое — мусор.

## Связанные темы
[[Write Barriers и Card Table]], [[Safepoints и Stop-The-World]], [[Finalization и Cleaner API]], [[Типы ссылок в Java (Reference Types)]], [[Структура памяти JVM]]

---

## Типы GC Roots

| Root | Описание |
|------|----------|
| **Thread stacks** | Локальные переменные и параметры методов в каждом живом потоке |
| **Static fields** | Все non-null static поля загруженных классов |
| **JNI References** | Глобальные (`NewGlobalRef`) и локальные JNI handles |
| **System classes** | `java.lang.*` и другие bootstrap-класслоадерные классы |
| **Synchronized monitors** | Объекты, на которых кто-то сейчас `synchronized` |
| **JVM internals** | Interned strings, class loaders, exception table refs |
| **Finalizer queue** | Объекты, ожидающие finalization (f-reachable) |

## Mark-and-Sweep алгоритм

```
Phase 1 — Mark (требует знания всех GC Roots):
  worklist = all GC Roots
  while worklist not empty:
      obj = worklist.pop()
      if not obj.marked:
          obj.marked = true
          for each ref in obj.fields:
              if ref != null: worklist.push(ref)

Phase 2 — Sweep:
  for each object in heap:
      if not obj.marked:
          free(obj)
      else:
          obj.marked = false  // reset для следующего цикла
```

## Граф достижимости

```
GC Roots
  ├── Thread-1 stack
  │     └── localVar → [Obj A] → [Obj B]
  │                         └── [Obj C]
  ├── Static field
  │     └── MyCache.instance → [Cache] → [Obj D]
  └── JNI global ref
        └── → [Obj E]

[Obj F]  ← недостижим, будет собран
[Obj G]  ← [Obj G] → [Obj H] → [Obj G]  ← циклические, но недостижимы → будут собраны
```

> [!INFO]
> Java GC не страдает от циклических ссылок (в отличие от reference counting в Python). Круговая ссылка A→B→A будет собрана, если ни A, ни B не достижимы из GC Root.

## Виды достижимости (Reference Types)

| Достижимость | Условие | Судьба при GC |
|-------------|---------|---------------|
| **Strongly** | Достижим через Strong refs | Никогда не собирается |
| **Softly** | Только через SoftReference | Собирается при нехватке памяти |
| **Weakly** | Только через WeakReference | Собирается при следующем GC |
| **Phantom** | Только через PhantomReference | Объект finalized, ждёт cleanup |
| **F-reachable** | Имеет `finalize()` override | Воскрешение возможно в finalize() |
| **Unreachable** | Никак не достижим | Собирается немедленно |

## Thread Stack Scanning

При STW GC сканирует стек каждого потока. Для этого JVM должна знать **OopMap** — карту регистров и позиций стека, где находятся ссылки (oop = ordinary object pointer).

```
// OopMap генерируется C2 при компиляции для каждого safepoint:
// "в register rax — ссылка, offset+8 на стеке — ссылка, ..."
```

Это одна из причин, почему поток должен достичь safepoint перед GC: только в safepoint OopMap корректен.

## Static Fields как источник утечек

```java
// Классическая утечка памяти:
public class EventBus {
    // static → GC Root → весь граф объектов не собирается
    private static final List<Listener> listeners = new ArrayList<>();
    
    public static void subscribe(Listener l) {
        listeners.add(l);  // добавляем, но никогда не удаляем
    }
    // Все зарегистрированные Listener'ы живут вечно
}

// Решение: WeakReference или explicit unsubscribe
private static final List<WeakReference<Listener>> listeners = new ArrayList<>();
```

## Concurrent Marking (G1/ZGC)

Concurrent GC не останавливает потоки для marking. Проблема: пока GC маркирует, приложение меняет граф ссылок.

**Tri-color invariant:**
- Белые — не посещены (потенциальный мусор)
- Серые — в очереди на обработку (ссылки не все просмотрены)
- Чёрные — полностью обработаны

Нарушение: чёрный объект получает ссылку на белый (через mutable field), а серый теряет ссылку на него → белый будет ошибочно собран.

Решение: **SATB barriers** (G1) или **Incremental Update** (CMS) + Remark фаза для доработки изменений.

## ClassLoader Leaks

```java
// Утечка ClassLoader → утечка всех загруженных им классов (Metaspace)
// → Static fields этих классов → весь heap
void createDynamicProxy() {
    URLClassLoader loader = new URLClassLoader(urls);
    Class<?> cls = loader.loadClass("com.example.Plugin");
    // ... используем
    // Если cls или loader достижим через static field → loader никогда не будет собран
}
```

Диагностика: `jmap -clstats <pid>` показывает количество классов и их ClassLoader'ы.

---

## Вопросы на интервью
- Назовите все типы GC Roots в JVM.
- Почему Java не страдает от проблемы цикличных ссылок?
- Как GC сканирует стеки потоков? Что такое OopMap?
- Объясните tri-color marking и как SATB решает проблему concurrent modification.
- Как static field может вызвать утечку памяти?

## Подводные камни
- ThreadLocal + ThreadPool = утечка: thread-local значение привязано к Thread (GC Root), пул recycleы потоки, значение никогда не GC'ится — нужно `remove()` после запроса
- JNI GlobalRef → если не освободить `DeleteGlobalRef` → объект никогда не собирается
- `String.intern()` в tight loop → interned strings сильно достижимы из JVM internals GC Root → permanent leak до Java 7, в Java 8+ в heap но всё равно не очевидно
- Финализируемые объекты (с `finalize()`) требуют двух GC-циклов для сборки → в Young Gen живут дольше, могут перетечь в Old Gen
