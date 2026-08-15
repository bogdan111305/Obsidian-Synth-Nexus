# Hibernate Event model

**Hibernate Event model** — это система событий, которая позволяет перехватывать и обрабатывать различные операции с сущностями на уровне ORM. Эта модель предоставляет более низкоуровневый контроль по сравнению с JPA Entity Callbacks.

## 1. Обзор Hibernate Event model

### 1.1. Что такое Hibernate Event model?

Hibernate Event model — это механизм, который позволяет перехватывать события жизненного цикла сущностей на уровне Hibernate. Он предоставляет более детальный контроль над операциями с базой данных.

### 1.2. Преимущества Hibernate Event model

- **Низкоуровневый контроль**: Доступ к внутренним механизмам Hibernate
- **Гибкость**: Возможность модификации данных перед сохранением
- **Производительность**: Оптимизация на уровне ORM
- **Расширенная функциональность**: Доступ к метаданным и состоянию

## 2. Основные интерфейсы событий

### 2.1. PreInsertEventListener

**Описание**: Выполняется перед вставкой сущности в базу данных.

**Сигнатура**:
```java
public interface PreInsertEventListener extends EventListener {
    boolean onPreInsert(PreInsertEvent event);
}
```

**Пример реализации**:
```java
public class AuditPreInsertListener implements PreInsertEventListener {
    
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
            
            // Обновление состояния
            Object[] state = event.getState();
            String[] propertyNames = event.getPersister().getPropertyNames();
            
            for (int i = 0; i < propertyNames.length; i++) {
                if ("createdAt".equals(propertyNames[i])) {
                    state[i] = auditable.getCreatedAt();
                } else if ("createdBy".equals(propertyNames[i])) {
                    state[i] = auditable.getCreatedBy();
                }
            }
        }
        
        return false; // Продолжить операцию
    }
    
    private String getCurrentUser() {
        try {
            return SecurityContextHolder.getContext().getAuthentication().getName();
        } catch (Exception e) {
            return "system";
        }
    }
}
```

### 2.2. PostInsertEventListener

**Описание**: Выполняется после вставки сущности в базу данных.

**Сигнатура**:
```java
public interface PostInsertEventListener extends EventListener {
    void onPostInsert(PostInsertEvent event);
}
```

**Пример реализации**:
```java
public class NotificationPostInsertListener implements PostInsertEventListener {
    
    @Override
    public void onPostInsert(PostInsertEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Order) {
            Order order = (Order) entity;
            sendOrderNotification(order);
        }
    }
    
    private void sendOrderNotification(Order order) {
        // Логика отправки уведомления
        System.out.println("Order notification sent for: " + order.getOrderNumber());
    }
}
```

### 2.3. PreUpdateEventListener

**Описание**: Выполняется перед обновлением сущности в базе данных.

**Сигнатура**:
```java
public interface PreUpdateEventListener extends EventListener {
    boolean onPreUpdate(PreUpdateEvent event);
}
```

**Пример реализации**:
```java
public class AuditPreUpdateListener implements PreUpdateEventListener {
    
    @Override
    public boolean onPreUpdate(PreUpdateEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
            
            // Обновление состояния
            Object[] state = event.getState();
            String[] propertyNames = event.getPersister().getPropertyNames();
            
            for (int i = 0; i < propertyNames.length; i++) {
                if ("updatedAt".equals(propertyNames[i])) {
                    state[i] = auditable.getUpdatedAt();
                } else if ("updatedBy".equals(propertyNames[i])) {
                    state[i] = auditable.getUpdatedBy();
                }
            }
        }
        
        return false; // Продолжить операцию
    }
    
    private String getCurrentUser() {
        try {
            return SecurityContextHolder.getContext().getAuthentication().getName();
        } catch (Exception e) {
            return "system";
        }
    }
}
```

### 2.4. PostUpdateEventListener

**Описание**: Выполняется после обновления сущности в базе данных.

