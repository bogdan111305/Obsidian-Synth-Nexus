---
title: "Java ClassLoaders — иерархия, делегирование, кастомные загрузчики"
tags: [java, classloader, jvm, jpms, metaspace]
updated: 2026-03-04
---

# ClassLoaders в Java

**ClassLoader** — компонент JVM, отвечающий за поиск, загрузку и определение классов в runtime. Понимание ClassLoader необходимо для диагностики `NoClassDefFoundError`, реализации плагинных систем, горячей перезагрузки кода и работы с JPMS.

## 1. Иерархия ClassLoader

```
Bootstrap ClassLoader  (C++, не Java-объект)
        ↑ parent
Platform ClassLoader   (Java 9+; Extension ClassLoader в Java 8)
        ↑ parent
Application ClassLoader (System ClassLoader)
        ↑ parent
Custom ClassLoader      (реализуется разработчиком)
```

### 1.1. Bootstrap ClassLoader

- Реализован на **C++**, встроен в JVM — не является Java-объектом
- Загружает **ядро JDK**: `java.lang`, `java.util`, `java.io` и другие классы из `java.base` модуля
- В Java 8: загружает `rt.jar`; в Java 9+: загружает bootstrap module (через internal URL)
- `String.class.getClassLoader()` возвращает **`null`** — это индикатор Bootstrap CL

```java
System.out.println(String.class.getClassLoader());      // null (Bootstrap)
System.out.println(ArrayList.class.getClassLoader());   // null (Bootstrap)
```

### 1.2. Platform ClassLoader (Java 9+)

- В Java 8 назывался **Extension ClassLoader** — загружал `jre/lib/ext/*.jar`
- В Java 9+ загружает **Platform модули**: `java.sql`, `java.xml`, `java.logging` и др.
- Родитель — Bootstrap ClassLoader
- `ClassLoader.getPlatformClassLoader()` — получить экземпляр

```java
ClassLoader platform = ClassLoader.getPlatformClassLoader();
System.out.println(platform.getName()); // "platform"
```

### 1.3. Application ClassLoader (System ClassLoader)

- Загружает классы из **classpath** (или modulepath в JPMS)
- Родитель — Platform ClassLoader
- Получить: `ClassLoader.getSystemClassLoader()` или `Thread.currentThread().getContextClassLoader()`

```java
ClassLoader appCL = ClassLoader.getSystemClassLoader();
System.out.println(appCL.getName()); // "app"

// Загрузить свой класс:
Class<?> cls = appCL.loadClass("com.example.MyClass");
```

### 1.4. Custom ClassLoader

- Создаётся разработчиком для реализации особой логики загрузки
- Примеры: плагинные системы (OSGi, IntelliJ Platform), горячая перезагрузка, загрузка из сети/БД

## 2. Delegation Model (Parent-First)

**Принцип работы**: перед загрузкой класса CL сначала делегирует запрос **родителю**.

```
1. AppCL.loadClass("com.example.MyClass")
2.   → Platform CL (parent). Найден? → вернуть
3.   → Bootstrap CL (parent родителя). Найден? → вернуть
4. Если Bootstrap не нашёл → Platform пробует найти
5. Если Platform не нашёл → App CL пробует найти
6. Если никто не нашёл → ClassNotFoundException
```

**Алгоритм `ClassLoader.loadClass()`**:

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 1. Проверить кэш (уже загружен?)
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                // 2. Делегировать родителю
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name); // Bootstrap CL
                }
            } catch (ClassNotFoundException e) {
                // родитель не нашёл — продолжаем
            }
            if (c == null) {
                // 3. Искать самостоятельно
                c = findClass(name);
            }
        }
        if (resolve) resolveClass(c);
        return c;
    }
}
```

**Зачем нужна делегация?**
- **Безопасность**: нельзя подменить `java.lang.String` своей реализацией — Bootstrap всегда загружает её первым
- **Консистентность**: один класс = один `Class` объект в рамках одного CL
- **Изоляция**: разные ClassLoader могут загружать одноимённые классы независимо

## 3. ClassLoader API

### Ключевые методы

|Метод|Описание|
|---|---|
|`loadClass(String name)`|Загружает класс по имени (с делегацией)|
|`findClass(String name)`|Ищет класс **только в этом CL** (без делегации)|
|`defineClass(byte[] b, ...)`|Создаёт `Class` из массива байт (bytecode)|
|`findLoadedClass(String name)`|Проверяет, уже ли загружен класс|
|`getResource(String name)`|Ищет ресурс в classpath/modulepath|
|`getParent()`|Возвращает родительский CL|

### Правило переопределения

**Переопределяйте `findClass()`, а не `loadClass()`** — так сохраняется делегация:

```java
public class UrlClassLoaderExample extends ClassLoader {
    private final Path classDir;

