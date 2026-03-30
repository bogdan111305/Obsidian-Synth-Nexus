# ClassLoaders в Java

> **ClassLoader** — загружает `.class` в JVM. Иерархия: Bootstrap → Platform → Application → Custom. Принцип **делегирования**: сначала спрашивает родителя, только потом ищет сам. Класс идентифицируется по `(ClassLoader, packageName, className)` — один класс, загруженный разными CL, несовместим!
> На интервью: delegation model, child-first vs parent-first, ClassLoader leaks в Metaspace, NoClassDefFoundError vs ClassNotFoundException, TCCL.

## Связанные темы
[[Java Memory Structure]], [[Java Reflection API]], [[Java Assembling]]

---

## Иерархия ClassLoader

```
Bootstrap ClassLoader  (C++, не Java-объект, загружает java.base)
        ↑ parent
Platform ClassLoader   (Java 9+; Extension CL в Java 8, загружает java.sql, java.xml...)
        ↑ parent
Application ClassLoader (System CL, загружает classpath/modulepath)
        ↑ parent
Custom ClassLoader      (плагины, hot-reload, загрузка из сети/БД)
```

```java
System.out.println(String.class.getClassLoader());      // null = Bootstrap CL
System.out.println(ArrayList.class.getClassLoader());   // null = Bootstrap CL
ClassLoader appCL = ClassLoader.getSystemClassLoader(); // "app"
ClassLoader platCL = ClassLoader.getPlatformClassLoader(); // "platform"
```

`null` — соглашение JVM для Bootstrap CL: он реализован на C++ и не является Java-объектом.

---

## Delegation Model (Parent-First)

```
loadClass("com.example.MyClass"):
  1. Проверить кэш (findLoadedClass)
  2. Делегировать родителю (рекурсивно до Bootstrap)
  3. Если никто не нашёл → findClass() в текущем CL
  4. ClassNotFoundException
```

```java
// Алгоритм loadClass() (исходник OpenJDK):
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        Class<?> c = findLoadedClass(name);           // 1. кэш
        if (c == null) {
            try {
                c = parent != null
                    ? parent.loadClass(name, false)   // 2. делегация
                    : findBootstrapClassOrNull(name);
            } catch (ClassNotFoundException ignored) {}
            if (c == null) c = findClass(name);       // 3. искать самому
        }
        if (resolve) resolveClass(c);
        return c;
    }
}
```

**Зачем нужна делегация:**
- **Безопасность** — нельзя подменить `java.lang.String` своей реализацией
- **Консистентность** — один класс = один `Class` объект в рамках одного CL
- **Изоляция** — разные CL могут загружать одноимённые классы независимо

---

## ClassLoader API

| Метод | Описание |
|---|---|
| `loadClass(String)` | Загружает с делегацией (переопределять не рекомендуется) |
| `findClass(String)` | Ищет только в этом CL (точка расширения) |
| `defineClass(byte[])` | Создаёт `Class` из байт-кода |
| `findLoadedClass(String)` | Проверить кэш |
| `getParent()` | Родительский CL |

**Переопределяй `findClass()`, не `loadClass()`** — сохраняется delegation model:

```java
public class FileClassLoader extends ClassLoader {
    private final Path classDir;

    public FileClassLoader(Path classDir, ClassLoader parent) {
        super(parent);
        this.classDir = classDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Path path = classDir.resolve(name.replace('.', '/') + ".class");
        try {
            byte[] bytes = Files.readAllBytes(path);
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

---

## Кастомный ClassLoader: плагинная система

```java
public class PluginLoader extends ClassLoader {
    private final Path pluginJar;

