---
title: "Java Date and Time — java.time API"
tags: [java, date, time, localdate, zoneddatetime, datetimeformatter, java8]
updated: 2026-03-04
---

# Java Date and Time (java.time API)

> [!QUOTE] Суть
> Современный API (`java.time`, Java 8+): **иммутабельный**, thread-safe. `LocalDate` (дата без времени), `LocalDateTime` (без TZ), `ZonedDateTime` (с TZ), `Instant` (Unix timestamp), `Duration`/`Period` (интервалы). Старые `Date`/`Calendar` — **устарели**, мутабельны, не thread-safe.

> [!WARNING] Ловушка: SimpleDateFormat не thread-safe
> `SimpleDateFormat` — НЕ thread-safe. Sharing одного инстанса между потоками → race condition → некорректные даты. Используй `DateTimeFormatter` (java.time, иммутабельный и thread-safe) или создавай `SimpleDateFormat` в каждом потоке отдельно.

Java предоставляет мощные инструменты для работы с датами и временем, которые значительно эволюционировали с появлением пакета `java.time` в Java 8. Это руководство охватывает как устаревший, так и современный API, их ключевые классы, практические примеры и лучшие практики для работы с датами, временем, часовыми поясами, интервалами и форматированием.
## Устаревший API дат и времени (до Java 8)

Оригинальный API Java для работы с датами и временем, включающий в основном `java.util.Date` и `java.util.Calendar`, считается устаревшим из-за ряда ограничений. Хотя он всё ещё работает, для новых проектов рекомендуется использовать современный API `java.time`.
### Ключевые классы

- `java.util.Date`: Представляет конкретный момент времени, объединяя дату и время. Однако большинство методов (например, `getYear()`, `getMonth()`) устарели и склонны к ошибкам из-за особенностей, таких как нумерация месяцев с нуля.
- `java.util.Calendar`: Более гибкая альтернатива `Date`, но API громоздкий, сложный и неинтуитивный.
- `java.text.SimpleDateFormat`: Используется для форматирования и парсинга дат, но не является потокобезопасным.
### Ограничения

- Мутабельность: `Date` и `Calendar` изменяемы, что может привести к ошибкам в многопоточных приложениях.
- Потокобезопасность: Ни `Date`, ни `SimpleDateFormat` не являются потокобезопасными, что требует ручной синхронизации.
- Работа с часовыми поясами: Неоднозначное и непоследовательное поведение при работе с часовыми поясами.
- Плохой дизайн API: Методы запутаны, а некоторые операции (например, добавление дней) неудобны.
### Пример (устаревший API)

```java
// Создание объекта Date
Date date = new Date(); // Текущая дата и время
System.out.println(date); // Например, Wed Jul 02 19:31:00 CEST 2025

// Использование Calendar
Calendar calendar = Calendar.getInstance();
calendar.set(2025, Calendar.JULY, 2); // Месяцы начинаются с 0!
Date oldDate = calendar.getTime();
```
### Почему избегать?

Из-за сложности и отсутствия потокобезопасности устаревший API лучше избегать в пользу современного `java.time`.
## Современный API дат и времени (Java 8 и новее)

Введённый в Java 8 пакет `java.time` (вдохновлённый Joda-Time) является неизменяемым, потокобезопасным и предоставляет понятный, интуитивный API для работы с датами, временем и часовыми поясами. Это рекомендуемый подход для всех операций с датами и временем.
### Ключевые классы

|Класс|Назначение|
|---|---|
|`LocalDate`|Представляет дату без времени (например, `2025-07-02`)|
|`LocalTime`|Представляет время без даты (например, `19:31:00`)|
|`LocalDateTime`|Комбинирует дату и время без часового пояса (например, `2025-07-02T19:31:00`)|
|`ZonedDateTime`|Дата и время с часовым поясом (например, `2025-07-02T19:31:00+02:00[Europe/Paris]`)|
|`Instant`|Точка во времени в UTC (например, `2025-07-02T17:31:00Z`)|
|`Duration`|Интервал времени (например, часы, минуты, секунды)|
|`Period`|Интервал дат (например, годы, месяцы, дни)|
|`DateTimeFormatter`|Форматирование и парсинг дат и времени|
### Преимущества

- Неизменяемость: Все классы неизменяемы, что обеспечивает потокобезопасность и предсказуемое поведение.
- Понятный API: Интуитивные методы, такие как `plusDays()`, `minusMonths()`, `isBefore()`.
- Поддержка часовых поясов: Надёжная работа с часовыми поясами через `ZoneId` и `ZoneOffset`.
- Стандартизация: Соответствует стандартам ISO-8601 для представления дат и времени.
## Практические примеры

### Создание дат и времени