**Сигнатура**:
```java
public interface PostUpdateEventListener extends EventListener {
    void onPostUpdate(PostUpdateEvent event);
}
```

**Пример реализации**:
```java
public class ChangeTrackingPostUpdateListener implements PostUpdateEventListener {
    
    @Override
    public void onPostUpdate(PostUpdateEvent event) {
        Object entity = event.getEntity();
        Object[] oldState = event.getOldState();
        Object[] newState = event.getState();
        String[] propertyNames = event.getPersister().getPropertyNames();
        
        // Отслеживание изменений
        for (int i = 0; i < propertyNames.length; i++) {
            if (!Objects.equals(oldState[i], newState[i])) {
                System.out.println("Property " + propertyNames[i] + " changed from " + 
                                 oldState[i] + " to " + newState[i]);
            }
        }
    }
}
```

### 2.5. PreDeleteEventListener

**Описание**: Выполняется перед удалением сущности из базы данных.

**Сигнатура**:
```java
public interface PreDeleteEventListener extends EventListener {
    boolean onPreDelete(PreDeleteEvent event);
}
```

**Пример реализации**:
```java
public class BackupPreDeleteListener implements PreDeleteEventListener {
    
    @Override
    public boolean onPreDelete(PreDeleteEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Document) {
            Document document = (Document) entity;
            createDocumentBackup(document);
        }
        
        return false; // Продолжить операцию
    }
    
    private void createDocumentBackup(Document document) {
        // Логика создания резервной копии
        System.out.println("Document backup created for: " + document.getFileName());
    }
}
```

### 2.6. PostDeleteEventListener

**Описание**: Выполняется после удаления сущности из базы данных.

**Сигнатура**:
```java
public interface PostDeleteEventListener extends EventListener {
    void onPostDelete(PostDeleteEvent event);
}
```

**Пример реализации**:
```java
public class CleanupPostDeleteListener implements PostDeleteEventListener {
    
    @Override
    public void onPostDelete(PostDeleteEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof File) {
            File file = (File) entity;
            cleanupFileResources(file);
        }
    }
    
    private void cleanupFileResources(File file) {
        // Логика очистки ресурсов файла
        System.out.println("File resources cleaned for: " + file.getFileName());
    }
}
```

### 2.7. PostLoadEventListener

**Описание**: Выполняется после загрузки сущности из базы данных.

**Сигнатура**:
```java
public interface PostLoadEventListener extends EventListener {
    void onPostLoad(PostLoadEvent event);
}
```

**Пример реализации**:
```java
public class CachePostLoadListener implements PostLoadEventListener {
    
    @Override
    public void onPostLoad(PostLoadEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Cacheable) {
            Cacheable cacheable = (Cacheable) entity;
            initializeCache(cacheable);
        }
    }
    
    private void initializeCache(Cacheable entity) {
        // Логика инициализации кэша
        System.out.println("Cache initialized for: " + entity.getClass().getSimpleName());
    }
}
```

## 3. Универсальные слушатели

### 3.1. Универсальный аудит слушатель

```java
public class UniversalAuditListener implements PreInsertEventListener, PreUpdateEventListener {
    
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
            auditable.setUpdatedBy(getCurrentUser());
            
            updateState(event.getState(), event.getPersister().getPropertyNames(), auditable);
        }
        
        return false;
    }
    
    @Override
    public boolean onPreUpdate(PreUpdateEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
            
            updateState(event.getState(), event.getPersister().getPropertyNames(), auditable);
        }
        
        return false;
    }
    
    private void updateState(Object[] state, String[] propertyNames, Auditable auditable) {
        for (int i = 0; i < propertyNames.length; i++) {
            switch (propertyNames[i]) {
                case "createdAt":
                    state[i] = auditable.getCreatedAt();
                    break;
                case "updatedAt":
                    state[i] = auditable.getUpdatedAt();
                    break;
                case "createdBy":
                    state[i] = auditable.getCreatedBy();
                    break;
                case "updatedBy":
                    state[i] = auditable.getUpdatedBy();
                    break;
            }
        }
    }
    
    private String getCurrentUser() {
        try {
            return SecurityContextHolder.getContext().getAuthentication().getName();
        } catch (Exception e) {
            return "system";
        }
    }
}
```

