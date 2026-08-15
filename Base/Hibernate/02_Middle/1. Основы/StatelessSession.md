# StatelessSession

## Оглавление
1. [Что такое StatelessSession](#что-это)
2. [Отличия от обычной Session](#отличия)
3. [Когда использовать](#когда)
4. [Ограничения](#ограничения)
5. [Паттерн: bulk insert через StatelessSession](#bulk-insert)

---

## 1. Что такое StatelessSession <a name="что-это"></a>

`StatelessSession` — низкоуровневый интерфейс Hibernate для работы с БД **без Persistence Context**. Нет dirty checking, нет кэша первого уровня, нет каскадирования, нет событий.

Каждая операция (`insert`, `update`, `delete`, `get`) немедленно транслируется в SQL и выполняется.

```java
StatelessSession statelessSession = sessionFactory.openStatelessSession();
```

> [!INFO]
> `StatelessSession` ближе к JDBC, чем к JPA. Это инструмент для **производительных batch-операций**, а не для обычной работы с сущностями.

---

## 2. Отличия от обычной Session <a name="отличия"></a>

| Характеристика | Session (StatefulSession) | StatelessSession |
|----------------|--------------------------|------------------|
| Persistence Context | Есть | Нет |
| Dirty Checking | Есть | Нет |
| L1 Cache | Есть | Нет |
| L2 Cache | Читает и пишет | Не использует |
| Каскадирование | Есть | Нет |
| Lifecycle events | @PrePersist, @PostLoad... | Нет |
| Lazy Loading | Работает | **Нет** (LazyInitializationException) |
| Identity Map | Есть | Нет (два `get()` → два SELECT) |
| Версионность (@Version) | Автоматически | Нужно управлять вручную |

```java
// Session: загружает один объект для обоих вызовов (Identity Map)
User u1 = session.get(User.class, 1L);
User u2 = session.get(User.class, 1L);
System.out.println(u1 == u2); // true

// StatelessSession: каждый get() → отдельный SELECT
User u1 = statelessSession.get(User.class, 1L);
User u2 = statelessSession.get(User.class, 1L);
System.out.println(u1 == u2); // false — разные объекты!
```

---

## 3. Когда использовать <a name="когда"></a>

**StatelessSession подходит для:**
- Bulk insert/update/delete больших объёмов данных (миллионы строк)
- ETL-процессы: читаем из одного источника, пишем в другой
- Пакетная обработка, где не нужна объектная идентичность и связи
- Миграция данных

**Не подходит для:**
- Обычной бизнес-логики с lazy-ассоциациями
- Работы со сложными графами объектов
- Сценариев, где нужны lifecycle callbacks

---

## 4. Ограничения <a name="ограничения"></a>

> [!WARNING]
> **Нет Lazy Loading**: обращение к неинициализированной lazy-ассоциации вызовет `LazyInitializationException`. Загружайте только то, что реально нужно.

> [!WARNING]
> **Нет каскадирования**: `CascadeType.PERSIST`, `CascadeType.REMOVE` и т.д. — не работают. Нужно вручную управлять связанными сущностями.

> [!WARNING]
> **Нет L2 Cache**: StatelessSession полностью игнорирует Second Level Cache. Записанные данные не попадут в кэш. После bulk-операций через StatelessSession L2 Cache может содержать устаревшие данные.

```java
// После bulk insert через StatelessSession
// L2 Cache НЕ знает о новых записях
// Если другая сессия запросит эти записи — они будут загружены из БД
// Но если запрашивается entity по ID, которого ещё нет в кэше — всё ок
```

---

## 5. Паттерн: bulk insert через StatelessSession <a name="bulk-insert"></a>

### Правильный bulk insert

```java
@Service
public class BulkImportService {
    
    @Autowired
    private SessionFactory sessionFactory;
    
    public void bulkInsertUsers(List<UserDto> dtos) {
        // Открываем StatelessSession отдельно от текущего EntityManager
        try (StatelessSession session = sessionFactory.openStatelessSession()) {
            Transaction tx = session.beginTransaction();
            
            try {
                int batchSize = 50; // должен совпадать с hibernate.jdbc.batch_size
                
                for (int i = 0; i < dtos.size(); i++) {
                    UserDto dto = dtos.get(i);
                    
                    User user = new User();
                    user.setEmail(dto.getEmail());
                    user.setFirstName(dto.getFirstName());
                    user.setActive(true);
                    
                    session.insert(user); // немедленно → JDBC batch buffer
                    
                    // StatelessSession не держит объекты в памяти,
                    // поэтому flush/clear не нужны
                }
                
                tx.commit();
            } catch (Exception e) {
                tx.rollback();
                throw e;
            }
        }
    }
}
```

### Сравнение с обычной Session

```java
// Обычная Session — нужен flush + clear каждые N записей
@Transactional
public void bulkInsertWithSession(List<UserDto> dtos) {
    int batchSize = 50;
    for (int i = 0; i < dtos.size(); i++) {
        User user = mapToEntity(dtos.get(i));
        entityManager.persist(user);
        
        if (i % batchSize == 0 && i > 0) {
            entityManager.flush();  // отправляем batch в БД
            entityManager.clear(); // освобождаем Persistence Context
            // без clear() — память растёт, dirty checking замедляется
        }
    }
}

// StatelessSession — flush/clear не нужны, нет PC
public void bulkInsertWithStatelessSession(List<UserDto> dtos) {
    try (StatelessSession session = sessionFactory.openStatelessSession()) {
        Transaction tx = session.beginTransaction();
        for (UserDto dto : dtos) {
            session.insert(mapToEntity(dto));
            // объект не удерживается в памяти — GC может его забрать
        }
        tx.commit();
    }
}
```

### Конфигурация для максимальной производительности

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true    # группировать INSERT по типу entity
        order_updates: true    # группировать UPDATE по типу entity
```

> [!INFO]
> `StatelessSession` позволяет вставить 1 миллион строк за секунды вместо минут, потому что отсутствие Persistence Context устраняет накладные расходы на dirty checking, snapshot и identity map.

---

**Связанные файлы:**
- [[Batch операции]] — конфигурация батчинга и паттерны
- [[Dirty Checking и FlushMode]] — почему flush+clear нужен в обычной Session
- [[First Level Cache vs Second Level Cache]] — что кэшируется, а что нет
