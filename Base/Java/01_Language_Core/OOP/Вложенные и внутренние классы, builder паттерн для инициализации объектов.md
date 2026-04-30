## Связанные темы

[[Инкапсуляция]], [[Поля, конструкторы, this, инициализаторы]], [[Java Stream API & Functional Programming]]

---

## Сравнительная таблица

| Тип | static? | Ссылка на внешний | Создание |
|---|:---:|:---:|---|
| Static nested class | Да | Нет | `new Outer.Nested()` |
| Inner (non-static) class | Нет | Да (`this$0`) | `outer.new Inner()` |
| Local class | Нет | Да | внутри метода |
| Anonymous class | Нет | Да | `new Interface() {...}` |

---

## Static Nested Class — группировка без связи

```java
public class HttpClient {
    // Static nested: логически связан с HttpClient, но не зависит от его инстанса
    public static class Builder {
        private String baseUrl;
        private int timeout = 30;

        public Builder baseUrl(String url) { this.baseUrl = url; return this; }
        public Builder timeout(int sec) { this.timeout = sec; return this; }
        public HttpClient build() { return new HttpClient(this); }
    }

    private final String baseUrl;
    private final int timeout;

    private HttpClient(Builder b) {
        this.baseUrl = b.baseUrl;
        this.timeout = b.timeout;
    }
}

HttpClient client = new HttpClient.Builder()
    .baseUrl("https://api.example.com")
    .timeout(60)
    .build();
```

`Map.Entry<K,V>` в JDK — пример static nested interface.

---

## Inner Class — доступ к внешнему объекту (осторожно с памятью)

```java
public class LinkedList<E> {
    private Node<E> head;
    private int size;

    // Iterator — классический случай inner class:
    // итератору нужен доступ к полям head, size
    private class ListIterator implements Iterator<E> {
        private Node<E> current = head; // доступ к полю внешнего класса!

        @Override
        public boolean hasNext() { return current != null; }

        @Override
        public E next() {
            E val = current.val;
            current = current.next;
            return val;
        }
    }

    public Iterator<E> iterator() { return new ListIterator(); }
}
```

**Байткод inner class:**
```
class LinkedList$ListIterator {
    final LinkedList this$0;  // синтетическое поле — ссылка на внешний объект
    // Любое обращение к head → this$0.head
}
```

**Ловушка:** inner class держит ссылку на внешний объект. Если inner class живёт долго (callback, future) → внешний объект не может быть GC'd → **утечка памяти**.

---

## Anonymous Class vs Lambda

```java
// Анонимный класс (до Java 8):
Runnable r1 = new Runnable() {
    @Override
    public void run() { System.out.println("run"); }
};
// → компилятор создаёт class Outer$1 implements Runnable

// Lambda (Java 8+) — предпочтительно для функциональных интерфейсов:
Runnable r2 = () -> System.out.println("run");
// → invokedynamic + LambdaMetafactory (нет нового класса)

// Анонимный класс нужен когда:
// 1. Нужно сохранять состояние (несколько полей)
// 2. Реализуется нефункциональный интерфейс (>1 метода)
// 3. Нужно extends, а не implements
Comparator<String> withState = new Comparator<String>() {
    private int compareCount = 0; // состояние
    @Override
    public int compare(String a, String b) {
        compareCount++;
        return a.compareTo(b);
    }
};
```

**Effectively final:** переменные из окружающего scope, захваченные анонимным классом или лямбдой, должны быть `final` или effectively final (не изменяться после инициализации). Причина: они копируются в поля объекта.

---

## Builder Pattern — когда конструктор слишком сложный

```java
// Телескопический конструктор — плохо:
new Pizza("large", true, false, true, false, "mozarella", "tomato");

// Builder — читаемо и безопасно:
Pizza pizza = new Pizza.Builder("large")
    .cheese(true)
    .pepperoni(false)
    .mushrooms(true)
    .build();

// Lombok @Builder — автогенерация:
@Builder
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Pizza {
    private final String size;
    private final boolean cheese;
    private final boolean pepperoni;
    // ... lombok генерирует Builder класс автоматически
}
```

**Альтернатива для Java 16+** — `record` + compact constructor для валидации:
```java
record Pizza(String size, boolean cheese, boolean pepperoni) {
    Pizza { if (size == null) throw new NullPointerException(); }
}
// Но record иммутабельный — Builder нужен когда объект строится постепенно
```

---

## Вопросы на интервью

- Что такое `this$0`? В каком типе вложенных классов оно появляется?
- Почему inner class может вызвать утечку памяти? Как избежать?
- Почему лямбды предпочтительнее анонимных классов?
- Что такое effectively final? Почему лямбда не может захватить изменяемую переменную?
- Когда Builder предпочтительнее конструктора? Когда `record`?

---

## Подводные камни

- **Inner class в Android/Swing callbacks** — если Activity/Frame передаётся как контекст в long-lived inner class (Handler, AsyncTask) → утечка памяти. Используй `WeakReference` или static nested class.
- **Анонимный класс захватывает this** — не только переменные, но и весь внешний объект. Даже `obj::method` метод-ссылка захватывает `obj`.
- **Builder без валидации** — `build()` может вернуть объект в неполном состоянии если не все обязательные поля заполнены. Всегда валидируй в `build()`.
- **`new Outer.Inner()` без экземпляра Outer** — `Outer.Inner inner = new Outer.Inner()` → ошибка компиляции. Нужен `new outerInstance.Inner()` или `outer.new Inner()`.
