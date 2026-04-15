# Vector API — SIMD в Java

> **Vector API** (JEP 338/414/426/438/460/489, Java 16-24, Incubator → Preview) позволяет писать Java код, который JIT компилирует в **SIMD инструкции** (SSE, AVX, AVX-512, ARM NEON). Обрабатывает несколько элементов за одну CPU инструкцию — до 16x ускорение для числодробилок.

## Связанные темы
[[Foreign Function and Memory API]], [[JIT Compiler & Optimizations]], [[Panama vs Unsafe vs JNI]]

---

## SIMD — что это

**SIMD** (Single Instruction, Multiple Data) — одна инструкция CPU обрабатывает несколько значений одновременно:

```
Scalar (обычный Java):
  a[0] + b[0] = c[0]  ← одна операция за такт
  a[1] + b[1] = c[1]
  a[2] + b[2] = c[2]
  a[3] + b[3] = c[3]
  = 4 тактов

SIMD (AVX2, 256-bit):
  [a0,a1,a2,a3,a4,a5,a6,a7] + [b0,b1,b2,b3,b4,b5,b6,b7] = [c0..c7]
  = 1 такт для 8 float (256 / 32 = 8 операций)
```

Без Vector API JIT иногда auto-vectorizes простые loops, но ненадёжно и без контроля программиста.

## Основные классы

```java
import jdk.incubator.vector.*;

// Species — описание вектора: тип элемента + размер
VectorSpecies<Float> SPECIES = FloatVector.SPECIES_256;  // 8 float = 256 бит
VectorSpecies<Float> SPECIES_PREFERRED = FloatVector.SPECIES_PREFERRED;  // оптимальный для CPU

// Размеры: 64, 128, 256, 512 бит
// PREFERRED — JVM выбирает оптимальный под текущее железо
```

## Пример: сложение массивов

```java
import jdk.incubator.vector.*;

public class VectorAdd {
    static final VectorSpecies<Float> SPECIES = FloatVector.SPECIES_PREFERRED;
    
    // Скалярная версия
    static void addScalar(float[] a, float[] b, float[] c) {
        for (int i = 0; i < a.length; i++) {
            c[i] = a[i] + b[i];
        }
    }
    
    // Векторная версия
    static void addVector(float[] a, float[] b, float[] c) {
        int i = 0;
        int upperBound = SPECIES.loopBound(a.length);  // округление вниз до кратного SPECIES.length()
        
        // Основной цикл — обрабатываем SPECIES.length() элементов за итерацию
        for (; i < upperBound; i += SPECIES.length()) {
            FloatVector va = FloatVector.fromArray(SPECIES, a, i);
            FloatVector vb = FloatVector.fromArray(SPECIES, b, i);
            FloatVector vc = va.add(vb);
            vc.intoArray(c, i);
        }
        
        // Хвост — оставшиеся элементы (< SPECIES.length())
        for (; i < a.length; i++) {
            c[i] = a[i] + b[i];
        }
    }
}
```

## Mask — conditional operations

```java
// Обработать только элементы > 0
static void processPositive(float[] data, float[] result) {
    int i = 0;
    int upperBound = SPECIES.loopBound(data.length);
    
    for (; i < upperBound; i += SPECIES.length()) {
        FloatVector v = FloatVector.fromArray(SPECIES, data, i);
        
        // Создаём mask: true где v > 0
        VectorMask<Float> mask = v.compare(VectorOperators.GT, 0.0f);
        
        // Применяем операцию только к элементам где mask = true
        FloatVector processed = v.mul(2.0f, mask);  // * 2 только для > 0
        
        processed.intoArray(result, i, mask);  // записываем только masked
    }
    // scalar tail...
}
```

## Реальные use cases

### Dot Product (ML, DSP)
```java
static float dotProduct(float[] a, float[] b) {
    var SPECIES = FloatVector.SPECIES_PREFERRED;
    FloatVector sum = FloatVector.zero(SPECIES);
    
    int i = 0;
    for (; i < SPECIES.loopBound(a.length); i += SPECIES.length()) {
        FloatVector va = FloatVector.fromArray(SPECIES, a, i);
        FloatVector vb = FloatVector.fromArray(SPECIES, b, i);
        sum = va.fma(vb, sum);  // fused multiply-add: sum += a[i] * b[i]
    }
    float result = sum.reduceLanes(VectorOperators.ADD);
    
    // scalar tail
    for (; i < a.length; i++) {
        result += a[i] * b[i];
    }
    return result;
}
```

### String/Byte processing (поиск паттернов, хэши)
```java
// Найти позицию символа в строке (упрощённо)
static int indexOf(byte[] bytes, byte target) {
    var SPECIES = ByteVector.SPECIES_PREFERRED;
    ByteVector targetVec = ByteVector.broadcast(SPECIES, target);
    
    int i = 0;
    for (; i < SPECIES.loopBound(bytes.length); i += SPECIES.length()) {
        ByteVector v = ByteVector.fromArray(SPECIES, bytes, i);
        VectorMask<Byte> mask = v.compare(VectorOperators.EQ, targetVec);
        if (mask.anyTrue()) {
            return i + mask.firstTrue();
        }
    }
    // scalar tail...
    return -1;
}
```

## Производительность

```
Тест: сложение двух float[] по 1М элементов
                    Throughput    Speedup
Scalar (JIT)        ~800 МБ/с    1x
AutoVectorized JIT  ~2 ГБ/с      2.5x
Vector API (AVX2)   ~6 ГБ/с      7.5x
Vector API (AVX-512) ~12 ГБ/с   15x

Реальные цифры зависят от CPU, размера данных, паттерна доступа.
```

## Статус и использование

```java
// Java 22-24: Incubator module
// Нужен флаг при компиляции и запуске:
// javac --add-modules jdk.incubator.vector MyClass.java
// java  --add-modules jdk.incubator.vector MyClass

// Maven:
// <compilerArgs><arg>--add-modules</arg><arg>jdk.incubator.vector</arg></compilerArgs>
```

Vector API используется внутри JDK: `java.util.Base64`, некоторые String операции.

## JDK использует Vector API внутри

- `java.util.Base64` — JDK 17+ использует Vector API для encode/decode
- String indexOf (некоторые реализации)
- `Arrays.mismatch`

---

## Вопросы на интервью
- Что такое SIMD и как Vector API его использует?
- Зачем нужен `loopBound()` и "scalar tail"?
- Что такое VectorMask и когда он нужен?
- Почему Vector API быстрее чем auto-vectorization JIT?
- В каких задачах Vector API даёт максимальный эффект?

## Подводные камни
- Vector API — Incubator/Preview на 2026. API может измениться между версиями Java
- Эффект заметен только на больших данных (>10K элементов) — иначе overhead initialization перевешивает
- `SPECIES_PREFERRED` — JVM выбирает под текущий CPU. На одном сервере может быть 512-bit, на другом 128-bit — разная производительность
- Неправильный `loopBound` + пропущенный scalar tail → ArrayIndexOutOfBoundsException или обработка не всех данных
- На виртуальных машинах/контейнерах AVX-512 может быть отключён администратором — `SPECIES_PREFERRED` упадёт до 256-bit или 128-bit
