# Java Bytecode — структура и опкоды

> Байткод — промежуточное представление Java-программы, выполняемое JVM. `.class` файл содержит: magic number, константный пул, флаги класса, поля, методы (с Code атрибутом), атрибуты. Байткод-инструкции (опкоды) — однобайтовые команды для стековой машины JVM.

## Связанные темы
[[Java Assembling]], [[JIT Compiler & Optimizations]], [[invokedynamic — механика]], [[Java Reflection API]], [[MethodHandle & LambdaMetafactory]]

---

## Структура .class файла

```
.class файл (big-endian):
┌─────────────────────────────────────────────────────┐
│ magic number: 0xCAFEBABE (4 байта)                  │
│ minor_version (2 байта)                             │
│ major_version (2 байта) — Java 21 = 65, Java 17 = 61│
│ constant_pool_count (2 байта)                       │
│ constant_pool[] — пул констант                      │
│ access_flags (2 байта) — ACC_PUBLIC, ACC_FINAL, ... │
│ this_class (индекс в constant_pool)                 │
│ super_class (индекс в constant_pool)                │
│ interfaces_count + interfaces[]                     │
│ fields_count + fields[]                             │
│ methods_count + methods[]                           │
│   └── Code attribute: max_stack, max_locals, code[] │
│ attributes_count + attributes[]                     │
│   ├── SourceFile                                    │
│   ├── Signature (дженерики — не erasure!)           │
│   ├── RuntimeVisibleAnnotations                     │
│   └── PermittedSubclasses (sealed classes)          │
└─────────────────────────────────────────────────────┘
```

**Важно:** дженерики в `Signature` атрибуте **не стираются** — это основа Super Type Token паттерна. Type erasure затрагивает только операнды/дескрипторы методов.

---

## Константный пул

Пул констант — таблица всех литералов, имён, дескрипторов, используемых в классе:

```
Constant Pool entries:
#1 = Utf8          "Hello"
#2 = String        #1           ← ссылка на #1
#3 = Utf8          "java/lang/System"
#4 = Class         #3           ← ссылка на #3
#5 = Utf8          "out"
#6 = Utf8          "Ljava/io/PrintStream;"
#7 = NameAndType   #5:#6        ← имя + дескриптор
#8 = Fieldref      #4.#7        ← класс + NameAndType

// Дескриптор метода void main(String[] args):
// ([Ljava/lang/String;)V
// L<classname>; = объект, [ = массив, V = void, I = int, ...
```

---

## JVM — стековая машина

JVM — **стековая машина**: большинство операций берут операнды со стека и кладут результат обратно. Каждый поток имеет свой стек, каждый метод — свой **стековый фрейм**:

```
Stack Frame:
├── Operand Stack    — операнды текущих инструкций
├── Local Variables  — массив локальных переменных [this, arg0, arg1, ...]
└── Frame Data       — ссылка на constant pool, возвращаемый адрес
```

---

## Ключевые группы опкодов

### Загрузка/сохранение

```
iload_<n>   — загрузить int локальную переменную n на стек
istore_<n>  — сохранить int со стека в локальную переменную n
aload_<n>   — загрузить reference (объект) локальную переменную n
astore_<n>  — сохранить reference
lload, fload, dload — long, float, double

iconst_<n>  — загрузить int константу (-1..5)
lconst_<n>  — long константу (0, 1)
bipush <b>  — byte константу на стек как int
sipush <s>  — short константу на стек как int
ldc #idx    — загрузить из constant pool (String, float, int, Class)
```

### Арифметика

```
iadd, isub, imul, idiv, irem  — int арифметика
ladd, fadd, dadd              — long, float, double
iinc <var>, <const>           — инкремент локальной переменной (не через стек!)
ineg, lneg                    — унарный минус
ishl, ishr, iushr             — битовые сдвиги
iand, ior, ixor               — битовые операции
```

### Сравнение и переходы

```
if_icmpeq, if_icmpne, if_icmplt, if_icmpge, if_icmpgt, if_icmple
    — сравнение двух int на стеке, прыжок если условие истинно
ifnull, ifnonnull
    — проверка reference на null
goto <offset>
    — безусловный прыжок
tableswitch  — dense switch (O(1))
lookupswitch — sparse switch (O(log n))
```

