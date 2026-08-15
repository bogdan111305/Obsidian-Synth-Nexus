# JIT Compiler & Optimizations

> **JIT** (Just-In-Time) компилирует "горячий" байт-код в нативный. Два компилятора: **C1** (быстрая компиляция, базовые оптимизации) → **C2** (агрессивные оптимизации: инлайнинг, escape analysis, SIMD). Деоптимизация: если предположение C2 нарушается — откат к интерпретатору.
> На интервью: Tiered Compilation, мегаморфный call site, escape analysis и lock elimination, deoptimization причины и диагностика.

## Связанные темы
[[Java Memory Structure]], [[JVM Profiling & Observability]], [[Модель памяти Java (JMM) и барьеры памяти]], [[CAS и Unsafe]]

---

## Tiered Compilation

```
0 — Интерпретатор (без профилирования)
1 — C1: простые оптимизации, нет профиля
2 — C1: ограниченное профилирование
3 — C1: полное профилирование (типы, branch frequencies)
4 — C2: агрессивная оптимизация по профилю C1
```

**Ключевые пороги (HotSpot defaults):**
- `CompileThreshold=10000` — вызовов метода до C2
- `OnStackReplacePercentage=140` — порог для OSR (back-edge count)

```bash
-XX:+PrintCompilation          # что компилируется
-XX:TieredStopAtLevel=1        # только C1 (диагностика)
-Xint                           # только интерпретатор
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation -XX:LogFile=jit.log
```

---

## Escape Analysis

Определяет, "вытекает" ли объект за пределы метода или потока:

| Escape уровень | Описание | Оптимизация JIT |
|---|---|---|
| **NoEscape** | Объект локален в методе | Scalar Replacement / Stack Allocation |
| **ArgEscape** | Передан как аргумент | Lock Elimination |
| **GlobalEscape** | Возвращён / записан в поле | Стандартная heap-аллокация |

```java
// NoEscape → Scalar Replacement: объект разбивается на локальные переменные
// Никакой heap-аллокации!
public static long sum(int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        // Point p = new Point(i, i*2) → JIT: p.x = i, p.y = i*2 (локальные переменные)
        sum += i + i * 2L;
    }
    return sum;
}

// ArgEscape → Lock Elimination: synchronized удаляется для локального объекта
public String buildString() {
    StringBuffer sb = new StringBuffer(); // NoEscape → все lock eliminated!
    sb.append("Hello ").append("World");
    return sb.toString(); // не медленнее StringBuilder в этом контексте
}
```

---

## Method Inlining

Самая важная оптимизация. Устраняет overhead вызова и открывает дверь для дальнейших оптимизаций (constant folding, DCE).

```
-XX:MaxInlineSize=35           // байт байт-кода для маленьких методов
-XX:FreqInlineSize=325         // байт для "горячих" методов
-XX:+PrintInlining             // решения об инлайнинге
```

**Мегаморфный call site — враг инлайнинга:**
```java
// Биморфный (2 типа) — JIT инлайнит оба с guard
// Мегаморфный (3+ типов) — JIT отказывается → vtable dispatch, нет инлайнинга
for (Shape shape : shapes) {   // Circle, Square, Triangle...
    shape.area();              // мегаморфный → 5-10x медленнее монофонического
}

// Монофонический → инлайнинг гарантирован:
for (Circle c : circles) {
    c.area();                  // JIT: "всегда Circle" → inline + constant fold
}
```

JIT профилирует типы в call sites. Если в продакшне появится новый подтип → деоптимизация.

---

## Loop Optimizations

**Loop Unrolling:** JIT разворачивает тело цикла, уменьшая overhead counter/branch.

**Loop Vectorization (SIMD):**
```java
// JIT автоматически векторизует:
for (int i = 0; i < a.length; i++) {
    c[i] = a[i] + b[i]; // → vmovups + vaddps (8 float за раз на AVX)
}
```
Условия: массивы примитивов, простое тело, нет алиасинга. Флаг: `-XX:+UseSuperWord` (по умолчанию).

