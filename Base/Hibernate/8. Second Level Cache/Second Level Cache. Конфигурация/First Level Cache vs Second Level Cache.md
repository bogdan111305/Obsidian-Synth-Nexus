# First Level Cache vs Second Level Cache

## Оглавление
1. [First Level Cache (L1)](#l1-cache)
2. [Second Level Cache (L2)](#l2-cache)
3. [Что кэшируется: entity state](#что-кэшируется)
4. [Инвалидация L2 Cache](#инвалидация)
5. [Ловушка: HQL/SQL минует L2](#ловушка)
6. [Когда использовать L2 Cache](#когда-использовать)

---

## 1. First Level Cache (L1) <a name="l1-cache"></a>

**First Level Cache** = Persistence Context = Identity Map.

- **Scope**: один `Session` / `EntityManager`
- **Включён всегда**: нельзя отключить
- **Не настраивается**: нет TTL, нет размера
- **Очищается**: при `close()` сессии, при `clear()`, при `evict(entity)`

```java
// L1 Cache в действии:
User u1 = entityManager.find(User.class, 1L); // SELECT * FROM users WHERE id=1
User u2 = entityManager.find(User.class, 1L); // Нет SQL! Берётся из L1 Cache
System.out.println(u1 == u2); // true — один и тот же объект
```

**Что хранит L1 Cache**:
- Объекты entity по их ID
- Snapshot состояния для dirty checking
- Ссылки на инициализированные коллекции

**Ограничения L1 Cache**:
- Не разделяется между сессиями
- Не разделяется между потоками
- Не переживает завершение транзакции (при transaction-scoped EM)
- Растёт неограниченно в рамках сессии → используй `clear()` при bulk операциях

---

## 2. Second Level Cache (L2) <a name="l2-cache"></a>

**Second Level Cache** — разделяемый кэш уровня `SessionFactory`.

- **Scope**: весь `SessionFactory` = всё приложение
- **Включён опционально**: требует явной конфигурации
- **Разделяется**: между сессиями, между потоками (но не между нодами без distributed cache)
- **Настраивается**: TTL, max size, eviction policy — через Ehcache, Caffeine, Infinispan и т.д.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
```

**Цепочка поиска при `find(User.class, 1L)`**:
```
1. Проверить L1 Cache (текущая сессия)
   → Найдено? Вернуть объект из PC
2. Проверить L2 Cache (SessionFactory)
   → Найдено? Восстановить entity из cached state, положить в L1
3. Выполнить SELECT в БД
   → Положить в L2 Cache, положить в L1 Cache, вернуть entity
```

---

## 3. Что кэшируется: entity state <a name="что-кэшируется"></a>

> [!INFO]
> L2 Cache хранит **не объект entity**, а **дезассемблированное состояние** (массив значений полей). При чтении из кэша Hibernate собирает entity заново.

```
Кэш хранит: { id=1, name="Alice", email="alice@example.com", version=3 }
При чтении: Hibernate создаёт new User() и заполняет поля из кэша
```

**Почему так**: объект entity может быть mutable, содержать ссылки на другие объекты, Proxy и т.д. Хранить raw state безопаснее и экономичнее.

**Что кэшируется отдельно**:
- Entity state (по умолчанию) — для каждого ID свой ключ
- Коллекции — отдельный кэш для каждой коллекции (`@Cache` на коллекции)
- Query results — Query Cache (отдельная конфигурация)

**Что НЕ кэшируется**:
- Lazy-ассоциации (только ID, не объект)
- Транзитивные сущности (только если они тоже помечены `@Cache`)

---

## 4. Инвалидация L2 Cache <a name="инвалидация"></a>

### Автоматическая инвалидация

Hibernate автоматически инвалидирует (удаляет) запись из L2 Cache при изменении entity через **Hibernate API**:

```java
User user = entityManager.find(User.class, 1L); // L2 Cache: HIT
user.setEmail("new@example.com");
// commit → Hibernate инвалидирует кэш для User#1
entityManager.find(User.class, 1L); // L2 Cache: MISS → SELECT
```

### Ручная инвалидация

```java
// Инвалидировать конкретную entity
sessionFactory.getCache().evict(User.class, 1L);

// Инвалидировать все entity класса
sessionFactory.getCache().evict(User.class);

// Инвалидировать весь L2 Cache
sessionFactory.getCache().evictAllRegions();
```

---

## 5. Ловушка: HQL/SQL минует L2 <a name="ловушка"></a>

> [!WARNING]
> **UPDATE/DELETE через HQL или нативный SQL НЕ инвалидирует L2 Cache автоматически!**

```java
// Обновляем через HQL bulk update
entityManager.createQuery(
    "UPDATE User u SET u.email = :email WHERE u.id = :id")
    .setParameter("email", "new@example.com")
    .setParameter("id", 1L)
    .executeUpdate();

// L2 Cache всё ещё содержит СТАРОЕ состояние User#1!
User user = entityManager.find(User.class, 1L);
// Если другая сессия запросит User#1 — она получит УСТАРЕВШИЕ данные из кэша
```

**Почему так**: bulk HQL/SQL обходит Persistence Context и работает напрямую с БД. Hibernate не знает, какие конкретно записи были изменены (особенно если в WHERE сложное условие).

**Решения**:

```java
// 1. Инвалидировать вручную после bulk update
entityManager.createQuery("UPDATE User u SET u.active = false WHERE ...")
    .executeUpdate();
sessionFactory.getCache().evict(User.class); // инвалидируем весь кэш User

// 2. Не использовать L2 Cache для entity, которые часто обновляются bulk-операциями
```

---

## 6. Когда использовать L2 Cache <a name="когда-использовать"></a>

**L2 Cache эффективен для**:
- Справочники (страны, категории, роли) — меняются редко, читаются часто
- Конфигурационные entity — меняются только через UI администратора
- Aggregate roots с редкими обновлениями

**L2 Cache НЕ эффективен для**:
- Entity, которые часто обновляются (заказы, транзакции)
- Entity с bulk HQL/SQL обновлениями — кэш будет инвалидирован постоянно
- Entity, уникальных для каждого пользователя (мало повторных чтений)

**Практическое правило**:
```
read:write ratio > 10:1  → L2 Cache даёт выигрыш
read:write ratio < 10:1  → L2 Cache создаёт overhead без пользы
```

---

**Связанные файлы:**
- [[Аннотация @Cache]] — настройка стратегии кэширования
- [[Cache Concurrency Strategy]] — регионы и стратегии
- [[Schema Generation и Flyway]] — конфигурация приложения
- [[Dirty Checking и FlushMode]] — как flush взаимодействует с кэшем
