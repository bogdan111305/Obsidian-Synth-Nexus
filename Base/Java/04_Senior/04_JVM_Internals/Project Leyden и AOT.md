# Project Leyden и AOT

> **Project Leyden** (JEP 483, Java 24 preview) — AOT (Ahead-of-Time) компиляция и снапшоты классов для JVM. Цель: ускорить startup и warm-up Java приложений до уровня нативных. **AppCDS** (Application Class-Data Sharing) — предшественник: кэш загруженных классов между запусками.

## Связанные темы
[[ClassLoaders]], [[JVM Startup и AppCDS]], [[JIT Compiler & Optimizations]], [[Сборка и запуск Java-приложений]]

---

## Проблема: Java Startup

```
Стандартный запуск Spring Boot приложения:
  JVM startup:      ~50 мс
  Class loading:    ~500 мс   ← тысячи классов
  JIT warmup:       ~2-5 сек  ← горячие пути компилируются в нативный код
  App initialization: ~3-10 сек

Итого: 5-15 секунд до "ready to serve"

Проблема в serverless/K8s: каждый cold start = задержка
GraalVM Native Image решает это, но ограничивает динамичность Java
```

## AppCDS — Class Data Sharing

**AppCDS** (Application Class-Data Sharing, Java 10+) кэширует результат загрузки и парсинга классов в shared archive:

```bash
# Шаг 1: Создать список классов (запустить приложение и записать)
java -XX:DumpLoadedClassList=app.classlist -jar app.jar

# Шаг 2: Создать архив
java -Xshare:dump -XX:SharedClassListFile=app.classlist \
     -XX:SharedArchiveFile=app.jsa -jar app.jar

# Шаг 3: Запустить с архивом
java -Xshare:on -XX:SharedArchiveFile=app.jsa -jar app.jar

# Экономия: ~200-500 мс на class loading
```

**Dynamic AppCDS** (Java 13+): не нужен отдельный шаг dump — архив создаётся автоматически при завершении:

```bash
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar  # запись
java -XX:SharedArchiveFile=app.jsa -jar app.jar      # использование
```

## Project Leyden: Ahead-of-Time Class Loading & Linking

**JEP 483 (Java 24)** — "Ahead-of-Time Class Loading & Linking":

```bash
# Training run: записываем что делает JVM при запуске
java -XX:AOTMode=record -XX:AOTConfiguration=app.aotconf -jar app.jar

# AOT assembly: компилируем данные в cache
java -XX:AOTMode=create -XX:AOTConfiguration=app.aotconf \
     -XX:AOTCache=app.aot -jar app.jar

# Production run: с кэшем
java -XX:AOTCache=app.aot -jar app.jar
```

Что кэшируется в AOT cache:
- Результаты class loading + linking + verification
- Результаты JVM initialization  
- (В будущем) AOT-скомпилированный машинный код от JIT

**Ускорение**: до 40-50% быстрее запуск, JIT warmup начинается раньше.

## Layered AOT Caches

```
Layer 0: JDK cache (rt.jar, java.base, стандартные классы)
          └── создаётся один раз при установке JDK
Layer 1: Framework cache (Spring Boot, Hibernate классы)
          └── создаётся при сборке образа
Layer 2: Application cache (бизнес-классы)
          └── создаётся при деплое
```

Каждый слой базируется на предыдущем. Нижние слои переиспользуются между разными приложениями.

## Leyden vs GraalVM Native Image

| Характеристика | Leyden AOT | GraalVM Native Image |
|----------------|-----------|---------------------|
| Bytecode vs native | JVM bytecode + JIT | Полностью нативный |
| Динамичность | Сохраняется | Ограничена (reflection конфиг) |
| Startup | ~200-500 мс (с кэшем) | ~10-50 мс |
| Peak throughput | Полный JIT | Ниже (без JIT профилирования) |
| Footprint | JVM + heap | Маленький |
| Совместимость | 100% Java | Ограничена |
| Зрелость | Java 24 preview | Mature (GraalVM CE/EE) |

**Вывод**: Leyden — для стандартных Java приложений где нужно ускорить startup без отказа от динамичности. GraalVM NI — для serverless/CLI где startup <100 мс критичен.

## JEP 483 — Технические детали

Три фазы AOT preparation:
1. **Training** (`AOTMode=record`): запускаем приложение, JVM записывает все class loading events, linking decisions, initialization order
2. **Assembly** (`AOTMode=create`): JVM replays training data, создаёт snapshot heap + кэш загруженных классов
3. **Production** (`AOTMode=on`/через `AOTCache`): JVM загружает snapshot вместо повторного class loading

## CDS — что реально кэшируется

```
Class-Data Sharing archive содержит:
  ├── Class metadata (bytecode, constant pool, vtable)
  ├── Resolved references (symbolic → direct pointer)
  ├── Heap objects (interned strings, Class mirrors)
  └── (Leyden) JIT compilation artifacts (будущее)

НЕ кэшируется:
  ├── Instance данные приложения (heap)
  ├── JNI bindings
  └── Runtime-generated classes (Proxy, lambda)
```

## Spring AOT + Leyden

Spring Boot 3.x + AOT mode:

```xml
<!-- pom.xml -->
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <configuration>
    <aot>true</aot>
  </configuration>
</plugin>
```

```bash
# Spring AOT генерирует статический код вместо рефлексии
# + Leyden AOTCache = быстрый startup без GraalVM
mvn spring-boot:process-aot
java -XX:AOTCache=spring.aot -jar app.jar
```

---

## Вопросы на интервью
- Что такое AppCDS и как он ускоряет запуск?
- Чем Project Leyden отличается от GraalVM Native Image?
- Что такое "training run" в контексте Leyden AOT?
- Почему обычная Java медленно стартует?
- Когда выбрать Leyden, а когда GraalVM Native Image?

## Подводные камни
- AppCDS архив привязан к конкретному JDK build — обновление JDK = пересоздание архива
- Leyden AOT (Java 24) — Preview feature, API может измениться
- Если training run не покрывает все code paths → некоторые классы не в кэше → частичное ускорение
- Dynamic class generation (Proxy, CGLIB, ByteBuddy) обычно не кэшируется — Spring AOT заменяет их статическим кодом
