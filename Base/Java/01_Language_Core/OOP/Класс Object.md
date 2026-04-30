## Связанные темы

[[Наследование]], [[Интерфейсы]], [[Java Reflection API]], [[Java Monitor]], [[Reference Types (Weak, Soft, Phantom)]]

---

## equals / hashCode — контракт пары

```java
// Контракт equals:
// 1. Рефлексивность: x.equals(x) == true
// 2. Симметричность: x.equals(y) == y.equals(x)
// 3. Транзитивность: x.equals(y) && y.equals(z) → x.equals(z)
// 4. Консистентность: результат стабилен пока объект не изменился
// 5. Нулевое сравнение: x.equals(null) == false

@Override
public boolean equals(Object obj) {
    if (this == obj) return true;                    // оптимизация ссылочного равенства
    if (!(obj instanceof Person other)) return false; // pattern matching (Java 16+)
    return age == other.age && Objects.equals(name, other.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age); // согласован с equals — те же поля!
}
```

**Контракт hashCode:**
- Если `x.equals(y)` → `x.hashCode() == y.hashCode()` (обязательно)
- Если `x.hashCode() == y.hashCode()` → `x.equals(y)` может быть false (коллизия, допустимо)

**Ловушка:** переопределил `equals` без `hashCode` → `HashSet.contains(obj)` вернёт `false`, `HashMap.get(key)` вернёт `null` — объект «потерян».

```java
// record — автогенерация правильного equals/hashCode/toString:
record Point(int x, int y) {}  // все 3 метода сгенерированы на основе полей
```

---

## toString

```java
// По умолчанию: "ClassName@hexHashCode" — бесполезно для отладки
// Переопределяй для читаемого вывода:
@Override
public String toString() {
    return "Person{name='%s', age=%d}".formatted(name, age);
}

// Или через lombok: @ToString
// Или record — toString() генерируется автоматически
```

---

## clone — shallow copy и проблемы

```java
// Требует реализации Cloneable (marker interface) + override clone()
public class Person implements Cloneable {
    private String name;
    private Address address; // изменяемый объект!

    @Override
    public Person clone() {
        try {
            Person cloned = (Person) super.clone(); // shallow copy!
            cloned.address = this.address.clone();  // deep copy вручную
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(); // не произойдёт — мы реализуем Cloneable
        }
    }
}
```

**Проблема `clone()`:** shallow copy копирует ссылки — изменение `address` в клоне меняет оригинал. Deep copy надо писать вручную рекурсивно.

**Предпочтительные альтернативы:**
- Copy constructor: `new Person(original)` — явный и понятный
- Static factory: `Person.copyOf(original)`
- Serialization/deserialization (для deep copy сложных графов)

---

## wait / notify / notifyAll

```java
// Вызов только внутри synchronized — иначе IllegalMonitorStateException
synchronized (lock) {
    while (!condition) {    // WHILE, не if — protection against spurious wakeups
        lock.wait();        // освобождает monitor + блокирует поток
    }
    // condition выполнена — работаем
}

// Другой поток:
synchronized (lock) {
    condition = true;
    lock.notifyAll();       // пробуждает ВСЕ ждущие потоки → они снова конкурируют за lock
    // lock.notify() — только один случайный поток
}
```

**Почему `while`, не `if`:** spurious wakeups — поток может проснуться без `notify()` (разрешено JMM). `while` защищает от этого.

**Современная альтернатива:** `Lock` + `Condition` из `java.util.concurrent.locks` — поддерживает несколько condition variables, timed wait, interruptible wait.

---

## getClass() vs instanceof

```java
Object obj = new Dog();

// getClass() — точный тип, не учитывает полиморфизм:
obj.getClass() == Dog.class    // true
obj.getClass() == Animal.class // false

// instanceof — с учётом иерархии (полиморфизм):
obj instanceof Dog    // true
obj instanceof Animal // true (Dog IS-A Animal)
obj instanceof Object // true (всегда)

// Pattern matching (Java 16+):
if (obj instanceof Dog d) {
    d.bark(); // d уже типизирован как Dog
}
```

В `equals()` обычно `getClass() != obj.getClass()` — запрещает равенство с подклассом. Если нужен симметричный equals в иерархии — использовать `instanceof`.

---

## Object Header — как объект выглядит в памяти

```
Object layout (64-bit JVM, CompressedOops):
┌─────────────────┬──────────────────────────────────┐
│ Mark Word (8B)  │ hash, GC age, lock state/monitor │
├─────────────────┼──────────────────────────────────┤
│ Klass ptr (4B)  │ указатель на Class metadata      │
├─────────────────┼──────────────────────────────────┤
│ Fields...       │                                  │
└─────────────────┴──────────────────────────────────┘

Минимальный объект: 16 байт (8 Mark Word + 4 Klass + 4 выравнивание)
```

`hashCode()` по умолчанию вычисляется JVM и кэшируется в Mark Word. После вычисления живёт там постоянно. Это объясняет почему `System.identityHashCode(obj)` после смены состояния не меняется — он закэширован.

---

## finalize → Cleaner API

```java
// finalize() — НИКОГДА не используй:
// - Нет гарантии вызова (GC может не вызвать никогда)
// - Задерживает сборку мусора (объект выживает минимум 2 GC цикла)
// - Может "воскресить" объект (сохранив this куда-то в finalize)
// - Удалён в Java 18 (JEP 421)

// Правильная замена — Cleaner API (Java 9+):
public class NativeResource implements AutoCloseable {
    private static final Cleaner CLEANER = Cleaner.create();
    private final Cleaner.Cleanable cleanable;
    private final long nativeHandle;

    public NativeResource(long handle) {
        this.nativeHandle = handle;
        // State должен быть static nested или внешним классом — НЕ inner
        // (иначе захватывает this → Cleaner не сможет GC объект)
        this.cleanable = CLEANER.register(this, new CleanupAction(handle));
    }

    private static class CleanupAction implements Runnable {
        private final long handle;
        CleanupAction(long handle) { this.handle = handle; }

        @Override
        public void run() { NativeLibrary.free(handle); } // вызовется при GC
    }

    @Override
    public void close() { cleanable.clean(); } // явное освобождение (предпочтительно)
}
```

---

## Вопросы на интервью

- Почему при переопределении `equals` обязательно переопределять `hashCode`?
- Можно ли нарушить контракт `equals`? К чему это приводит?
- Что такое spurious wakeup? Почему `wait()` нужно вызывать в `while`, не в `if`?
- Чем `clone()` плохо? Какие альтернативы?
- `getClass() == other.getClass()` vs `instanceof` в `equals()` — когда что использовать?
- Что хранится в Object Header? Что такое Mark Word?

---

## Подводные камни

- **Переопределил `equals`, забыл `hashCode`** — `HashMap`/`HashSet` не находят объект. Нарушение контракта.
- **Изменяемые поля в `equals`/`hashCode`** — после изменения поля объект нельзя найти в HashMap (лежит в старом бакете). Всегда иммутабельные ключи.
- **`wait()` без `synchronized`** — `IllegalMonitorStateException` в runtime. JVM не может проверить это статически.
- **`clone()` без deep copy** — `cloned.address == original.address`, изменения в одном меняют другой.
- **`Cloneable` без override `clone()`** — `super.clone()` доступен, но `protected` → надо override с public.
- **`finalize()` в Java 18+** — удалён. Использование → compile warning → runtime → ничего не вызовется.
