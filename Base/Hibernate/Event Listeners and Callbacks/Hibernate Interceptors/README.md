# Hibernate Interceptors

Этот раздел содержит документацию по перехватчикам Hibernate, которые позволяют перехватывать и модифицировать операции с сущностями на низком уровне.

## Содержание раздела

### Основные темы

1. **[Hibernate Interceptors.md](./Hibernate Interceptors.md)** - Перехватчики Hibernate
   - Интерфейс Interceptor
   - EmptyInterceptor (deprecated)
   - Регистрация интерцепторов
   - Практические примеры

### Ключевые концепции

- **Interceptor** - интерфейс для перехвата операций Hibernate
- **EmptyInterceptor** - базовая реализация (deprecated с Hibernate 6)
- **Глобальные интерцепторы** - применяются ко всем сессиям
- **Сессионные интерцепторы** - применяются к конкретной сессии
- **Жизненный цикл** - onSave, onLoad, onDelete, onFlushDirty

### Файлы для изучения

- `Hibernate Interceptors.md` - Полное руководство по интерцепторам Hibernate

### Связанные разделы

- [Entity Callbacks](../Entity Callbacks/README.md)
- [Listener Callbacks](../Listener Callbacks/README.md)
- [Hibernate Event Listeners](../Hibernate Event Listeners/README.md)

## Статус разработки

✅ **Завершено:**
- Структура раздела создана
- Планирование содержания

🔄 **В процессе:**
- Создание файлов с подробным содержанием
- Интеграция с существующими материалами

📋 **Планируется:**
- Подробные руководства по каждому разделу
- Практические примеры
- Интеграция с другими модулями 