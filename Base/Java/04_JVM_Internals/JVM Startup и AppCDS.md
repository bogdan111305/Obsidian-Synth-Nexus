# JVM Startup и AppCDS

> JVM startup — загрузка и инициализация классов, JIT-прогрев, до первого запроса. **CDS (Class Data Sharing)** — кэш предварительно обработанных классов в mmap-регионе. **AppCDS** (Java 10+) — расширение CDS на приложение. Цель: сократить время старта с секунд до сотен миллисекунд.

## Связанные темы
[[ClassLoaders]], [[JIT Compiler & Optimizations]], [[Project Leyden и AOT]], [[Java Memory Structure]]

---

## Фазы запуска JVM

```
1. JVM инициализация (~10-50ms)
   ├── Загрузка libjvm.so
   ├── Инициализация heap, GC
   └── Загрузка bootstrap classloader

2. Загрузка классов (~100-500ms для Spring Boot)
   ├── Bootstrap: java.lang.*, java.util.*
   ├── System: classpath классы
   └── Application: jar-файлы
   Каждый класс: чтение .class → парсинг → верификация → разрешение → инициализация

3. JIT прогрев (~секунды)
   ├── Интерпретация до порога C1 (~2000 вызовов)
   ├── C1 компиляция
   └── C2 компиляция (~15000 вызовов) — пиковая производительность

4. Первый запрос
   ├── Возможно до завершения JIT прогрева
   └── Latency spike на первых запросах — "cold start"
```

---

## Class Data Sharing (CDS)

CDS предварительно обрабатывает системные классы и сохраняет их в shared archive (`.jsa`):

```bash
# JVM автоматически использует CDS для JDK классов:
# $JAVA_HOME/lib/server/classes.jsa — стандартный архив

# Проверить, используется ли CDS:
java -Xshare:on -verbose:class -cp . Main 2>&1 | grep "Loaded"

# Режимы:
-Xshare:on     # использовать (ошибка если архив не найден)
-Xshare:auto   # использовать если есть (default)
-Xshare:off    # не использовать
-Xshare:dump   # создать архив
```

**Что CDS даёт:**
- Классы загружаются из mmap (memory-mapped) региона — нет парсинга
- Один mmap-регион разделяется между JVM-процессами → экономия RAM в контейнерах
- Ускорение загрузки системных классов: ~100-300ms

---

## AppCDS — Application Class Data Sharing (Java 10+)

AppCDS расширяет CDS на классы приложения:

```bash
# Шаг 1: Записать список загруженных классов
java -XX:DumpLoadedClassList=classes.lst -cp app.jar com.example.Main

# Шаг 2: Создать архив
java -Xshare:dump \
     -XX:SharedClassListFile=classes.lst \
     -XX:SharedArchiveFile=app-cds.jsa \
     -cp app.jar

# Шаг 3: Запуск с архивом
java -Xshare:on \
     -XX:SharedArchiveFile=app-cds.jsa \
     -cp app.jar com.example.Main

# Результат: -300 до -1000ms от startup time
```

**Spring Boot 3.3+ — встроенная поддержка AppCDS:**

```bash
# В Maven/Gradle через spring-boot:build-image:
./mvnw spring-boot:build-image \
  -Dspring-boot.build-image.imageName=myapp \
  -Dspring-boot.build-image.pullPolicy=IF_NOT_PRESENT

# Dockerfile с AppCDS:
FROM eclipse-temurin:21-jdk as builder
COPY app.jar app.jar
RUN java -XX:ArchiveClassesAtExit=app-cds.jsa -jar app.jar &
RUN sleep 30 && kill %1 || true

FROM eclipse-temurin:21-jre
COPY --from=builder app.jar .
COPY --from=builder app-cds.jsa .
ENTRYPOINT ["java", "-XX:SharedArchiveFile=app-cds.jsa", "-jar", "app.jar"]
```

---

## Dynamic CDS Archive (Java 13+)

Динамический архив создаётся при **завершении** JVM-процесса — не требует предварительного списка классов:

```bash
# Один шаг: архив создаётся при выходе из JVM
java -XX:ArchiveClassesAtExit=dynamic.jsa -jar app.jar

# Запуск с динамическим архивом:
java -XX:SharedArchiveFile=dynamic.jsa -jar app.jar
```

**Минус:** нужно запустить приложение хотя бы раз, загрузить нужные классы, потом завершить. Для приложений с lazy loading не все классы попадут в архив.

---

## JVM Startup Optimization: инструменты

```bash
# Профиль startup:
java -Xlog:class+load=info:file=class-load.log -jar app.jar
# Показывает каждый загруженный класс с временем

# Benchmark startup time:
hyperfine 'java -jar app.jar' --warmup 3

# Spring Boot Actuator — startup steps:
# GET /actuator/startup → список шагов инициализации с временем
```

**Типичные hotspots при старте:**
- Classpath scanning (Spring component scan) — сотни мс
- Reflection-based DI — сотни мс
- Загрузка JDBC-драйверов → статические блоки
- Hibernate schema validation

---

## Вопросы на интервью

- Какие фазы проходит JVM при запуске?
- Что такое CDS? Как оно ускоряет старт?
- Чем AppCDS отличается от базового CDS?
- Что такое Dynamic CDS Archive? Как создать?
- Как AppCDS работает с Docker/контейнерами?
- Что такое JIT-прогрев и как он влияет на latency первых запросов?

## Подводные камни

- **AppCDS инвалидируется при изменении classpath** — любое изменение jar → нужно пересоздать архив. В CI/CD добавить шаг генерации архива.
- **mmap ≠ загрузка в RAM** — CDS использует memory-mapped файл. Реальная загрузка в RAM происходит при page fault. Польза — нет парсинга и верификации, а не нулевое I/O.
- **Архив привязан к JVM-версии** — архив для JDK 21 не работает с JDK 21.0.1 (другой build). В контейнерах используй тот же JDK-образ для сборки и запуска.
- **Spring Boot с CDS: осторожно с @Lazy** — классы, загружаемые лениво, не попадут в архив при генерации. Для полного покрытия нужно прогреть все endpoint'ы при генерации архива.
