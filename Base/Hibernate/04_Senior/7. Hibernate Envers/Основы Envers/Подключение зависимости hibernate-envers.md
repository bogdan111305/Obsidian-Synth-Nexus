# Подключение зависимости hibernate-envers

## Оглавление
1. [Введение](#введение)
2. [Maven зависимость](#maven-зависимость)
3. [Gradle зависимость](#gradle-зависимость)
4. [Конфигурация в persistence.xml](#конфигурация-в-persistencexml)
5. [Конфигурация в hibernate.cfg.xml](#конфигурация-в-hibernatecfgxml)
6. [Spring Boot конфигурация](#spring-boot-конфигурация)
7. [Проверка подключения](#проверка-подключения)
8. [Возможные проблемы](#возможные-проблемы)

---

## <a name="введение"></a>Введение

**Hibernate Envers** — это модуль Hibernate для автоматического аудита и версионирования сущностей. Он позволяет отслеживать все изменения в сущностях, создавая историю изменений с возможностью восстановления предыдущих версий данных.

### Основные возможности Envers

- **Автоматический аудит**: Отслеживание всех изменений в сущностях
- **Версионирование**: Создание ревизий с метаинформацией
- **Восстановление данных**: Возможность отката к предыдущим версиям
- **Гибкая настройка**: Кастомизация аудита на уровне полей и сущностей
- **Производительность**: Оптимизированные запросы к аудиту

### Архитектура Envers

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Основная      │    │   Audit          │    │   Revision      │
│   таблица       │    │   таблица        │    │   таблица       │
│   (Entity)      │◄──►│   (_AUD)         │◄──►│   (REVINFO)     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

---

## <a name="maven-зависимость"></a>Maven зависимость

### Базовое подключение

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-envers</artifactId>
    <version>6.4.0.Final</version>
</dependency>
```

### Полный пример pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.example</groupId>
    <artifactId>hibernate-envers-demo</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <hibernate.version>6.4.0.Final</hibernate.version>
    </properties>
    
    <dependencies>
        <!-- Hibernate Core -->
        <dependency>
            <groupId>org.hibernate.orm</groupId>
            <artifactId>hibernate-core</artifactId>
            <version>${hibernate.version}</version>
        </dependency>
        
        <!-- Hibernate Envers -->
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-envers</artifactId>
            <version>${hibernate.version}</version>
        </dependency>
        
        <!-- Database Driver (пример для PostgreSQL) -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.7.1</version>
        </dependency>
        
        <!-- Logging -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.9</version>
        </dependency>
        
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.4.11</version>
        </dependency>
    </dependencies>
</project>
```

### Версии для разных Hibernate

| Hibernate Version | Envers Version | Java Version |
|------------------|----------------|--------------|
| 6.4.x           | 6.4.0.Final   | 17+          |
| 6.3.x           | 6.3.0.Final   | 17+          |
| 6.2.x           | 6.2.0.Final   | 17+          |
| 6.1.x           | 6.1.0.Final   | 17+          |
| 5.6.x           | 5.6.15.Final  | 8+           |

---

## <a name="gradle-зависимость"></a>Gradle зависимость

### Базовое подключение

```gradle
dependencies {
    implementation 'org.hibernate:hibernate-envers:6.4.0.Final'
}
```

### Полный пример build.gradle

```gradle
plugins {
    id 'java'
    id 'application'
}

group = 'com.example'
version = '1.0.0'

repositories {
    mavenCentral()
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

dependencies {
    // Hibernate Core
    implementation 'org.hibernate.orm:hibernate-core:6.4.0.Final'
    
    // Hibernate Envers
    implementation 'org.hibernate:hibernate-envers:6.4.0.Final'
    
    // Database Driver (пример для PostgreSQL)
    implementation 'org.postgresql:postgresql:42.7.1'
    
    // Logging
    implementation 'org.slf4j:slf4j-api:2.0.9'
    implementation 'ch.qos.logback:logback-classic:1.4.11'
    
    // Testing
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
    testImplementation 'com.h2database:h2:2.2.224'
}

test {
    useJUnitPlatform()
}
```

---

## <a name="конфигурация-в-persistencexml"></a>Конфигурация в persistence.xml

### Базовая конфигурация

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
             http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd"
             version="2.2">
    
    <persistence-unit name="envers-demo" transaction-type="RESOURCE_LOCAL">
        
        <!-- Entity classes -->
        <class>com.example.entity.User</class>
        <class>com.example.entity.Order</class>
        
        <!-- Properties -->
        <properties>
            <!-- Database connection -->
            <property name="javax.persistence.jdbc.driver" value="org.postgresql.Driver"/>
            <property name="javax.persistence.jdbc.url" value="jdbc:postgresql://localhost:5432/envers_demo"/>
            <property name="javax.persistence.jdbc.user" value="postgres"/>
            <property name="javax.persistence.jdbc.password" value="password"/>
            
            <!-- Hibernate properties -->
            <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQLDialect"/>
            <property name="hibernate.hbm2ddl.auto" value="create-drop"/>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            
            <!-- Envers properties -->
            <property name="hibernate.envers.audit_table_suffix" value="_AUD"/>
            <property name="hibernate.envers.revision_field_name" value="REV"/>
            <property name="hibernate.envers.revision_type_field_name" value="REVTYPE"/>
            <property name="hibernate.envers.store_data_at_delete" value="true"/>
            <property name="hibernate.envers.audit_strategy" value="org.hibernate.envers.strategy.DefaultAuditStrategy"/>
        </properties>
    </persistence-unit>
</persistence>
```

### Расширенная конфигурация

```xml
<persistence-unit name="envers-advanced" transaction-type="RESOURCE_LOCAL">
    
    <class>com.example.entity.User</class>
    <class>com.example.entity.Order</class>
    <class>com.example.entity.Product</class>
    
    <properties>
        <!-- Database -->
        <property name="javax.persistence.jdbc.driver" value="org.postgresql.Driver"/>
        <property name="javax.persistence.jdbc.url" value="jdbc:postgresql://localhost:5432/envers_demo"/>
        <property name="javax.persistence.jdbc.user" value="postgres"/>
        <property name="javax.persistence.jdbc.password" value="password"/>
        
        <!-- Hibernate -->
        <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQLDialect"/>
        <property name="hibernate.hbm2ddl.auto" value="create-drop"/>
        <property name="hibernate.show_sql" value="true"/>
        <property name="hibernate.format_sql" value="true"/>
        <property name="hibernate.use_sql_comments" value="true"/>
        
        <!-- Envers Configuration -->
        <property name="hibernate.envers.audit_table_suffix" value="_AUD"/>
        <property name="hibernate.envers.audit_table_prefix" value=""/>
        <property name="hibernate.envers.revision_field_name" value="REV"/>
        <property name="hibernate.envers.revision_type_field_name" value="REVTYPE"/>
        <property name="hibernate.envers.revision_listener" value="com.example.listener.CustomRevisionListener"/>
        <property name="hibernate.envers.store_data_at_delete" value="true"/>
        <property name="hibernate.envers.audit_strategy" value="org.hibernate.envers.strategy.DefaultAuditStrategy"/>
        <property name="hibernate.envers.auto_register_audit_listener" value="true"/>
        <property name="hibernate.envers.audit_strategy_validity_end_rev_field_name" value="REVEND"/>
        <property name="hibernate.envers.audit_strategy_validity_store_revend_timestamp" value="true"/>
        <property name="hibernate.envers.audit_strategy_validity_revend_timestamp_field_name" value="REVEND_TSTMP"/>
    </properties>
</persistence-unit>
```

---

## <a name="конфигурация-в-hibernatecfgxml"></a>Конфигурация в hibernate.cfg.xml

### Базовая конфигурация

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        
        <!-- Database connection -->
        <property name="hibernate.connection.driver_class">org.postgresql.Driver</property>
        <property name="hibernate.connection.url">jdbc:postgresql://localhost:5432/envers_demo</property>
        <property name="hibernate.connection.username">postgres</property>
        <property name="hibernate.connection.password">password</property>
        
        <!-- Hibernate properties -->
        <property name="hibernate.dialect">org.hibernate.dialect.PostgreSQLDialect</property>
        <property name="hibernate.hbm2ddl.auto">create-drop</property>
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>
        
        <!-- Envers properties -->
        <property name="hibernate.envers.audit_table_suffix">_AUD</property>
        <property name="hibernate.envers.revision_field_name">REV</property>
        <property name="hibernate.envers.revision_type_field_name">REVTYPE</property>
        <property name="hibernate.envers.store_data_at_delete">true</property>
        
        <!-- Entity mappings -->
        <mapping class="com.example.entity.User"/>
        <mapping class="com.example.entity.Order"/>
        
    </session-factory>
</hibernate-configuration>
```

---

## <a name="spring-boot-конфигурация"></a>Spring Boot конфигурация

### Maven зависимость для Spring Boot

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-envers</artifactId>
</dependency>
```

### application.yml конфигурация

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/envers_demo
    username: postgres
    password: password
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        
        # Envers Configuration
        envers:
          audit_table_suffix: _AUD
          revision_field_name: REV
          revision_type_field_name: REVTYPE
          store_data_at_delete: true
          audit_strategy: org.hibernate.envers.strategy.DefaultAuditStrategy
          auto_register_audit_listener: true
```

### application.properties конфигурация

```properties
# Database
spring.datasource.url=jdbc:postgresql://localhost:5432/envers_demo
spring.datasource.username=postgres
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA/Hibernate
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Envers
spring.jpa.properties.hibernate.envers.audit_table_suffix=_AUD
spring.jpa.properties.hibernate.envers.revision_field_name=REV
spring.jpa.properties.hibernate.envers.revision_type_field_name=REVTYPE
spring.jpa.properties.hibernate.envers.store_data_at_delete=true
spring.jpa.properties.hibernate.envers.audit_strategy=org.hibernate.envers.strategy.DefaultAuditStrategy
spring.jpa.properties.hibernate.envers.auto_register_audit_listener=true
```

---

## <a name="проверка-подключения"></a>Проверка подключения

### Простой тест подключения

```java
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;
import org.hibernate.envers.AuditReader;
import org.hibernate.envers.AuditReaderFactory;

public class EnversConnectionTest {
    
    public static void main(String[] args) {
        try {
            // Создание SessionFactory
            SessionFactory sessionFactory = new Configuration()
                .configure("hibernate.cfg.xml")
                .buildSessionFactory();
            
            // Проверка создания Session
            Session session = sessionFactory.openSession();
            System.out.println("✅ Session создана успешно");
            
            // Проверка создания AuditReader
            AuditReader auditReader = AuditReaderFactory.get(session);
            System.out.println("✅ AuditReader создан успешно");
            
            session.close();
            sessionFactory.close();
            
            System.out.println("✅ Envers подключен и работает корректно");
            
        } catch (Exception e) {
            System.err.println("❌ Ошибка подключения Envers: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

### Тест с JUnit

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;
import org.hibernate.envers.AuditReader;
import org.hibernate.envers.AuditReaderFactory;

import static org.junit.jupiter.api.Assertions.*;

public class EnversSetupTest {
    
    private SessionFactory sessionFactory;
    
    @BeforeEach
    void setUp() {
        sessionFactory = new Configuration()
            .configure("hibernate.cfg.xml")
            .buildSessionFactory();
    }
    
    @Test
    void testEnversConnection() {
        Session session = sessionFactory.openSession();
        
        // Проверяем, что можем создать AuditReader
        AuditReader auditReader = AuditReaderFactory.get(session);
        assertNotNull(auditReader, "AuditReader должен быть создан");
        
        session.close();
    }
    
    @Test
    void testSessionFactoryCreation() {
        assertNotNull(sessionFactory, "SessionFactory должен быть создан");
        assertFalse(sessionFactory.isClosed(), "SessionFactory не должен быть закрыт");
    }
}
```

---

## <a name="возможные-проблемы"></a>Возможные проблемы

### 1. Несовместимость версий

**Проблема**: Версия Envers не соответствует версии Hibernate Core.

**Решение**:
```xml
<!-- Убедитесь, что версии совпадают -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>6.4.0.Final</version>
</dependency>
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-envers</artifactId>
    <version>6.4.0.Final</version>
</dependency>
```

### 2. Отсутствие аудита таблиц

**Проблема**: Таблицы аудита не создаются автоматически.

**Решение**:
```properties
# Убедитесь, что включено автоматическое создание схемы
hibernate.hbm2ddl.auto=create-drop
```

### 3. Ошибки в логах

**Проблема**: Появляются предупреждения о том, что Envers не найден.

**Решение**:
```properties
# Добавьте в конфигурацию
hibernate.envers.auto_register_audit_listener=true
```

### 4. Проблемы с транзакциями

**Проблема**: Аудит не работает в транзакциях.

**Решение**:
```java
// Убедитесь, что используете транзакции
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

try {
    // Ваши операции с сущностями
    session.save(entity);
    tx.commit();
} catch (Exception e) {
    tx.rollback();
    throw e;
} finally {
    session.close();
}
```

### 5. Проблемы с Spring Boot

**Проблема**: Envers не работает в Spring Boot приложении.

**Решение**:
```java
@Configuration
public class EnversConfig {
    
    @Bean
    public HibernatePropertiesCustomizer hibernatePropertiesCustomizer() {
        return hibernateProperties -> {
            hibernateProperties.put("hibernate.envers.audit_table_suffix", "_AUD");
            hibernateProperties.put("hibernate.envers.revision_field_name", "REV");
            hibernateProperties.put("hibernate.envers.revision_type_field_name", "REVTYPE");
            hibernateProperties.put("hibernate.envers.store_data_at_delete", "true");
        };
    }
}
```

---

## Заключение

Правильная настройка Hibernate Envers включает в себя:

1. **Подключение зависимости** - добавление hibernate-envers в проект
2. **Конфигурация** - настройка параметров в persistence.xml или application.properties
3. **Проверка** - тестирование подключения и базовой функциональности
4. **Устранение проблем** - решение возможных конфликтов версий и конфигурации

После успешной настройки Envers готов к использованию для аудита сущностей. Следующий шаг - изучение аннотаций @Audited и @NotAudited для настройки аудита на уровне сущностей. 