### 3.2. Универсальный валидатор

```java
public class UniversalValidationListener implements PreInsertEventListener, PreUpdateEventListener {
    
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        return validateEntity(event.getEntity(), "insert");
    }
    
    @Override
    public boolean onPreUpdate(PreUpdateEvent event) {
        return validateEntity(event.getEntity(), "update");
    }
    
    private boolean validateEntity(Object entity, String operation) {
        if (entity instanceof Validatable) {
            Validatable validatable = (Validatable) entity;
            try {
                validatable.validate();
                System.out.println("Validation passed for " + operation + ": " + entity.getClass().getSimpleName());
                return false; // Продолжить операцию
            } catch (ValidationException e) {
                System.err.println("Validation failed for " + operation + ": " + e.getMessage());
                return true; // Прервать операцию
            }
        }
        return false;
    }
}
```

### 3.3. Универсальный логгер

```java
public class UniversalLoggingListener implements PreInsertEventListener, PostInsertEventListener,
                                              PreUpdateEventListener, PostUpdateEventListener,
                                              PreDeleteEventListener, PostDeleteEventListener,
                                              PostLoadEventListener {
    
    private static final Logger logger = LoggerFactory.getLogger(UniversalLoggingListener.class);
    
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        logger.info("PreInsert: {} will be inserted", event.getEntity().getClass().getSimpleName());
        return false;
    }
    
    @Override
    public void onPostInsert(PostInsertEvent event) {
        logger.info("PostInsert: {} inserted successfully", event.getEntity().getClass().getSimpleName());
    }
    
    @Override
    public boolean onPreUpdate(PreUpdateEvent event) {
        logger.info("PreUpdate: {} will be updated", event.getEntity().getClass().getSimpleName());
        return false;
    }
    
    @Override
    public void onPostUpdate(PostUpdateEvent event) {
        logger.info("PostUpdate: {} updated successfully", event.getEntity().getClass().getSimpleName());
    }
    
    @Override
    public boolean onPreDelete(PreDeleteEvent event) {
        logger.info("PreDelete: {} will be deleted", event.getEntity().getClass().getSimpleName());
        return false;
    }
    
    @Override
    public void onPostDelete(PostDeleteEvent event) {
        logger.info("PostDelete: {} deleted successfully", event.getEntity().getClass().getSimpleName());
    }
    
    @Override
    public void onPostLoad(PostLoadEvent event) {
        logger.info("PostLoad: {} loaded successfully", event.getEntity().getClass().getSimpleName());
    }
}
```

## 4. Специализированные слушатели

### 4.1. Слушатель для документов

