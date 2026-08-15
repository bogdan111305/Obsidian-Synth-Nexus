# Java Agents & Instrumentation API

> **Java Agent** (`-javaagent:agent.jar`) подключается к JVM до или во время работы, трансформирует байт-код через `ClassFileTransformer`. Основа APM (Datadog, Dynatrace), AOP (AspectJ LTW), JaCoCo, покрытия кода.
> На интервью: premain vs agentmain, почему Advice лучше subclassing, Bridge Methods фильтрация, ограничения redefineClasses, SafePoint и STW при ретрансформации.

## Связанные темы
[[Java Reflection API]], [[ClassLoaders]], [[JIT Compiler & Optimizations]], [[JVM Profiling & Observability]]

---

## Два режима загрузки агента

### Static Agent — premain

Загружается **до** `main()` приложения через `-javaagent:`.

```java
// MANIFEST.MF:
// Premain-Class: com.example.MyAgent
// Can-Retransform-Classes: true
// Can-Redefine-Classes: true

public class MyAgent {
    public static void premain(String args, Instrumentation inst) {
        inst.addTransformer(new MyTransformer(), true); // true = canRetransform
        // inst.retransformClasses(SomeClass.class); — ретрансформация уже загруженных
    }
}
```

```bash
java -javaagent:myagent.jar=param1=value1 -jar myapp.jar
```

### Dynamic Agent — agentmain (Attach API)

Загружается в **уже работающую** JVM. Используется профилировщиками (async-profiler, JProfiler).

```java
// В агенте (MANIFEST.MF: Agent-Class: com.example.MyAgent):
public class MyAgent {
    public static void agentmain(String args, Instrumentation inst) {
        inst.addTransformer(new MyTransformer(), true);
        inst.retransformClasses(TargetClass.class);
    }
}

// Аттачинг из отдельного процесса:
VirtualMachine vm = VirtualMachine.attach(pid); // pid как String
vm.loadAgent("/path/to/agent.jar", "args");
vm.detach();
```

Attach API требует прав ОС на целевой процесс. Self-attach: `-Djdk.attach.allowAttachSelf=true`.

---

## Instrumentation API

```java
public interface Instrumentation {
    void addTransformer(ClassFileTransformer transformer, boolean canRetransform);

    // Пропустить уже загруженные классы через трансформаторы заново:
    void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;

    // Заменить класс новым байт-кодом полностью:
    void redefineClasses(ClassDefinition... definitions) throws ...;

    long getObjectSize(Object obj);       // shallow size только!
    Class<?>[] getAllLoadedClasses();
}
```

### ClassFileTransformer

```java
public class TimingTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(
        ClassLoader loader,
        String className,              // "com/example/MyService" — с '/', не '.'
        Class<?> classBeingRedefined,
        ProtectionDomain domain,
        byte[] classfileBuffer
    ) throws IllegalClassFormatException {
        if (!className.startsWith("com/example/")) return null; // null = без изменений
        return modifiedBytes; // трансформированный байт-код
    }
}
```

---

## Low-Level: ASM

Visitor pattern на уровне байт-кода. Требует ручного управления стеком операндов и фреймами.

```java
public class MethodTimingTransformer extends ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className,
                            Class<?> redefined, ProtectionDomain pd, byte[] bytes) {
        if (!className.equals("com/example/UserService")) return null;

        ClassReader reader = new ClassReader(bytes);
        ClassWriter writer = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

        ClassVisitor visitor = new ClassVisitor(Opcodes.ASM9, writer) {
            @Override
            public MethodVisitor visitMethod(int access, String name,
                                             String desc, String signature, String[] ex) {
                // Фильтр bridge-методов! Иначе дважды перехватишь один логический метод
                if ((access & Opcodes.ACC_BRIDGE) != 0) {
                    return super.visitMethod(access, name, desc, signature, ex);
                }
                MethodVisitor mv = super.visitMethod(access, name, desc, signature, ex);
                return new MethodEnterVisitor(mv, name);
            }
        };

        reader.accept(visitor, ClassReader.EXPAND_FRAMES);
        return writer.toByteArray();
    }
}

class MethodEnterVisitor extends MethodVisitor {
    private final String methodName;

    MethodEnterVisitor(MethodVisitor mv, String name) {
        super(Opcodes.ASM9, mv);
        this.methodName = name;
    }

    @Override
    public void visitCode() {
        super.visitCode();
        // Инжектируем: System.out.println("Entering: " + methodName)
        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn("Entering: " + methodName);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream",
                           "println", "(Ljava/lang/String;)V", false);
    }
}
```

---

## High-Level: ByteBuddy

Высокоуровневое API. Используется в Mockito, Hibernate, Spring Data. Не требует знания байт-кода.

### Subclassing (для новых объектов)

```java
Class<?> dynamicType = new ByteBuddy()
    .subclass(Object.class)
    .method(named("toString"))
    .intercept(FixedValue.value("Hello from ByteBuddy!"))
    .make()
    .load(getClass().getClassLoader())
    .getLoaded();
```

Ограничение: не работает для `final` классов и уже созданных объектов.

### Advice — инструментирование существующих классов

Встраивает код прямо в байт-код метода (inline). Работает с `final` классами. Нет дополнительных allocations — идеально для APM агентов с overhead < 1%.

