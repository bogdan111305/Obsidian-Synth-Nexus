# Аудит вики по Senior Java Roadmap

> Сопоставление текущего содержимого этого vault'а (`Base/`, 219+ заметок) с [`LEARNING_ROADMAP.md`](file:///C:/Users/brubc/OpenideProjects/digital-bunker-yandex-storage/LEARNING_ROADMAP.md) из репозитория `digital-bunker-yandex-storage` — учебным планом Junior → Senior Java Developer, привязанным к фазам проекта «Цифровой Бункер» (см. `CLAUDE.md` там же).
>
> Ссылки на `LEARNING_ROADMAP.md §X.Y` и `CLAUDE.md §X` ниже указывают на разделы этих двух файлов в репозитории `digital-bunker-yandex-storage` (обычные пути, не wikilink — файлы вне этого vault'а).

## Как читать этот документ

- ✅ **Закрыто** — тема раскрыта на нужном уровне, действий не требуется.
- ⚠️ **Частично** — тема есть, но не в том разрезе/глубине, что нужен для проекта.
- ❌ **Отсутствует** — заметок нет вообще, тема слепая зона.
- Для каждого пробела указано: какой раздел `LEARNING_ROADMAP.md` он закрывает, какая фаза `CLAUDE.md` (проект «Цифровой Бункер») от него зависит, и куда в `Base/` эту заметку логично положить (по аналогии с уже принятой структурой нумерованных подпапок).

---

## 1. Сводная таблица

| Домен | Статус | Roadmap | Блокирует фазу |
|---|---|---|---|
| Java Core / OOP / Generics / Exceptions | ✅ | §1.1 | — |
| Collections & Streams | ✅ | §1.1, §2.1 | — |
| Concurrency & JMM (включая Virtual Threads) | ✅ | §2.2, §3.1, §3.2 | — |
| JVM Internals / GC / Modern Platform (Panama, Valhalla) | ✅ | §3.1, §4.2 | — |
| Hibernate / JPA (все аспекты, вплоть до Envers и 2nd level cache) | ✅ | §2.5 | — |
| Spring Core / Spring Data | ✅ | §2.3, §2.4, §2.5 | — |
| JDBC | ✅ | §1.3 | — |
| Git | ✅ | §1.2 | — |
| Миграции (Flyway/Liquibase) | ✅ | §2.6 | — |
| Kafka | ✅ (вне скоупа MVP, но пригодится) | — | — |
| Observability (Grafana/Prometheus/OTel/Jaeger) | ⚠️ | §4.4 | Phase 4 |
| SQL/Postgres: индексы, `EXPLAIN ANALYZE`, isolation levels отдельной темой | ⚠️ | §1.3, §2.6 | Phase 0–2 |
| **Spring Security** | ⚠️ (пустой skeleton) | §3.3, §3.7 | **Phase 1** |
| **Криптография для разработчиков** | ❌ | §3.6 | **Phase 2** |
| **gRPC & Protocol Buffers** | ❌ | §3.4 | **Phase 1–2** |
| **Redis** | ❌ | §3.5 | **Phase 2** |
| **Docker & Docker Compose** | ❌ | §2.8 | **Phase 0** |
| REST API Design (DTO, валидация, обработка ошибок, OpenAPI) | ⚠️ | §2.7 | Phase 1–2 |
| Тестирование как отдельная база (JUnit5, Mockito, Testcontainers) | ⚠️ | §1.5, §2.9 | Phase 0–2 |
| API Gateway Pattern | ❌ | §3.8 | Phase 1 |
| Отказоустойчивость (Resilience4j, retry, Saga) | ❌ | §4.3 | Phase 2–3 |
| CI/CD | ❌ | §4.5 | Phase 4 |
| Продвинутая безопасность (Zero-Trust, mTLS, threat modeling, key management) | ❌ | §4.6 | Phase 4–5 |
| System Design для Senior (ADR, CAP, backup/DR) | ❌ | §4.7 | Phase 3–4 |
| Cross-device аутентификация / Device Trust | ❌ | §3.11 | Phase 4 |
| Проектная заметка «Цифровой Бункер» — устарела | ⚠️ | — | — |

---

## 2. Что уже закрыто сильно (трогать не нужно)

`Base/Java/`, `Base/Hibernate/`, `Base/Spring/1. Основы Spring/` и `4. Spring Data/`, `Base/JDBC/`, `Base/Git/`, `Base/Migration/`, `Base/Kafka/` — это уже база уровня выше, чем требует `LEARNING_ROADMAP.md` для Middle/Middle+ (например, GC-алгоритмы, Project Panama/Valhalla, Hibernate Envers — это глубже, чем формально нужно по плану, и это хорошо, не нужно урезать). Единственное, что стоит сделать — **связать** эти уже написанные заметки с планом обратными ссылками (см. §6).

---

## 3. Критические пробелы (блокируют старт проекта, Phase 0–2)

### 3.1 Docker & Docker Compose — ❌ отсутствует полностью
Ни одной заметки с ключевыми словами docker/compose/контейнер во всём vault'е. Без этого нельзя пройти даже Phase 0 (`CLAUDE.md §14` — вся инфраструктура на Docker Compose).

- **Roadmap:** §2.8 (Middle)
- **Что писать:** `Dockerfile` для Java (multi-stage build, `eclipse-temurin` base), `docker-compose.yml` (сервисы/сети/volumes), bind mount vs named volume, healthcheck, `.env` в compose.
- **Куда класть:** новая папка `Base/Docker/` — `Основы.md`, `Dockerfile для Java.md`, `Docker Compose — сети и volumes.md`.

### 3.2 Spring Security — ⚠️ пустой skeleton
`Base/Spring/3. Spring Security/README.md` — это только оглавление, перечисляющее файлы (`Spring Security.md`, `Безопасность REST API.md`, `JWT и сессии.md`, `SpringOauth2 - Filters.md`), которых **физически не существует**. По факту раздел пуст. Это прямой блокер `bunker-auth` (Phase 1).

- **Roadmap:** §3.3 (Spring Security), §3.7 (Web Security, WebAuthn/FIDO2)
- **Что писать:** filter chain и как запрос через неё проходит, Authentication vs Authorization, `SecurityContext`, JWT (структура header.payload.signature, `HS256` vs `ES256`/`EdDSA` — см. `CLAUDE.md §7.2`), method security (`@PreAuthorize`), и **отдельно** — WebAuthn/FIDO2 (CTAP2, `signCount`, клон-детекция — см. `CLAUDE.md §6.3`), которого в оглавлении раздела вообще нет (там только OAuth2/OIDC — из другого проекта, «Корпоративная соцсеть»).
- **Куда класть:** реально создать файлы, которые уже заявлены в README (`Spring Security.md`, `JWT и сессии.md`), плюс новый `WebAuthn и FIDO2.md`, `Device Trust Tiers и Step-up Auth.md` (§3.11 роадмапа, `CLAUDE.md §7.4`).

### 3.3 Криптография для разработчиков — ❌ отсутствует полностью
Ни одной заметки про AES/Argon2/Tink/HMAC/Envelope Encryption. Это ядро всего проекта «Цифровой Бункер» (`CLAUDE.md §6`) и отдельный уровень в `LEARNING_ROADMAP.md §3.6` — самый весомый пробел относительно того, зачем вообще затевался этот pet-проект.

- **Roadmap:** §3.6 (Middle+, ядро проекта)
- **Что писать:** симметричное/асимметричное шифрование, AES-GCM и почему AEAD, nonce/IV и запрет на переиспользование, KDF (PBKDF2 vs Argon2id — почему memory-hard), HMAC, Envelope Encryption (MK→CK→DEK — прямая связь с `CLAUDE.md §6.1`), Google Tink и почему не голый JCE, `StreamingAead`, типичные ошибки (ECB, статичный IV, логирование секретов, `==` вместо constant-time compare).
- **Куда класть:** новая папка `Base/Криптография/` — `AES-GCM и AEAD.md`, `KDF — PBKDF2 vs Argon2id.md`, `Envelope Encryption.md`, `Google Tink.md`, `Типичные ошибки в крипто-коде.md`.

### 3.4 gRPC & Protocol Buffers — ❌ отсутствует полностью
Весь межсервисный контракт проекта (`bunker-auth` ↔ `bunker-storage` ↔ `bunker-gateway`) — на gRPC (`CLAUDE.md §10`). Заметок нет.

- **Roadmap:** §3.4 (Middle+)
- **Что писать:** `.proto`-синтаксис (message/service), генерация кода, unary vs client/server/bidi streaming (в проекте `UploadFile`/`DownloadFile` — streaming), обработка ошибок (`Status`/`StatusRuntimeException`), interceptors (mTLS, логирование, метрики).
- **Куда класть:** новая папка `Base/gRPC/` — `Protocol Buffers — основы.md`, `Streaming RPC.md`, `Interceptors и обработка ошибок.md`.

### 3.5 Redis — ❌ отсутствует полностью
Redis держит на себе Category Key TTL-кэш и Kill Switch (`CLAUDE.md §12–13`) — критичный компонент Phase 2.

- **Roadmap:** §3.5 (Middle+)
- **Что писать:** структуры данных, TTL, eviction policies, паттерн cache-aside, Lua-скрипты для атомарных операций (и почему атомарность здесь критична — race condition между проверкой и удалением ключей, как в Kill Switch).
- **Куда класть:** новая папка `Base/Redis/` — `Основы и структуры данных.md`, `TTL и eviction.md`, `Lua-скрипты и атомарность.md`.

---

## 4. Важные пробелы (нужны скоро, Phase 1–4)

### 4.1 REST API Design — ⚠️ частично
Bean Validation (JSR 303) есть, но только в контексте `Hibernate/9. DAO & Repository`, не как самостоятельная тема REST-слоя. DTO vs Entity, единообразная обработка ошибок (`@ControllerAdvice`), OpenAPI/Swagger — не встречаются нигде.

- **Roadmap:** §2.7 (Middle)
- **Куда класть:** `Base/Spring/5. Spring Web/` — добавить `DTO vs Entity.md`, `Обработка ошибок — @ControllerAdvice.md`, `OpenAPI и Swagger.md`.

### 4.2 Тестирование как отдельная база — ⚠️ частично
Есть только `Spring/4. Spring Data/3. Data JPA Transactions/3. Тестирование/` — узко про тестирование транзакций (`@Commit`/`@Rollback`, `TransactionalTestExecutionListener`). Нет базового JUnit5 (`@Test`, assertions, параметризация), Mockito (`@Mock`/`when`/`verify`), Testcontainers как общей практики (не только для транзакций).

- **Roadmap:** §1.5 (Junior), §2.9 (Middle)
- **Куда класть:** новая папка `Base/Тестирование/` — `JUnit 5 — основы.md`, `Mockito.md`, `Testcontainers.md` (с явной пометкой, почему H2 — плохая замена реальному PostgreSQL в тестах, роадмап уже фиксирует эту тонкость).

### 4.3 API Gateway Pattern — ❌ отсутствует
`bunker-gateway` (Spring Cloud Gateway) — единая точка входа проекта (`CLAUDE.md §2–3`).

- **Roadmap:** §3.8 (Middle+)
- **Куда класть:** `Base/Spring/` — новая подпапка `6. Spring Cloud Gateway/` — `Routing и фильтры.md`, `Rate Limiting.md`.

### 4.4 Отказоустойчивость (Resilience4j) — ❌ отсутствует
`CLAUDE.md §5` требует circuit breaker вокруг вызовов к Яндекс Диску.

- **Roadmap:** §4.3 (Senior)
- **Куда класть:** новая папка `Base/Отказоустойчивость/` — `Circuit Breaker — Resilience4j.md`, `Retry и Backoff.md`, `Идемпотентность операций.md`.

### 4.5 CI/CD — ❌ отсутствует
- **Roadmap:** §4.5 (Senior)
- **Куда класть:** новая папка `Base/CI-CD/` — `GitHub Actions — pipeline.md`, `SAST и сканирование зависимостей.md`.

### 4.6 Продвинутая безопасность — ❌ отсутствует
Zero-Trust, mTLS, key management, threat modeling (STRIDE), secure deletion — всё, что превращает Middle-безопасность в Senior (`LEARNING_ROADMAP.md §4.6`, `CLAUDE.md P1/P5`).

- **Roadmap:** §4.6 (Senior)
- **Куда класть:** `Base/Криптография/` (из §3.3 выше) — добавить `mTLS.md`, `Threat Modeling — STRIDE.md`, `Key Management и ротация.md`, `Secure Deletion.md`.

### 4.7 System Design для Senior — ❌ отсутствует
ADR, CAP-теорема применительно к решениям, RPO/RTO для бэкапов.

- **Roadmap:** §4.7 (Senior)
- **Куда класть:** новая папка `Base/System Design/` — `ADR — шаблон и примеры.md` (со ссылкой на реальные решения `CLAUDE.md`), `CAP-теорема на практике.md`, `Backup & Disaster Recovery — RPO-RTO.md`.

### 4.8 Cross-device аутентификация / Device Trust — ❌ отсутствует
Это новая тема, добавленная в `LEARNING_ROADMAP.md §3.11` после проработки десктоп-клиента для «Цифрового Бункера» (`CLAUDE.md §7.4`): OAuth 2.0 Device Authorization Grant (RFC 8628), WebAuthn authenticator transports (platform vs roaming), number matching как защита от relay-атак, risk-based/step-up authentication.

- **Roadmap:** §3.11 (Middle+)
- **Куда класть:** `Base/Spring/3. Spring Security/` — `Device Trust Tiers и QR Companion Pairing.md`, `Step-up Authentication.md`.

---

## 5. Точечные улучшения существующего

### 5.1 Observability — ⚠️ хороший Middle-уровень, не хватает Senior-глубины
`Grafana, Prometheus, OpenTelemetry и Jaeger/Основы.md` — качественная база (логи/метрики/трейсы, установка, интеграция со Spring Boot Actuator). Не хватает того, что именно различает Middle и Senior здесь:
- Micrometer custom metrics (не только встроенные `http_server_requests`).
- Audit log как продуктовое, а не техническое требование — разница между debug-логом и юридически значимым аудитом (см. `LEARNING_ROADMAP.md §4.4`, `CLAUDE.md audit_log`).
- Проектирование алертов (не просто "AlertManager существует", а какие пороги и почему).

**Куда класть:** дополнить существующую заметку или добавить `Grafana, Prometheus, OpenTelemetry и Jaeger/Micrometer и Audit Log.md`.

### 5.2 SQL/Postgres — индексы и EXPLAIN отдельной темой
Изоляция транзакций частично упомянута в `Hibernate/5. Transactions & Locks/Введение в транзакции и блокировки в базах данных.md`, но чисто SQL-темы — B-tree индексы, когда индекс не используется, `EXPLAIN ANALYZE` — нигде не встречаются как отдельная заметка (сейчас всё через призму Hibernate/JPA, а не голого SQL).

**Куда класть:** новая папка `Base/PostgreSQL/` — `Индексы — B-tree и когда не используются.md`, `EXPLAIN ANALYZE — чтение плана запроса.md`.

---

## 6. Устаревший проектный артефакт

`Base/Проекты/Цифровой Бункер/Экосистема «Цифровой Бункер».md` **побайтово идентичен** файлу `Экосистема «Цифровой Бункер».md` в `digital-bunker-yandex-storage` — это черновик **полной** (не MVP) архитектуры: с чатом, X3DH/Double Ratchet, Kafka, ClickHouse, топологией Sentinel/Vault на WireGuard и обязательным NFC-кольцом/Titan M2. Действующий `CLAUDE.md` в том же репозитории — это переработанный **MVP**-вариант (Vault Secrets + Encrypted Files поверх Яндекс Диска, single-node, WebAuthn вместо обязательного NFC-железа, плюс Device Trust Tiers для десктопа), и эта заметка в vault'е за ним не поспевает.

**Рекомендация:** заменить содержимое заметки на актуальный `CLAUDE.md` (или хотя бы добавить в начало пометку `> ⚠️ Устарело — актуальная архитектура: digital-bunker-yandex-storage/CLAUDE.md`), чтобы vault не противоречил рабочей версии.

---

## 7. Порядок закрытия пробелов

Привязано к тому, что реально блокирует `CLAUDE.md`-фазы проекта, а не к произвольному приоритету:

1. **Docker & Docker Compose** (§3.1) — без этого не поднять даже Phase 0.
2. **Spring Security + WebAuthn** (§3.2) — блокирует `bunker-auth`, Phase 1.
3. **gRPC** (§3.4) — контракт между сервисами нужен уже в Phase 1, интенсивно — в Phase 2.
4. **Криптография** (§3.3) и **Redis** (§3.5) — ядро Phase 2 (Envelope Encryption, CK-кэш).
5. **REST API Design** и **Тестирование** (§4.1–4.2) — подтягиваются параллельно с Phase 0–2, не блокируют, но откладывать надолго не стоит.
6. **Отказоустойчивость** (§4.4) — нужна к Phase 2–3 (circuit breaker вокруг Яндекс Диска).
7. **API Gateway** (§4.3) — нужен там же, где `bunker-gateway` реально появляется в коде (Phase 1).
8. **Observability-глубина, Продвинутая безопасность, CI/CD, System Design, Device Trust** (§4.5–4.8, §5.1) — Phase 3–5, можно закрывать по мере приближения к этим фазам.
9. **Устаревшая проектная заметка** (§6) — низкий приоритет по срочности, но легко сделать в любой момент — простая синхронизация файла.