```java
public class DocumentListener implements PreInsertEventListener, PostInsertEventListener,
                                      PreUpdateEventListener, PostUpdateEventListener,
                                      PreDeleteEventListener, PostDeleteEventListener {
    
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Document) {
            Document document = (Document) entity;
            document.setCreatedAt(LocalDateTime.now());
            document.setStatus("DRAFT");
            
            updateDocumentState(event.getState(), event.getPersister().getPropertyNames(), document);
        }
        
        return false;
    }
    
    @Override
    public void onPostInsert(PostInsertEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Document) {
            Document document = (Document) entity;
            sendDocumentNotification(document, "CREATED");
        }
    }
    
    @Override
    public boolean onPreUpdate(PreUpdateEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Document) {
            Document document = (Document) entity;
            document.setUpdatedAt(LocalDateTime.now());
            
            updateDocumentState(event.getState(), event.getPersister().getPropertyNames(), document);
        }
        
        return false;
    }
    
    @Override
    public void onPostUpdate(PostUpdateEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Document) {
            Document document = (Document) entity;
            sendDocumentNotification(document, "UPDATED");
        }
    }
    
    @Override
    public boolean onPreDelete(PreDeleteEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Document) {
            Document document = (Document) entity;
            createDocumentBackup(document);
        }
        
        return false;
    }
    
    @Override
    public void onPostDelete(PostDeleteEvent event) {
        Object entity = event.getEntity();
        
        if (entity instanceof Document) {
            Document document = (Document) entity;
            cleanupDocumentFiles(document);
        }
    }
    
    private void updateDocumentState(Object[] state, String[] propertyNames, Document document) {
        for (int i = 0; i < propertyNames.length; i++) {
            switch (propertyNames[i]) {
                case "createdAt":
                    state[i] = document.getCreatedAt();
                    break;
                case "updatedAt":
                    state[i] = document.getUpdatedAt();
                    break;
                case "status":
                    state[i] = document.getStatus();
                    break;
            }
        }
    }
    
    private void sendDocumentNotification(Document document, String action) {
        // Логика отправки уведомления
        System.out.println("Document notification sent for: " + document.getFileName() + " - " + action);
    }
    
    private void createDocumentBackup(Document document) {
        // Логика создания резервной копии
        System.out.println("Document backup created for: " + document.getFileName());
    }
    
    private void cleanupDocumentFiles(Document document) {
        // Логика очистки файлов
        System.out.println("Document files cleaned for: " + document.getFileName());
    }
}
```

## 5. Вопросы для собеседования

### Базовые вопросы

1. **Что такое Hibernate Event model?**
   - Система событий для перехвата операций с сущностями
   - Низкоуровневый контроль над ORM операциями
   - Более детальный контроль по сравнению с JPA Callbacks

2. **Какие основные интерфейсы событий доступны?**
   - PreInsertEventListener, PostInsertEventListener
   - PreUpdateEventListener, PostUpdateEventListener
   - PreDeleteEventListener, PostDeleteEventListener
   - PostLoadEventListener

3. **В чем разница между Hibernate Events и JPA Callbacks?**
   - Hibernate Events: низкоуровневый контроль, доступ к состоянию
   - JPA Callbacks: высокоуровневый API, простота использования

### Продвинутые вопросы

4. **Как модифицировать состояние сущности в PreInsertEventListener?**
   - Получить массив state из события
   - Найти нужные свойства по propertyNames
   - Обновить значения в массиве state

5. **Как создать универсальный слушатель для всех сущностей?**
   - Использовать instanceof для проверки типа
   - Применять интерфейсы для типизации
   - Обрабатывать различные типы сущностей

6. **Как тестировать Hibernate Event Listeners?**
   - Unit тесты с моками событий
   - Integration тесты с реальной базой данных
   - Проверка модификации состояния

### Практические вопросы

7. **Реализуйте универсальный аудит слушатель**
```java
public class UniversalAuditListener implements PreInsertEventListener, PreUpdateEventListener {
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        if (event.getEntity() instanceof Auditable) {
            Auditable auditable = (Auditable) event.getEntity();
            auditable.setCreatedAt(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
        }
        return false;
    }
}
```

8. **Создайте слушатель для валидации**
```java
public class ValidationListener implements PreInsertEventListener, PreUpdateEventListener {
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        if (event.getEntity() instanceof Validatable) {
            Validatable validatable = (Validatable) event.getEntity();
            validatable.validate();
        }
        return false;
    }
}
```

9. **Реализуйте слушатель для логирования**
```java
public class LoggingListener implements PreInsertEventListener, PostInsertEventListener {
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        logger.info("PreInsert: {}", event.getEntity().getClass().getSimpleName());
        return false;
    }
    
    @Override
    public void onPostInsert(PostInsertEvent event) {
        logger.info("PostInsert: {}", event.getEntity().getClass().getSimpleName());
    }
}
``` 