---

## On-Stack Replacement (OSR)

OSR позволяет заменить интерпретируемый цикл JIT-кодом **без перезапуска метода**.

Применяется для методов с горячими циклами, которые редко вызываются, но долго выполняются (например, `main()` с большим циклом). При достижении back-edge count порога JVM "прыгает" из интерпретатора в JIT-код прямо посередине итерации.

---

## Deoptimization (Деоптимизация)

JIT делает **спекулятивные оптимизации** на основе профиля. При нарушении предположений — откат к интерпретатору.

**Причины:**
- `null_check` — NullPointerException в "non-null" ветке
- `class_check` — новый подтип сломал монофоничность
- `unreached` — "мёртвый" блок кода неожиданно выполнился
- `unstable_if` — branch prediction оказался неверным

```bash
-XX:+TraceDeoptimization       # трассировка деоптимизаций
-XX:+PrintCompilation          # метод помечается "made not entrant"
```

---

## Intrinsics

JIT заменяет некоторые методы оптимизированными нативными имплементациями:

```java
Math.abs(), Math.min(), Math.max()    // прямые CPU инструкции
System.arraycopy()                     // memcpy/memmove
String.equals()                        // векторное SIMD сравнение
Arrays.fill()                          // memset
Integer.bitCount()                     // POPCNT инструкция CPU
Integer.numberOfLeadingZeros()         // LZCNT / BSR
```

`-XX:+PrintIntrinsics` — список intrinsified методов.

---

## Cache Locality

```java
// BAD: LinkedList → каждый узел — отдельный heap объект → cache miss на каждом
LinkedList<Integer> list = ...;
for (int x : list) { process(x); } // L2/L3 miss при каждом шаге

// GOOD: ArrayList → элементы рядом в памяти → CPU prefetcher работает
ArrayList<Integer> list = ...;
for (int x : list) { process(x); } // почти L1 cache
```

---

## Вопросы на интервью

- Что такое Tiered Compilation? Зачем нужны оба компилятора C1 и C2?
- Что такое мегаморфный call site? Почему он опасен для производительности?
- Что такое Escape Analysis? Какие оптимизации он открывает?
- Почему `new StringBuffer()` в локальном контексте не медленнее `StringBuilder`?
- Что такое OSR? Когда применяется?
- Что такое деоптимизация? Назови причины и как диагностировать.
- Что такое JIT intrinsic? Назови примеры.
- Почему `final`/`private` методы легче оптимизировать JIT?

---

## Подводные камни

- **Мегаморфный call site после прогрева** — добавление нового подтипа в hot loop в продакшне → деоптимизация + повторная компиляция. Может вызвать пик latency. Мониторируй `-XX:+TraceDeoptimization`.
- **Escape Analysis не работает через non-final поля** — если объект передаётся в метод который JIT не может инлайнить, escape analysis теряет его из виду → нет scalar replacement.
- **Слишком большие методы не инлайнятся** — если метод > `FreqInlineSize` (325 байт байт-кода), JIT пропускает инлайнинг. Длинные методы = потеря оптимизаций. Рефактори, но не ради рефакторинга.
- **Code Cache переполнение** — если `-XX:ReservedCodeCacheSize` исчерпан, JIT перестаёт компилировать → приложение деградирует к интерпретатору. Симптом: внезапный рост CPU + `-XX:+PrintCompilation` показывает "CodeCache is full".
- **OSR на long-running методах** — если метод с горячим циклом никогда не возвращается (main loop), OSR код может быть менее оптимальным чем обычная компиляция. JIT не может применить все оптимизации из-за необходимости совместимости состояния стека.
- **`-Xint` в продакшне** — полностью отключает JIT, приложение в 10-50x медленнее. Иногда используется для диагностики, не оставляй случайно.