```java
// Текущая дата и время
LocalDate date = LocalDate.now();           // Например, 2025-07-02
LocalTime time = LocalTime.now();           // Например, 19:31:00
LocalDateTime dateTime = LocalDateTime.now(); // Например, 2025-07-02T19:31:00

// Конкретная дата
LocalDate specificDate = LocalDate.of(2025, Month.DECEMBER, 25); // 2025-12-25
LocalDateTime specificDateTime = LocalDateTime.of(2025, 7, 2, 19, 31); // 2025-07-02T19:31
```
### Форматирование дат

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd.MM.yyyy");
String formattedDate = date.format(formatter); // Например, 02.07.2025

// С временем и часовым поясом
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Europe/Paris"));
DateTimeFormatter zdtFormatter = DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm:ss z");
String formattedZdt = zdt.format(zdtFormatter); // Например, 02.07.2025 19:31:00 CEST
```
### Парсинг дат

```java
String dateStr = "02.07.2025";
LocalDate parsedDate = LocalDate.parse(dateStr, formatter); // 2025-07-02

String zdtStr = "02.07.2025 19:31:00 CEST";
ZonedDateTime parsedZdt = ZonedDateTime.parse(zdtStr, zdtFormatter); // 2025-07-02T19:31:00+02:00[Europe/Paris]
```
### Работа с часовыми поясами

```java
// Текущее время в конкретном часовом поясе
ZonedDateTime parisTime = ZonedDateTime.now(ZoneId.of("Europe/Paris")); // Например, 2025-07-02T19:31:00+02:00[Europe/Paris]

// Конвертация в другой часовой пояс
ZonedDateTime newYorkTime = parisTime.withZoneSameInstant(ZoneId.of("America/New_York")); // Например, 2025-07-02T13:31:00-04:00[America/New_York]
```
### Вычисление разницы

```java
// Разница между датами
LocalDate startDate = LocalDate.of(2023, 1, 1);
LocalDate endDate = LocalDate.now();
Period period = Period.between(startDate, endDate);
System.out.println("Лет: " + period.getYears() + ", Месяцев: " + period.getMonths() + ", Дней: " + period.getDays());

// Разница между временными точками
Instant start = Instant.now();
try { Thread.sleep(1000); } catch (InterruptedException e) {}
Instant end = Instant.now();
Duration duration = Duration.between(start, end);
System.out.println("Миллисекунды: " + duration.toMillis()); // ~1000
```
### Добавление/вычитание времени

```java
LocalDate tomorrow = date.plusDays(1);      // 2025-07-03
LocalDate nextMonth = date.plusMonths(1);   // 2025-08-02
LocalDateTime inOneHour = dateTime.plusHours(1); // 2025-07-02T20:31:00
```
## Подробный разбор ключевых классов

### ZonedDateTime: Дата, время и часовой пояс

`ZonedDateTime` идеально подходит для работы с датами и временем с учётом часового пояса. Он объединяет `LocalDateTime`, `ZoneId` и `ZoneOffset`.
#### Создание

```java
// Текущее время в системном часовом поясе
ZonedDateTime now = ZonedDateTime.now();

// Конкретный часовой пояс
ZonedDateTime tokyo = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));

// Из LocalDateTime
LocalDateTime local = LocalDateTime.of(2025, 7, 2, 19, 31);
ZonedDateTime zoned = local.atZone(ZoneId.of("Europe/Paris"));
```
#### Конвертации

```java
// В Instant (UTC)
Instant instant = zoned.toInstant();

// В другой часовой пояс
ZonedDateTime utcZoned = instant.atZone(ZoneId.of("UTC"));

// Извлечение компонентов
LocalDate localDate = zoned.toLocalDate();
LocalDateTime localDateTime = zoned.toLocalDateTime();
ZoneId zone = zoned.getZone(); // Например, Europe/Paris
ZoneOffset offset = zoned.getOffset(); // Например, +02:00
```
### Instant: Точка во времени

`Instant` представляет одну точку на временной шкале, измеряемую в наносекундах с момента начала эпохи Unix (1970-01-01T00:00:00Z).
#### Операции

```java
Instant now = Instant.now();
long epochMillis = now.toEpochMilli(); // Миллисекунды с начала эпохи

// Создание из epoch
Instant fromMillis = Instant.ofEpochMilli(1625212800000L);

// Конвертация в ZonedDateTime
ZonedDateTime zoned = now.atZone(ZoneId.of("Europe/Paris"));
```
### Duration: Интервалы времени

`Duration` измеряет временные интервалы в часах, минутах, секундах или наносекундах.

```java
// Измерение прошедшего времени
Instant start = Instant.now();
try { Thread.sleep(2000); } catch (InterruptedException e) {}
Instant end = Instant.now();
Duration duration = Duration.between(start, end);
System.out.println("Прошло: " + duration.toMillis() + " мс");

