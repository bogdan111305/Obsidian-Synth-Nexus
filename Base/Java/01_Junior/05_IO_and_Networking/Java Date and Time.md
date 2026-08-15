---
title: "Java Date and Time — java.time API"
tags: [java, date, time, localdate, zoneddatetime, datetimeformatter, java8]
updated: 2026-08-15
---

# Java Date and Time (java.time API)

> [!QUOTE] Суть
> Современный API (`java.time`, Java 8+): **иммутабельный**, thread-safe. `LocalDate` (дата без времени), `LocalDateTime` (без TZ), `ZonedDateTime` (с TZ), `Instant` (Unix timestamp), `Duration`/`Period` (интервалы). Старые `Date`/`Calendar` — **устарели**, мутабельны, не thread-safe.

> [!WARNING] Ловушка: SimpleDateFormat не thread-safe
> `SimpleDateFormat` — НЕ thread-safe. Sharing одного инстанса между потоками → race condition → некорректные даты. Используй `DateTimeFormatter` (java.time, иммутабельный и thread-safe) или создавай `SimpleDateFormat` в каждом потоке отдельно.

## Устаревший API (до Java 8) — не используй в новом коде

- **`java.util.Date`** — момент времени; большинство методов (`getYear()`, `getMonth()`) deprecated и багоопасны (месяцы нумеруются с нуля).
- **`java.util.Calendar`** — гибче `Date`, но громоздкий, неинтуитивный API.
- **`java.text.SimpleDateFormat`** — форматирование/парсинг, **не потокобезопасен**.
- **Ограничения в целом**: мутабельность (гонки в многопоточном коде), непоследовательная работа с часовыми поясами, неудобные операции (например, добавление дней).

```java
Date date = new Date(); // текущая дата и время

Calendar calendar = Calendar.getInstance();
calendar.set(2025, Calendar.JULY, 2); // месяцы начинаются с 0!
Date oldDate = calendar.getTime();
```

## Современный API (Java 8+): `java.time`

Пакет `java.time` (вдохновлён Joda-Time) — неизменяемый, потокобезопасный, соответствует ISO-8601. Рекомендуемый подход для всех новых операций с датами и временем.

| Класс | Описание | Пример |
|-------|----------|--------|
| `LocalDate` | Дата без времени | `2024-01-15` |
| `LocalTime` | Время без даты | `14:30:00` |
| `LocalDateTime` | Дата + время, без TZ | `2024-01-15T14:30` |
| `ZonedDateTime` | Дата + время + TZ | `2024-01-15T14:30+03:00` |
| `Instant` | Unix timestamp | `1705320600` |
| `Duration` | Интервал в секундах/минутах/часах | `PT1H30M` |
| `Period` | Интервал в днях/месяцах/годах | `P1Y2M` |
| `DateTimeFormatter` | Форматирование и парсинг | `dd.MM.yyyy` |

Преимущества перед старым API: неизменяемость (потокобезопасность "бесплатно"), интуитивные методы (`plusDays()`, `isBefore()`...), надёжная работа с часовыми поясами через `ZoneId`/`ZoneOffset`.

## `LocalDate` / `LocalTime` / `LocalDateTime` — создание, формат, парсинг, арифметика

```java
// Создание:
LocalDate date = LocalDate.now();                                   // 2025-07-02
LocalDateTime dateTime = LocalDateTime.of(2025, 7, 2, 19, 31);       // 2025-07-02T19:31
LocalDate specificDate = LocalDate.of(2025, Month.DECEMBER, 25);     // 2025-12-25

// Арифметика — все методы возвращают НОВЫЙ объект (иммутабельность):
LocalDate tomorrow = date.plusDays(1);
LocalDate nextMonth = date.plusMonths(1);
LocalDateTime inOneHour = dateTime.plusHours(1);

// Форматирование и парсинг — один и тот же DateTimeFormatter в обе стороны:
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd.MM.yyyy");
String formatted = date.format(formatter);              // 02.07.2025
LocalDate parsed = LocalDate.parse("02.07.2025", formatter); // 2025-07-02
```

## `ZonedDateTime` и часовые пояса

`ZonedDateTime` объединяет `LocalDateTime` + `ZoneId` + `ZoneOffset` — используй его (или `Instant`), когда часовой пояс важен: расписания, логи с разных серверов, встречи между пользователями в разных TZ.

