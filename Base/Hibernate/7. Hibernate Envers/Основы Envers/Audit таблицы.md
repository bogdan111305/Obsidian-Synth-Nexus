# Audit таблицы

## Оглавление
1. [Введение](#введение)
2. [Структура audit таблиц](#структура-audit-таблиц)
3. [Создание audit таблиц](#создание-audit-таблиц)
4. [Настройка имен таблиц](#настройка-имен-таблиц)
5. [Структура таблицы REVINFO](#структура-таблицы-revinfo)
6. [Типы изменений](#типы-изменений)
7. [Практические примеры](#практические-примеры)
8. [Лучшие практики](#лучшие-практики)

---

## <a name="введение"></a>Введение

**Audit таблицы** — это специальные таблицы, которые Hibernate Envers создает автоматически для хранения истории изменений сущностей. Каждая аудируемая сущность получает соответствующую audit таблицу, которая содержит все версии данных с метаинформацией о времени и типе изменений.

### Основные концепции

- **Audit таблица** - таблица для хранения истории изменений
- **REVINFO таблица** - таблица с метаинформацией о ревизиях
- **REV поле** - ссылка на номер ревизии
- **REVTYPE поле** - тип изменения (INSERT, UPDATE, DELETE)
- **Автоматическое создание** - таблицы создаются при инициализации Hibernate

---

## <a name="структура-audit-таблиц"></a>Структура audit таблиц

### Базовая структура

```
┌─────────────────────────────────────────────────────────────┐
│                    Audit таблица                           │
├─────────────────────────────────────────────────────────────┤
│ REV (INTEGER) - Номер ревизии                             │
│ REVTYPE (TINYINT) - Тип изменения (0=ADD, 1=MOD, 2=DEL) │
│ [Все поля оригинальной сущности]                          │
└─────────────────────────────────────────────────────────────┘
```

### Пример структуры

#### Оригинальная таблица User
```sql
CREATE TABLE user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    email VARCHAR(255),
    created_at TIMESTAMP
);
```

#### Audit таблица User_AUD
```sql
CREATE TABLE user_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    email VARCHAR(255),
    created_at TIMESTAMP,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

### Структура с коллекциями

```java
@Entity
@Audited
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "department")
    @Audited
    private List<Employee> employees;
}
```

#### Audit таблицы
```sql
-- Основная audit таблица
CREATE TABLE department_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV)
);

-- Audit таблица для коллекции
CREATE TABLE department_employees_aud (
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    department_id BIGINT NOT NULL,
    employees_id BIGINT NOT NULL,
    PRIMARY KEY (REV, department_id, employees_id)
);
```

---

## <a name="создание-audit-таблиц"></a>Создание audit таблиц

### Автоматическое создание

#### 1. Через hbm2ddl.auto

```properties
# В persistence.xml или application.properties
hibernate.hbm2ddl.auto=create-drop
```

#### 2. Программное создание

```java
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;
import org.hibernate.tool.schema.SchemaExport;

public class AuditTableCreator {
    
    public static void createAuditTables() {
        Configuration configuration = new Configuration()
            .configure("hibernate.cfg.xml");
        
        SchemaExport schemaExport = new SchemaExport(configuration);
        schemaExport.create(true, true);
    }
    
    public static void main(String[] args) {
        createAuditTables();
        System.out.println("Audit таблицы созданы успешно");
    }
}
```

### Ручное создание

#### 1. SQL скрипт для создания audit таблиц

```sql
-- Создание таблицы ревизий
CREATE TABLE revinfo (
    REV INTEGER PRIMARY KEY AUTO_INCREMENT,
    REVTSTMP BIGINT
);

-- Создание audit таблицы для User
CREATE TABLE user_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    email VARCHAR(255),
    created_at TIMESTAMP,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

-- Создание audit таблицы для Product
CREATE TABLE product_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    price DECIMAL(10,2),
    description TEXT,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

#### 2. Spring Boot с Flyway

```java
@Component
public class AuditTableMigration implements CommandLineRunner {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public void run(String... args) throws Exception {
        // Создание таблицы ревизий
        jdbcTemplate.execute("""
            CREATE TABLE IF NOT EXISTS revinfo (
                REV INTEGER PRIMARY KEY AUTO_INCREMENT,
                REVTSTMP BIGINT
            )
        """);
        
        // Создание audit таблиц
        createAuditTables();
    }
    
    private void createAuditTables() {
        // Создание audit таблиц для всех аудируемых сущностей
        String[] entities = {"user", "product", "order"};
        
        for (String entity : entities) {
            String sql = String.format("""
                CREATE TABLE IF NOT EXISTS %s_aud (
                    id BIGINT NOT NULL,
                    REV INTEGER NOT NULL,
                    REVTYPE TINYINT,
                    PRIMARY KEY (id, REV),
                    FOREIGN KEY (REV) REFERENCES revinfo(REV)
                )
            """, entity);
            
            jdbcTemplate.execute(sql);
        }
    }
}
```

---

## <a name="настройка-имен-таблиц"></a>Настройка имен таблиц

### Конфигурация суффиксов и префиксов

```properties
# В persistence.xml или application.properties
hibernate.envers.audit_table_suffix=_AUD
hibernate.envers.audit_table_prefix=
```

### Кастомные имена таблиц

#### 1. На уровне аннотации

```java
@Entity
@Audited(auditTableName = "user_history")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    // getters and setters
}
```

#### 2. На уровне конфигурации

```properties
# Глобальная настройка
hibernate.envers.audit_table_suffix=_HISTORY
hibernate.envers.audit_table_prefix=AUDIT_
```

#### 3. Программная настройка

```java
@Configuration
public class EnversConfig {
    
    @Bean
    public HibernatePropertiesCustomizer hibernatePropertiesCustomizer() {
        return hibernateProperties -> {
            hibernateProperties.put("hibernate.envers.audit_table_suffix", "_VERSIONS");
            hibernateProperties.put("hibernate.envers.audit_table_prefix", "");
        };
    }
}
```

### Примеры именования

| Конфигурация | Оригинальная таблица | Audit таблица |
|--------------|---------------------|---------------|
| `_AUD` | `user` | `user_aud` |
| `_HISTORY` | `product` | `product_history` |
| `AUDIT_` | `order` | `audit_order` |
| Кастомное имя | `customer` | `customer_versions` |

---

## <a name="структура-таблицы-revinfo"></a>Структура таблицы REVINFO

### Базовая структура

```sql
CREATE TABLE revinfo (
    REV INTEGER PRIMARY KEY AUTO_INCREMENT,
    REVTSTMP BIGINT
);
```

### Расширенная структура с кастомными полями

```java
@Entity
@RevisionEntity(CustomRevisionListener.class)
public class CustomRevisionEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @RevisionNumber
    private Integer id;
    
    @RevisionTimestamp
    private Long timestamp;
    
    private String username;
    private String ipAddress;
    private String action;
    
    // getters and setters
}
```

### Структура с дополнительными полями

```sql
CREATE TABLE revinfo (
    REV INTEGER PRIMARY KEY AUTO_INCREMENT,
    REVTSTMP BIGINT,
    username VARCHAR(255),
    ip_address VARCHAR(45),
    action VARCHAR(50)
);
```

### Примеры данных в REVINFO

```sql
-- Вставка данных в таблицу ревизий
INSERT INTO revinfo (REV, REVTSTMP, username, ip_address, action) VALUES
(1, 1640995200000, 'admin', '192.168.1.100', 'CREATE'),
(2, 1640995260000, 'user1', '192.168.1.101', 'UPDATE'),
(3, 1640995320000, 'user2', '192.168.1.102', 'DELETE');
```

---

## <a name="типы-изменений"></a>Типы изменений

### Значения REVTYPE

| Значение | Тип | Описание |
|----------|-----|----------|
| 0 | ADD | Добавление новой записи |
| 1 | MOD | Изменение существующей записи |
| 2 | DEL | Удаление записи |

### Примеры в audit таблицах

#### 1. Добавление пользователя

```sql
-- Вставка в user_aud при создании пользователя
INSERT INTO user_aud (id, name, email, created_at, REV, REVTYPE) VALUES
(1, 'John Doe', 'john@example.com', '2024-01-01 10:00:00', 1, 0);
```

#### 2. Обновление пользователя

```sql
-- Вставка в user_aud при обновлении пользователя
INSERT INTO user_aud (id, name, email, created_at, REV, REVTYPE) VALUES
(1, 'John Smith', 'john.smith@example.com', '2024-01-01 10:00:00', 2, 1);
```

#### 3. Удаление пользователя

```sql
-- Вставка в user_aud при удалении пользователя
INSERT INTO user_aud (id, name, email, created_at, REV, REVTYPE) VALUES
(1, 'John Smith', 'john.smith@example.com', '2024-01-01 10:00:00', 3, 2);
```

### Программная работа с типами изменений

```java
import org.hibernate.envers.RevisionType;

public class AuditTypeExample {
    
    public static void printRevisionType(int revType) {
        switch (revType) {
            case 0:
                System.out.println("Тип изменения: ADD (Добавление)");
                break;
            case 1:
                System.out.println("Тип изменения: MOD (Изменение)");
                break;
            case 2:
                System.out.println("Тип изменения: DEL (Удаление)");
                break;
            default:
                System.out.println("Неизвестный тип изменения: " + revType);
        }
    }
    
    public static String getRevisionTypeName(int revType) {
        return switch (revType) {
            case 0 -> "ADD";
            case 1 -> "MOD";
            case 2 -> "DEL";
            default -> "UNKNOWN";
        };
    }
}
```

---

## <a name="практические-примеры"></a>Практические примеры

### Пример 1: Система управления продуктами

```java
@Entity
@Audited
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private BigDecimal price;
    private String description;
    private Integer stockQuantity;
    
    @Enumerated(EnumType.STRING)
    private ProductStatus status;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // getters and setters
}
```

#### Созданные audit таблицы

```sql
-- Основная таблица
CREATE TABLE product (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    price DECIMAL(10,2),
    description TEXT,
    stock_quantity INTEGER,
    status VARCHAR(50),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Audit таблица
CREATE TABLE product_aud (
    id BIGINT NOT NULL,
    name VARCHAR(255),
    price DECIMAL(10,2),
    description TEXT,
    stock_quantity INTEGER,
    status VARCHAR(50),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

### Пример 2: Система управления заказами с коллекциями

```java
@Entity
@Audited
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String orderNumber;
    private LocalDateTime orderDate;
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    @Audited
    private List<OrderItem> items;
    
    // getters and setters
}

@Entity
@Audited
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    private Order order;
    
    @ManyToOne
    @Audited
    private Product product;
    
    private Integer quantity;
    private BigDecimal unitPrice;
    
    // getters and setters
}
```

#### Созданные audit таблицы

```sql
-- Основные таблицы
CREATE TABLE `order` (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_number VARCHAR(255),
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(50)
);

CREATE TABLE order_item (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT,
    product_id BIGINT,
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    FOREIGN KEY (order_id) REFERENCES `order`(id),
    FOREIGN KEY (product_id) REFERENCES product(id)
);

-- Audit таблицы
CREATE TABLE order_aud (
    id BIGINT NOT NULL,
    order_number VARCHAR(255),
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(50),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

CREATE TABLE order_item_aud (
    id BIGINT NOT NULL,
    order_id BIGINT,
    product_id BIGINT,
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);

-- Audit таблица для коллекции
CREATE TABLE order_items_aud (
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    order_id BIGINT NOT NULL,
    items_id BIGINT NOT NULL,
    PRIMARY KEY (REV, order_id, items_id),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

### Пример 3: Система управления пользователями с наследованием

```java
@MappedSuperclass
@Audited
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    @ManyToOne
    @Audited
    private User createdBy;
    
    @ManyToOne
    @Audited
    private User updatedBy;
    
    // getters and setters
}

@Entity
public class Customer extends BaseEntity {
    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    
    @Enumerated(EnumType.STRING)
    private CustomerStatus status;
    
    // getters and setters
}
```

#### Созданные audit таблицы

```sql
-- Основная таблица
CREATE TABLE customer (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    created_by_id BIGINT,
    updated_by_id BIGINT,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(255),
    status VARCHAR(50),
    FOREIGN KEY (created_by_id) REFERENCES user(id),
    FOREIGN KEY (updated_by_id) REFERENCES user(id)
);

-- Audit таблица
CREATE TABLE customer_aud (
    id BIGINT NOT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    created_by_id BIGINT,
    updated_by_id BIGINT,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(255),
    status VARCHAR(50),
    REV INTEGER NOT NULL,
    REVTYPE TINYINT,
    PRIMARY KEY (id, REV),
    FOREIGN KEY (REV) REFERENCES revinfo(REV)
);
```

---

## <a name="лучшие-практики"></a>Лучшие практики

### 1. Именование таблиц

```properties
# Используйте понятные суффиксы
hibernate.envers.audit_table_suffix=_AUDIT
hibernate.envers.audit_table_suffix=_HISTORY
hibernate.envers.audit_table_suffix=_VERSIONS
```

### 2. Структура таблиц

```sql
-- Всегда добавляйте индексы для производительности
CREATE INDEX idx_user_aud_rev ON user_aud(REV);
CREATE INDEX idx_user_aud_revtype ON user_aud(REVTYPE);
CREATE INDEX idx_user_aud_id_rev ON user_aud(id, REV);
```

### 3. Очистка старых данных

```sql
-- Периодическая очистка старых audit записей
DELETE FROM user_aud WHERE REV < (SELECT MAX(REV) - 1000 FROM revinfo);
DELETE FROM revinfo WHERE REV < (SELECT MAX(REV) - 1000 FROM revinfo);
```

### 4. Мониторинг размера таблиц

```sql
-- Проверка размера audit таблиц
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)'
FROM information_schema.tables 
WHERE table_schema = 'your_database' 
AND table_name LIKE '%_aud'
ORDER BY (data_length + index_length) DESC;
```

### 5. Оптимизация запросов

```sql
-- Создание составных индексов для часто используемых запросов
CREATE INDEX idx_user_aud_rev_revtype ON user_aud(REV, REVTYPE);
CREATE INDEX idx_user_aud_id_rev_revtype ON user_aud(id, REV, REVTYPE);
```

---

## Заключение

Audit таблицы в Hibernate Envers предоставляют мощный механизм для отслеживания изменений в данных:

1. **Автоматическое создание** - таблицы создаются автоматически при инициализации
2. **Гибкая настройка** - кастомизация имен и структуры таблиц
3. **Структурированность** - четкая организация данных с метаинформацией
4. **Производительность** - оптимизированные индексы и структура
5. **Масштабируемость** - возможность очистки старых данных

Правильная настройка audit таблиц обеспечивает эффективное хранение и доступ к истории изменений данных. 