```java
public class ServiceAdvice {
    @Advice.OnMethodEnter
    static long enter(@Advice.Origin String method) {
        return System.nanoTime();
    }

    @Advice.OnMethodExit(onThrowable = Throwable.class)
    static void exit(@Advice.Origin String method,
                     @Advice.Enter long startTime,
                     @Advice.Thrown Throwable thrown) {
        long elapsed = System.nanoTime() - startTime;
        if (thrown != null) {
            System.out.printf("ERROR in %s after %d µs: %s%n", method, elapsed / 1000, thrown.getMessage());
        } else {
            System.out.printf("%s took %d µs%n", method, elapsed / 1000);
        }
    }
}

// В premain:
public static void premain(String args, Instrumentation inst) {
    new AgentBuilder.Default()
        .type(nameStartsWith("com.example"))
        .transform((builder, type, loader, module, domain) ->
            builder.visit(Advice.to(ServiceAdvice.class).on(isMethod().and(isPublic())))
        )
        .installOn(inst);
}
```

---

## Bridge Methods — ловушка для агентов

Bridge методы генерирует компилятор для поддержки type erasure и ковариантных возвращаемых типов:

```java
interface Processor<T> { T process(T input); }

class StringProcessor implements Processor<String> {
    @Override
    public String process(String input) { return input.toUpperCase(); }
    // Компилятор добавляет синтетический bridge:
    // public Object process(Object input) { return process((String) input); }
    // Флаги: ACC_BRIDGE + ACC_SYNTHETIC
}
```

**Правило:** всегда фильтруй bridge-методы в `ClassVisitor.visitMethod()`:
```java
if ((access & (ACC_BRIDGE | ACC_SYNTHETIC)) == (ACC_BRIDGE | ACC_SYNTHETIC)) {
    return super.visitMethod(...); // пропустить без инструментирования
}
```

Без фильтрации один логический вызов `process("hello")` будет перехвачен дважды.

---

## redefineClasses vs retransformClasses

```
retransformClasses — пропускает уже загруженный класс через все зарегистрированные трансформаторы
redefineClasses    — заменяет класс новым байт-кодом (не через трансформаторы, напрямую)

Оба: STW SafePoint пауза на время трансформации

НЕЛЬЗЯ через оба:
  - Добавить / удалить поля
  - Добавить / удалить методы
  - Изменить иерархию (extends, implements)
  - Изменить имя класса

МОЖНО:
  - Изменить тела методов
  - Изменить аннотации
  - Изменить видимость методов
```

---

## Практические применения

**APM-агенты** (Datadog, New Relic, OpenTelemetry) — инструментируют Spring MVC/Servlet, JDBC, Kafka без изменения кода.

**Измерение размера объектов:**
```java
public static void premain(String args, Instrumentation inst) {
    InstrumentationHolder.instrumentation = inst;
}

// getObjectSize = shallow size (только сам объект, без referenced объектов):
System.out.println(sizeOf(new ArrayList<>())); // ~40 байт (заголовок + 3 поля)
System.out.println(sizeOf("Hello"));           // ~56 байт (заголовок + char[] ref + hash)

// Для deep size нужно рекурсивно обходить граф через Reflection + IdentityHashMap
// Библиотека JAMM делает это автоматически
```

**Hot Reload** (изменение тел методов без рестарта):
```java
byte[] newBytecode = compileUpdatedClass("MyService.java");
inst.redefineClasses(new ClassDefinition(MyService.class, newBytecode));
// Только изменение тел — поля и методы нельзя добавить/удалить
```

---

## Вопросы на интервью

- Чем premain отличается от agentmain? Когда использовать каждый?
- Почему ByteBuddy Advice предпочтительнее subclassing для APM-агентов?
- Что такое Bridge Method? Почему агент должен их фильтровать?
- Чем redefineClasses отличается от retransformClasses? Что нельзя изменить через оба?
- Почему `getObjectSize()` возвращает shallow size? Как получить deep size?
- Что такое SafePoint? Почему частая ретрансформация опасна в продакшне?
- Как Attach API позволяет подключить агент к работающей JVM?

---

## Подводные камни

- **Bridge Methods без фильтрации** — инструментируешь и `process(String)` и синтетический `process(Object)`, таймеры дублируются, результаты некорректны. Всегда: `(access & ACC_BRIDGE) != 0 → skip`.
- **retransformClasses = STW** — трансформация в рантайме через `retransformClasses()` вызывает SafePoint и паузу всех потоков. На больших heap с многими загруженными классами пауза может быть значительной. Трансформируй в `premain`, не в рантайме.
- **redefineClasses не добавляет методы** — горячая замена работает только для изменения тел существующих методов. Попытка добавить метод → `UnsupportedOperationException`. Spring DevTools/JRebel обходят это через новый ClassLoader, а не redefineClasses.
- **Subclassing не работает с `final`** — ByteBuddy `.subclass()` не может субклассировать `final` классы (String, etc.). Используй `Advice` или ASM для direct bytecode injection.
- **Advice code в production-classpath** — класс `ServiceAdvice` должен быть доступен в агентском jar, не в приложении. Иначе `ClassNotFoundException` при инструментировании. Используй isolated classloader или shadow jar.
- **ClassLoader isolation для трансформатора** — трансформатор вызывается для каждого загружаемого класса, включая классы JDK. Всегда проверяй `className` в начале `transform()` и возвращай `null` для чужих классов, иначе агент инструментирует внутренности JDK.
