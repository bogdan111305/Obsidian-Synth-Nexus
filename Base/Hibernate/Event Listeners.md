**Event Listeners в Hibernate**

Event Listeners (слушатели событий) в Hibernate — это механизм, позволяющий разработчикам реагировать на определённые события в жизненном цикле сущностей или других операциях Hibernate, таких как сохранение, обновление, удаление или загрузка данных. Они обеспечивают возможность внедрения пользовательской логики в процесс работы Hibernate, что делает их мощным инструментом для кастомизации поведения приложения.

**Зачем нужны Event Listeners?**

- **Кастомизация поведения**: Позволяют выполнять дополнительную логику при возникновении событий, например, логирование, аудит, валидация или модификация данных.
- **Автоматизация задач**: Упрощают реализацию сквозной функциональности, такой как установка временных меток или проверка данных перед сохранением.
- **Гибкость**: Поддерживают интеграцию с внешними системами или бизнес-логикой без изменения основного кода сущностей.

**Основные элементы и их использование**

**Система событий в Hibernate**

Hibernate предоставляет встроенную систему событий, которая генерируется во время операций с сущностями. Эти события связаны с жизненным циклом сущностей (например, создание, обновление, удаление) или другими действиями, такими как выполнение запросов. Слушатели событий регистрируются для обработки этих событий.

**Типы событий**:

- `PreInsertEvent`: Перед вставкой сущности в базу данных.
- `PostInsertEvent`: После вставки сущности.
- `PreUpdateEvent`: Перед обновлением сущности.
- `PostUpdateEvent`: После обновления сущности.
- `PreDeleteEvent`: Перед удалением сущности.
- `PostDeleteEvent`: После удаления сущности.
- `PostLoadEvent`: После загрузки сущности из базы данных.
- Другие: События для выполнения запросов, управления транзакциями и т.д.

**Event Listener**

Event Listener — это класс, реализующий интерфейсы из пакета `org.hibernate.event.spi`, соответствующие конкретным событиям. Каждый интерфейс определяет методы для обработки определённого события.

**Как использовать**:

1. Создайте класс, реализующий интерфейс слушателя (например, `PreInsertEventListener`).
2. Реализуйте метод обработки события (например, `onPreInsert`).
3. Зарегистрируйте слушатель в конфигурации Hibernate.

**Пример: Слушатель для аудита перед вставкой**

```java
import org.hibernate.event.spi.PreInsertEvent;
import org.hibernate.event.spi.PreInsertEventListener;
import java.io.Serializable;
import java.time.LocalDateTime;

public class AuditPreInsertListener implements PreInsertEventListener {
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        Object entity = event.getEntity();
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setCreatedBy("system"); // Пример значения
            // Обновляем состояние сущности
            event.getState()[getCreatedAtIndex(event)] = auditable.getCreatedAt();
            event.getState()[getCreatedByIndex(event)] = auditable.getCreatedBy();
        }
        return false; // false означает, что вставка продолжается
    }

    private int getCreatedAtIndex(PreInsertEvent event) {
        String[] propertyNames = event.getPersister().getPropertyNames();
        for (int i = 0; i < propertyNames.length; i++) {
            if ("createdAt".equals(propertyNames[i])) {
                return i;
            }
        }
        return -1;
    }

    private int getCreatedByIndex(PreInsertEvent event) {
        String[] propertyNames = event.getPersister().getPropertyNames();
        for (int i = 0; i < propertyNames.length; i++) {
            if ("createdBy".equals(propertyNames[i])) {
                return i;
            }
        }
        return -1;
    }
}

interface Auditable {
    void setCreatedAt(LocalDateTime createdAt);
    void setCreatedBy(String createdBy);
    LocalDateTime getCreatedAt();
    String getCreatedBy();
}
```

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import java.time.LocalDateTime;

@Entity
public class User implements Auditable {
    @Id
    private Long id;
    private String name;
    private LocalDateTime createdAt;
    private String createdBy;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    @Override
    public LocalDateTime getCreatedAt() { return createdAt; }
    @Override
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    @Override
    public String getCreatedBy() { return createdBy; }
    @Override
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
}
```

**Регистрация слушателя**:  
Слушатель можно зарегистрировать через конфигурацию Hibernate или Spring:

```java
import org.hibernate.event.service.spi.EventListenerRegistry;
import org.hibernate.event.spi.EventType;
import org.hibernate.internal.SessionFactoryImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.persistence.EntityManagerFactory;

@Component
public class HibernateListenerConfigurer {

    @Autowired
    private EntityManagerFactory entityManagerFactory;