### Вызов методов

```
invokevirtual   — виртуальный вызов instance-метода (через vtable)
invokespecial   — конструкторы, private методы, super вызовы
invokestatic    — статические методы
invokeinterface — методы через интерфейс (через itable)
invokedynamic   — динамически связанный вызов (лямбды, String concat)
```

### Работа с объектами

```
new <class>           — создать объект (выделить память, поместить ref на стек)
dup                   — дублировать top стека
checkcast <class>     — explicit cast, бросает ClassCastException
instanceof <class>    — проверка типа
getfield / putfield   — доступ к instance-полям
getstatic / putstatic — доступ к static-полям
arraylength           — длина массива
newarray / anewarray  — создать примитивный/объектный массив
```

### Возврат

```
ireturn, lreturn, freturn, dreturn, areturn — возврат примитива/reference
return — void возврат
athrow — выброс исключения
```

---

## Пример: разбор байткода метода

```java
public static int add(int a, int b) {
    return a + b;
}
```

```
javap -c MyClass:
public static int add(int, int);
  Code:
     0: iload_0          // загрузить a (local var 0)
     1: iload_1          // загрузить b (local var 1)
     2: iadd             // сложить, результат на стек
     3: ireturn          // вернуть int со стека
```

```java
public void example() {
    String s = "Hello";
    System.out.println(s);
}
```

```
  Code:
     0: ldc           #2    // загрузить String "Hello" из constant pool
     2: astore_1            // сохранить в local var 1 (s)
     3: getstatic     #3    // System.out (PrintStream)
     6: aload_1             // загрузить s
     7: invokevirtual #4    // PrintStream.println(String)
    10: return
```

---

## Инструменты для работы с байткодом

```bash
# Встроенный дизассемблер:
javap -c MyClass.class       # код методов
javap -verbose MyClass.class # полный вывод: constant pool, атрибуты
javap -p MyClass.class       # включая private члены

# ASM — библиотека для анализа и генерации байткода:
# ClassReader → AnalyzerAdapter → ClassWriter
# Используется: Spring (CGLIB), Hibernate, Mockito, ByteBuddy

# ByteBuddy — высокоуровневый API поверх ASM:
new ByteBuddy()
    .subclass(Object.class)
    .method(ElementMatchers.named("toString"))
    .intercept(FixedValue.value("intercepted"))
    .make()
    .load(ClassLoader.getSystemClassLoader());

# Viewing в IDE: IntelliJ IDEA → View → Show Bytecode
```

---

## Вопросы на интервью

- Что такое байткод? Чем он отличается от машинного кода?
- Что такое magic number в `.class` файле? Что такое major_version?
- JVM — стековая или регистровая машина? Что такое operand stack и local variables?
- Чем `invokevirtual` отличается от `invokeinterface` и `invokespecial`?
- Что такое `tableswitch` vs `lookupswitch`? Когда компилятор выбирает каждый?
- Что хранит константный пул? Почему дженерики в `Signature` не стираются?
- Что делает `invokedynamic`? Для чего он используется в современном Java?
- Как посмотреть байткод класса?

## Подводные камни

- **`new` ≠ инициализация** — опкод `new` только выделяет память и кладёт ref на стек. `<init>` (конструктор) вызывается отдельно через `invokespecial`. Между `new` и `invokespecial` объект технически существует, но не инициализирован.
- **`invokespecial` ≠ `invokevirtual`** — `invokespecial` используется для `super.method()`, конструкторов и `private` методов. Это прямой невиртуальный вызов — нет lookup в vtable.
- **Дескрипторы методов** — `([Ljava/lang/String;)V` это `void main(String[])`. `L` = object reference, `[` = array, `;` — конец class name. Путаница в этих дескрипторах = неправильный MethodHandle.
- **major_version при компиляции** — если скомпилировать с `--release 11`, major_version = 55. JVM 11 не загрузит класс с major_version 65 (Java 21).
- **`iinc` — инкремент без стека** — `i++` в цикле компилируется в `iinc`, а не `iload + iconst_1 + iadd + istore`. Это специальная оптимизация, но она работает только для локальных переменных.