    public UrlClassLoaderExample(Path classDir, ClassLoader parent) {
        super(parent);
        this.classDir = classDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String path = name.replace('.', '/') + ".class";
        try {
            byte[] bytes = Files.readAllBytes(classDir.resolve(path));
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

## 4. Кастомный ClassLoader: пример плагинной системы

```java
public class PluginLoader extends ClassLoader {
    private final Path pluginJar;

    public PluginLoader(Path pluginJar) {
        // parent = System ClassLoader (делегация для JDK/app классов)
        super(ClassLoader.getSystemClassLoader());
        this.pluginJar = pluginJar;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try (JarFile jar = new JarFile(pluginJar.toFile())) {
            String entryName = name.replace('.', '/') + ".class";
            JarEntry entry = jar.getJarEntry(entryName);
            if (entry == null) throw new ClassNotFoundException(name);

            byte[] bytes = jar.getInputStream(entry).readAllBytes();
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}

// Использование:
ClassLoader pluginCL = new PluginLoader(Path.of("plugin.jar"));
Class<?> pluginClass = pluginCL.loadClass("com.plugin.MyPlugin");
Object instance = pluginClass.getDeclaredConstructor().newInstance();
```

**Горячая перезагрузка**: для перезагрузки класса нужно создать **новый** `ClassLoader` — старый класс выгрузится, когда станет недостижимым его ClassLoader.

## 5. Связь с JPMS (Java 9+ Module System)

В Java 9+ Module System (Project Jigsaw) изменила иерархию CL:

```
Bootstrap CL     → загружает java.base module (core)
Platform CL      → загружает platform modules (java.sql, java.xml, ...)
Application CL   → загружает app module (classpath / modulepath)
```

**Ключевые изменения:**
- `Extension ClassLoader` → `Platform ClassLoader`
- `rt.jar`, `tools.jar` убраны — заменены модульной структурой
- ClassLoader.loadClass() работает по-прежнему, но с учётом модульных границ
- `--add-opens`, `--add-exports` для доступа к непубличным модульным API

```java
// Получить модуль класса:
Module module = String.class.getModule();
System.out.println(module.getName()); // "java.base"
System.out.println(module.isNamed()); // true
```

## 6. Когда классы выгружаются

Класс **выгружается из Metaspace** только когда:
1. Его `ClassLoader` объект стал **недостижимым** для GC
2. Все классы, загруженные этим CL, стали недостижимыми
3. Все `Class` объекты этих классов стали недостижимыми

```
ClassLoader → [Class A, Class B, Class C]
                  ↑
           если ClassLoader недостижим →
           GC выгружает все его классы из Metaspace
```

**Практически**: это происходит в OSGi, контейнерах приложений (Tomcat), hot-deploy. Стандартный App ClassLoader никогда не выгружается — его классы живут до завершения JVM.

**ClassLoader leaks** (утечки памяти в Metaspace):
- Происходят когда кастомный CL стал "outdated", но на него есть ссылки из ThreadLocal, static поля, JDBC drivers, логирование (log4j)
- Симптом: `OutOfMemoryError: Metaspace` при hot-deploy
- Диагностика: `jmap -histo <pid>` → поиск `ClassLoader` объектов; JFR `jdk.ClassLoaderStatistics`

## 7. NoClassDefFoundError vs ClassNotFoundException

|Ошибка|Когда возникает|Причина|
|---|---|---|
|`ClassNotFoundException`|`Class.forName()`, `loadClass()` — явная загрузка|Класс не найден в classpath/modulepath|
|`NoClassDefFoundError`|JVM пытается загрузить класс, нужный для другого класса|Класс был в compile-time, но отсутствует в runtime|

```java
// ClassNotFoundException — контролируемая:
try {
    Class<?> cls = Class.forName("com.example.Missing");
} catch (ClassNotFoundException e) {
    // обрабатываем
}

// NoClassDefFoundError — неконтролируемая Error:
// class A { B b = new B(); }  — если B.class отсутствует в runtime
// → java.lang.NoClassDefFoundError: com/example/B
```

**Типичный сценарий `NoClassDefFoundError`:**
- Класс успешно скомпилирован (зависимость есть в compile classpath)
- В runtime зависимость отсутствует (war/jar без нужной библиотеки)
- Или: класс был загружен, но его статический инициализатор бросил исключение → класс помечается как "failed" → повторная загрузка даёт `NoClassDefFoundError`

## 8. Thread Context ClassLoader

Проблема: библиотеки (JNDI, JDBC, SPI) загружаются Bootstrap или Platform CL, но им нужно загружать классы из Application classpath (драйверы, провайдеры).

**Решение — Thread Context ClassLoader:**

```java
// По умолчанию = System ClassLoader
ClassLoader tccl = Thread.currentThread().getContextClassLoader();

// Библиотека использует TCCL вместо своего CL:
Class<?> driver = tccl.loadClass("com.mysql.cj.jdbc.Driver");

// Смена TCCL (например, в контейнерах приложений):
Thread.currentThread().setContextClassLoader(webAppClassLoader);
try {
    // выполнение кода с контекстом веб-приложения
} finally {
    Thread.currentThread().setContextClassLoader(originalCL);
}
```

## Interview Q&A (Senior Level)

**Q1: Почему `String.class.getClassLoader()` возвращает `null`, а не объект ClassLoader?**

> `null` — это соглашение JVM для обозначения **Bootstrap ClassLoader**. Bootstrap CL реализован на C++ как часть JVM и не представлен Java-объектом. `Class.getClassLoader()` возвращает `null` для любого класса, загруженного Bootstrap CL (все классы `java.lang`, `java.util`, и т.д.). При написании кода, работающего с ClassLoader, всегда проверяйте: `cl != null ? cl : ClassLoader.getSystemClassLoader()`.

**Q2: Чем отличается переопределение `loadClass()` от переопределения `findClass()`? Что предпочтительнее?**

> `loadClass()` реализует весь алгоритм: проверка кэша → делегация родителю → `findClass()`. Переопределение `loadClass()` ломает delegation model — можно случайно загрузить `java.lang.String` собственным кодом (угроза безопасности). `findClass()` — точка расширения: вызывается только когда родители не нашли класс. Переопределяйте **всегда `findClass()`**. Исключение: OSGi и подобные системы сознательно переопределяют `loadClass()` для реализации child-first delegation.

**Q3: Как ClassLoader связан с утечками памяти в Metaspace при hot-deploy?**

> Метаданные класса живут в Metaspace пока жив его ClassLoader. При hot-deploy создаётся новый CL с новыми классами. Если старый CL остаётся достижимым (например, через статическое поле драйвера JDBC, ThreadLocal, или shutdown hook), GC не может выгрузить его классы из Metaspace. При каждом redeploy добавляются новые классы → Metaspace заполняется → `OutOfMemoryError: Metaspace`. Диагностика: `jmap -histo <pid>` → ищем множество экземпляров ClassLoader. Решение: явно обнулять статические ссылки, отменять регистрацию драйверов, очищать ThreadLocal.

**Q4: Что такое "child-first" ClassLoader и зачем он нужен в контейнерах приложений?**

> Стандартная delegation model — "parent-first": родительский CL имеет приоритет. Контейнеры (Tomcat, JBoss) реализуют "child-first" (WebApp CL) для **изоляции веб-приложений**: сначала ищем класс в WAR, только если не нашли — делегируем родителю. Это позволяет разным приложениям использовать разные версии одной библиотеки (например, Hibernate 5 и Hibernate 6 одновременно). Без child-first: первая загруженная версия библиотеки "выигрывает" для всех приложений в контейнере.

**Q5: Можно ли перезагрузить класс в runtime без перезапуска JVM? Как?**

> Да, через новый ClassLoader. `ClassLoader` кэширует загруженные классы (`findLoadedClass`), поэтому один CL не может загрузить один класс дважды. Алгоритм hot-reload: (1) создать новый `PluginClassLoader`; (2) загрузить обновлённый `MyClass` через `defineClass(newBytecode)`; (3) создать экземпляр нового класса; (4) убрать ссылки на старый CL → GC выгружает старый класс из Metaspace. Проблема: объекты старого и нового классов несовместимы по типу — нужен общий интерфейс, загружаемый через App CL.

## Связанные темы

- [[Java Memory Structure]] — Metaspace и выгрузка классов
- [[Java Reflection API]] — работа с ClassLoader через рефлексию
- [[Java Assembling]] — classpath, modulepath, jar, JPMS