    @Autowired
    private AuditPreInsertListener auditPreInsertListener;

    public void registerListeners() {
        SessionFactoryImpl sessionFactory = entityManagerFactory.unwrap(SessionFactoryImpl.class);
        EventListenerRegistry registry = sessionFactory.getServiceRegistry().getService(EventListenerRegistry.class);
        registry.getEventListenerGroup(EventType.PRE_INSERT).appendListener(auditPreInsertListener);
    }
}
```

В этом примере слушатель `AuditPreInsertListener` автоматически устанавливает поля `createdAt` и `createdBy` для сущностей, реализующих интерфейс `Auditable`, перед вставкой в базу данных.

**Интеграция через аннотации (@EntityListeners)**

JPA предоставляет альтернативный способ реализации слушателей через аннотацию `@EntityListeners`. Это позволяет привязать логику обработки событий к конкретной сущности.

**Как использовать**:

1. Создайте класс с методами, помеченными аннотациями, такими как `@PrePersist`, `@PostPersist`, `@PreUpdate`, `@PostLoad`.
2. Укажите этот класс в аннотации `@EntityListeners` в сущности.

**Пример: Слушатель с использованием @EntityListeners**

```java
import javax.persistence.PrePersist;
import javax.persistence.PostPersist;
import java.time.LocalDateTime;

public class AuditListener {
    @PrePersist
    public void setCreatedAt(Auditable entity) {
        entity.setCreatedAt(LocalDateTime.now());
        entity.setCreatedBy("system");
    }

    @PostPersist
    public void logCreation(Auditable entity) {
        System.out.println("Entity created: " + entity);
    }
}
```

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.EntityListeners;
import java.time.LocalDateTime;

@Entity
@EntityListeners(AuditListener.class)
public class User implements Auditable {
    @Id
    private Long id;
    private String name;
    private LocalDateTime createdAt;
    private String createdBy;

    // Геттеры и сеттеры
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    @Override
    public LocalDateTime getCreatedAt() { return createdAt; }
    @Override
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    @Override
    public String getCreatedBy() { return createdBy; }
    @Override
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
}
```

В этом примере `AuditListener` автоматически устанавливает поля `createdAt` и `createdBy` перед сохранением сущности `User` и логирует создание после вставки.

**Интеграторы (Integrator)**

Для автоматической регистрации слушателей на уровне приложения Hibernate предоставляет механизм интеграторов (`Integrator`). Это полезно для глобальной настройки слушателей без изменения конфигурации каждой сущности.

**Как использовать**:

1. Реализуйте интерфейс `org.hibernate.integrator.spi.Integrator`.
2. Зарегистрируйте интегратор в конфигурации Hibernate через свойство `hibernate.integrator_provider`.

**Пример: Регистрация слушателя через Integrator**

```java
import org.hibernate.boot.Metadata;
import org.hibernate.engine.spi.SessionFactoryImplementor;
import org.hibernate.event.service.spi.EventListenerRegistry;
import org.hibernate.event.spi.EventType;
import org.hibernate.integrator.spi.Integrator;
import org.hibernate.service.spi.SessionFactoryServiceRegistry;

public class AuditEventIntegrator implements Integrator {
    @Override
    public void integrate(Metadata metadata, SessionFactoryImplementor sessionFactory,
                          SessionFactoryServiceRegistry serviceRegistry) {
        EventListenerRegistry registry = serviceRegistry.getService(EventListenerRegistry.class);
        registry.getEventListenerGroup(EventType.PRE_INSERT).appendListener(new AuditPreInsertListener());
    }

    @Override
    public void disintegrate(SessionFactoryImplementor sessionFactory, SessionFactoryServiceRegistry serviceRegistry) {
        // Очистка, если требуется
    }
}
```

**Конфигурация в persistence.xml**:

```xml
<property name="hibernate.integrator_provider"
          value="com.example.AuditEventIntegrator"/>
```

**Итог**

- **Basic Event Listeners**: Реализуются через интерфейсы `org.hibernate.event.spi`, такие как `PreInsertEventListener`, для обработки событий жизненного цикла сущностей.
- **@EntityListeners**: JPA-механизм для привязки слушателей к конкретным сущностям с помощью аннотаций (`@PrePersist`, `@PostPersist` и т.д.).
- **Integrators**: Позволяют глобально регистрировать слушатели для всех сущностей через конфигурацию Hibernate.

Event Listeners в Hibernate предоставляют гибкий способ внедрения пользовательской логики в процесс работы с сущностями, делая их незаменимым инструментом для реализации аудита, валидации и других задач.