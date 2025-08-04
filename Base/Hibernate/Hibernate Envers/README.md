# Hibernate Envers

Этот раздел содержит документацию по Hibernate Envers - модулю для аудита и версионирования сущностей в Hibernate.

## Содержание раздела

### Основные темы

1. **[Part 1: Основы Envers](./Part 1 - Основы Envers/README.md)** - Базовые концепции и настройка
   - Подключение зависимости hibernate-envers
   - Аннотации @Audited и @NotAudited
   - Audit таблицы и их структура
   - Audit зависимых сущностей
   - Базовые примеры использования

2. **[Part 2: Расширенная настройка](./Part 2 - Расширенная настройка/README.md)** - Кастомизация и слушатели
   - Интерфейс EnversListener
   - Классы AuditProcess и AuditWorkUnit
   - Custom revinfo table
   - Класс RevisionListener
   - Продвинутые сценарии аудита

3. **[Part 3: Работа с аудитом](./Part 3 - Работа с аудитом/README.md)** - Чтение и анализ аудита
   - Класс AuditReaderFactory
   - Перемещение по ревизиям
   - Методы класса AuditReader
   - Audit Queries
   - Revision rollback
   - Практические примеры запросов

### Ключевые концепции

- **Аудит (Audit)** - отслеживание изменений в сущностях
- **Ревизии (Revisions)** - версии данных с метаинформацией
- **Audit таблицы** - таблицы для хранения истории изменений
- **RevisionListener** - кастомизация информации о ревизиях
- **AuditReader** - API для чтения аудита
- **Audit Queries** - запросы к аудиту

### Файлы для изучения

- `Part 1 - Основы Envers/` - Базовые концепции и настройка
- `Part 2 - Расширенная настройка/` - Кастомизация и слушатели
- `Part 3 - Работа с аудитом/` - Чтение и анализ аудита

### Связанные разделы

- [Event Listeners and Callbacks](../Event Listeners and Callbacks/README.md)
- [Entity Associations](../Entity Associations/README.md)
- [Transactions & Locks](../Transactions & Locks/README.md)

## Статус разработки

✅ **Завершено:**
- Структура раздела создана
- Part 1: Основы Envers
- Part 2: Расширенная настройка
- Part 3: Работа с аудитом

🔄 **В процессе:**
- Интеграция с другими модулями

📋 **Планируется:**
- Дополнительные примеры
- Интеграция с Spring
- Создание тестов и упражнений 