```java
// Создание — в конкретной зоне или из LocalDateTime:
ZonedDateTime tokyo = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));
ZonedDateTime paris = LocalDateTime.of(2025, 7, 2, 19, 31).atZone(ZoneId.of("Europe/Paris"));

// Конвертация в другую зону — тот же момент времени, другое представление:
ZonedDateTime newYork = paris.withZoneSameInstant(ZoneId.of("America/New_York"));
System.out.println(paris);   // 2025-07-02T19:31+02:00[Europe/Paris]
System.out.println(newYork); // 2025-07-02T13:31-04:00[America/New_York]

// Разница между двумя ZonedDateTime — Duration учитывает фактическую разницу в offset'ах:
Duration diff = Duration.between(newYork, paris);
System.out.println(diff.toHours()); // 0 — это один и тот же момент времени, просто разное представление

// Извлечение компонентов и переход в Instant (UTC):
Instant instant = paris.toInstant();
LocalDate localDate = paris.toLocalDate();
ZoneOffset offset = paris.getOffset(); // +02:00
```

## `Instant` — точка на временной шкале

`Instant` — момент времени в наносекундах от эпохи Unix (`1970-01-01T00:00:00Z`), без привязки к часовому поясу. Используй для машинно-читаемых timestamp'ов (логи, метрики, хранение в БД); `LocalDateTime`/`ZonedDateTime` — для представления, понятного человеку.

```java
Instant now = Instant.now();
long epochMillis = now.toEpochMilli();

Instant fromMillis = Instant.ofEpochMilli(1625212800000L);
ZonedDateTime zoned = now.atZone(ZoneId.of("Europe/Paris")); // привязка к зоне при необходимости
```

## `Duration` и `Period` — интервалы времени

- **`Duration`** — интервал в секундах/наносекундах, для измерения времени между двумя `Instant`/`LocalTime`/`LocalDateTime`.
- **`Period`** — интервал в годах/месяцах/днях, для разницы между `LocalDate`.

```java
// Duration — время выполнения операции:
Instant start = Instant.now();
Thread.sleep(1000);
Duration elapsed = Duration.between(start, Instant.now());
System.out.println(elapsed.toMillis()); // ~1000

Duration fiveMinutes = Duration.ofMinutes(5).plus(Duration.ofSeconds(30));

// Period — разница между календарными датами:
Period period = Period.between(LocalDate.of(2023, 1, 1), LocalDate.now());
System.out.println(period.getYears() + " лет, " + period.getMonths() + " месяцев, " + period.getDays() + " дней");
```

## `DateTimeFormatter` — форматирование и парсинг

Иммутабельный и потокобезопасный (в отличие от `SimpleDateFormat`, см. предупреждение выше) — безопасно хранить как `static final` и переиспользовать между потоками.

```java
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm:ss z");

ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Europe/Paris"));
String formatted = zdt.format(fmt);              // 02.07.2025 19:31:00 CEST
ZonedDateTime parsed = ZonedDateTime.parse(formatted, fmt);

// Предопределённые форматтеры для стандартных случаев:
DateTimeFormatter.ISO_LOCAL_DATE_TIME; // 2025-07-02T19:31:00
DateTimeFormatter.RFC_1123_DATE_TIME;  // Wed, 2 Jul 2025 19:31:00 +0200
DateTimeFormatter.BASIC_ISO_DATE;      // 20250702
```

## Конвертация между устаревшим и современным API

Через `Instant` — единственную точку соприкосновения обоих API. Всегда указывай `ZoneId` явно, иначе конвертация неявно зависит от системного часового пояса.

```java
// Date/Calendar → java.time:
LocalDateTime ldt = LocalDateTime.ofInstant(new Date().toInstant(), ZoneId.systemDefault());
Calendar calendar = Calendar.getInstance();
LocalDateTime ldtFromCal = LocalDateTime.ofInstant(calendar.toInstant(), calendar.getTimeZone().toZoneId());

// java.time → Date:
Date newDate = Date.from(LocalDateTime.now().atZone(ZoneId.systemDefault()).toInstant());
```

## Лучшие практики и распространённые ошибки

- **Используй `java.time` для нового кода** — `Date`/`Calendar` только при интеграции с legacy-системами.
- **Указывай `ZoneId` явно** (`ZoneId.of("Europe/Paris")`) вместо `ZoneId.systemDefault()` — иначе поведение зависит от окружения выполнения.
- **`Instant` для машин, `LocalDateTime`/`ZonedDateTime` для людей** — не путай назначение классов.
- **Не игнорируй переход на летнее время** при арифметике над `ZonedDateTime` — прибавление часов может привести к неочевидному сдвигу даты.
- **Обрабатывай `DateTimeParseException`** при парсинге строк, шаблон `DateTimeFormatter` должен точно соответствовать входным данным.

## Дополнительные ресурсы

- [Официальная документация Java](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/package-summary.html)
- [Joda-Time](http://www.joda.org/joda-time/) (источник вдохновения `java.time`)
- [Руководство по датам и времени в Java](https://www.baeldung.com/java-8-date-time)