    public PluginLoader(Path pluginJar) {
        super(ClassLoader.getSystemClassLoader()); // делегация для JDK/app классов
        this.pluginJar = pluginJar;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try (JarFile jar = new JarFile(pluginJar.toFile())) {
            JarEntry entry = jar.getJarEntry(name.replace('.', '/') + ".class");
            if (entry == null) throw new ClassNotFoundException(name);
            byte[] bytes = jar.getInputStream(entry).readAllBytes();
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}

// Hot-reload: новый CL = новая версия класса
ClassLoader pluginCL = new PluginLoader(Path.of("plugin-v2.jar"));
Class<?> cls = pluginCL.loadClass("com.plugin.MyPlugin");
Object instance = cls.getDeclaredConstructor().newInstance();
// Убрать ссылки на старый CL → GC выгрузит его классы из Metaspace
```

---

## Когда классы выгружаются (Metaspace leaks)

Класс выгружается из Metaspace только когда:
1. `ClassLoader` объект стал **недостижимым** для GC
2. Все загруженные им классы стали недостижимыми
3. Все `Class` объекты этих классов стали недостижимыми

**ClassLoader leaks** — частая проблема при hot-deploy в контейнерах:
- Старый CL держится через: статические поля JDBC driver, ThreadLocal, shutdown hook, log4j appender
- Симптом: `OutOfMemoryError: Metaspace` при каждом redeploy
- Диагностика: `jmap -histo <pid>` → ищи множество `ClassLoader` экземпляров; JFR `jdk.ClassLoaderStatistics`

---

## NoClassDefFoundError vs ClassNotFoundException

| Ошибка | Когда | Причина |
|---|---|---|
| `ClassNotFoundException` | `Class.forName()`, `loadClass()` — явная загрузка | Не найден в classpath |
| `NoClassDefFoundError` | JVM загружает класс нужный для другого | Был в compile-time, отсутствует в runtime |

```java
try {
    Class<?> cls = Class.forName("com.example.Missing"); // ClassNotFoundException
} catch (ClassNotFoundException e) { ... }

// NoClassDefFoundError — не поймаешь через catch ClassNotFoundException:
// class A { B b = new B(); } // если B.class отсутствует в runtime → NCDFE
```

**Ещё причина NCDFE:** класс был загружен, но его static инициализатор бросил исключение → класс помечается "failed" → повторная попытка загрузки → `NoClassDefFoundError`.

---

## Thread Context ClassLoader

Проблема: Bootstrap/Platform CL загружает библиотеки (JDBC, JNDI, SPI), но им нужны классы из Application classpath (драйверы, провайдеры).

```java
// По умолчанию = System ClassLoader
ClassLoader tccl = Thread.currentThread().getContextClassLoader();

// Библиотека использует TCCL вместо своего CL:
Class<?> driver = tccl.loadClass("com.mysql.cj.jdbc.Driver");

// Замена TCCL в контейнерах приложений:
ClassLoader original = Thread.currentThread().getContextClassLoader();
Thread.currentThread().setContextClassLoader(webAppClassLoader);
try {
    // код выполняется с CL веб-приложения
} finally {
    Thread.currentThread().setContextClassLoader(original); // обязателен restore
}
```

---

## JPMS (Java 9+)

```
Bootstrap CL  → java.base module (core JDK)
Platform CL   → platform modules (java.sql, java.xml, java.logging...)
Application CL → app module (classpath / modulepath)
```

`Extension ClassLoader` переименован в `Platform ClassLoader`. `rt.jar`/`tools.jar` убраны. `ClassLoader.loadClass()` работает по-прежнему, но с учётом модульных границ.

```java
Module module = String.class.getModule();
System.out.println(module.getName()); // "java.base"
```

---

## Вопросы на интервью

- Почему `String.class.getClassLoader()` возвращает `null`?
- Почему нужно переопределять `findClass()`, а не `loadClass()`?
- Что такое child-first ClassLoader? Зачем он нужен в Tomcat?
- Как ClassLoader связан с утечками памяти в Metaspace при hot-deploy?
- Чем `ClassNotFoundException` отличается от `NoClassDefFoundError`?
- Что такое Thread Context ClassLoader? Почему он нужен для JDBC?
- Можно ли перезагрузить класс без перезапуска JVM? Как?
- Почему объекты одного класса, загруженного разными CL, несовместимы по типу?

---

## Подводные камни

- **ClassCastException между разными CL** — `(Foo) fooFromOtherCL` → `ClassCastException` даже если тот же `.class` файл. Используй общий интерфейс, загружаемый через App CL.
- **Переопределение `loadClass()`** — ломает delegation model. Только OSGi/системные плагины делают это сознательно. Обычный код → `findClass()`.
- **Статический инициализатор + NCDFE** — если `static {}` блок бросил исключение, класс помечается "failed initialization" навсегда. Повторный `Class.forName()` → `NoClassDefFoundError`, не `ClassNotFoundException`.
- **TCCL без restore** — `setContextClassLoader()` без `finally { restore }` = следующий код в потоке получает "чужой" CL. Томкат уже видел это как источник загрузки классов из неправильного webapp.
- **CL leak через ThreadLocal** — если `ThreadLocal` хранит объект класса, загруженного кастомным CL, и поток из пула пережил redeploy, кастомный CL не будет собран GC. Очищай ThreadLocal при завершении задач.
- **Bootstrap CL = null checks** — код `cl.getParent()` упадёт с NPE для Bootstrap. Всегда: `ClassLoader parent = cl.getParent(); if (parent == null) parent = ClassLoader.getSystemClassLoader();`