// Ручное создание
Duration fiveMinutes = Duration.ofMinutes(5);
Duration combined = fiveMinutes.plus(Duration.ofSeconds(30));
```
### Period: Интервалы дат

`Period` измеряет интервалы в терминах лет, месяцев и дней.

```java
LocalDate start = LocalDate.of(2023, 1, 1);
LocalDate end = LocalDate.of(2025, 7, 2);
Period period = Period.between(start, end);
System.out.println("Период: " + period.getYears() + " лет, " + period.getMonths() + " месяцев, " + period.getDays() + " дней");
```

### DateTimeFormatter: Форматирование и парсинг

`DateTimeFormatter` используется для форматирования дат в строки и парсинга строк в даты.
#### Форматирование

```java
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Europe/Paris"));
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm:ss z");
String formatted = zdt.format(fmt); // Например, 02.07.2025 19:31:00 CEST
```
#### Парсинг

```java
String dateStr = "02.07.2025 19:31:00 CEST";
ZonedDateTime parsed = ZonedDateTime.parse(dateStr, fmt); // 2025-07-02T19:31:00+02:00[Europe/Paris]
```
#### Предопределённые форматтеры

```java
DateTimeFormatter.ISO_LOCAL_DATE_TIME; // 2025-07-02T19:31:00
DateTimeFormatter.RFC_1123_DATE_TIME;  // Wed, 2 Jul 2025 19:31:00 +0200
DateTimeFormatter.BASIC_ISO_DATE;      // 20250702
```
## Конвертация между устаревшим и современным API

При работе с устаревшим кодом может потребоваться конвертация между старым и новым API.
### Из устаревшего в современный

```java
// Date в LocalDateTime
Date oldDate = new Date();
Instant instant = oldDate.toInstant();
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());

// Calendar в LocalDateTime
Calendar calendar = Calendar.getInstance();
LocalDateTime ldtFromCal = LocalDateTime.ofInstant(calendar.toInstant(), calendar.getTimeZone().toZoneId());
```
### Из современного в устаревший

```java
LocalDateTime ldt = LocalDateTime.now();
Date newDate = Date.from(ldt.atZone(ZoneId.systemDefault()).toInstant());
```
### Примечание

Всегда указывайте `ZoneId` явно при конвертации, чтобы избежать неоднозначности с системным часовым поясом по умолчанию.
## Реальный сценарий: сравнение времени в разных часовых поясах

Этот пример показывает, как сравнить один и тот же момент времени в разных часовых поясах и вычислить разницу.

```java
ZonedDateTime parisTime = ZonedDateTime.now(ZoneId.of("Europe/Paris"));
ZonedDateTime newYorkTime = parisTime.withZoneSameInstant(ZoneId.of("America/New_York"));

System.out.println("Париж: " + parisTime); // Например, 2025-07-02T19:31:00+02:00[Europe/Paris]
System.out.println("Нью-Йорк: " + newYorkTime); // Например, 2025-07-02T13:31:00-04:00[America/New_York]

Duration timeDifference = Duration.between(newYorkTime, parisTime);
System.out.println("Разница во времени: " + timeDifference.toHours() + " часов");
```
## Лучшие практики и распространённые ошибки
### Лучшие практики

- Используйте `java.time` для нового кода: Избегайте `Date` и `Calendar`, если не требуется взаимодействие с устаревшими системами.
- Указывайте часовые пояса явно: Используйте `ZoneId.of("Europe/Paris")` вместо `ZoneId.systemDefault()`, чтобы избежать неожиданного поведения.
- Используйте неизменяемые объекты: Воспользуйтесь неизменяемостью классов `java.time` для предотвращения случайных изменений.
- Применяйте предопределённые форматтеры: Используйте константы `DateTimeFormatter.ISO_*` для стандартных форматов, чтобы обеспечить согласованность.
- Обрабатывайте исключения: Всегда обрабатывайте `DateTimeParseException` при парсинге строк, чтобы избежать ошибок во время выполнения.
### Распространённые ошибки

- Предположение о системном часовом поясе: Использование `ZoneId.systemDefault()` может привести к непредсказуемому поведению в разных окружениях.
- Игнорирование перехода на летнее время: Учитывайте переход на летнее время при работе с `ZonedDateTime`.
- Неправильное использование `Instant` и `LocalDateTime`: Используйте `Instant` для машинно-читаемых временных меток, а `LocalDateTime` для локального времени, понятного людям.
- Ошибки в шаблонах парсинга: Убедитесь, что шаблон `DateTimeFormatter` точно соответствует входной строке, чтобы избежать ошибок парсинга.
## Дополнительные ресурсы

- [Официальная документация Java](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/package-summary.html)
- [Joda-Time](http://www.joda.org/joda-time/) (для понимания источника вдохновения `java.time`)
- [Руководство по датам и времени в Java](https://www.baeldung.com/java-8-date-time)