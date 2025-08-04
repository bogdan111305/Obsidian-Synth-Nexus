# Event Listeners and Callbacks

Этот раздел содержит документацию по различным механизмам перехвата событий и обратных вызовов в Hibernate.

## Содержание раздела

### Основные темы

1. **[Entity Callbacks](./Entity Callbacks/README.md)** - Callbacks на уровне сущностей
   - @PostLoad, @PrePersist, @PostPersist
   - @PreUpdate, @PostUpdate, @PreRemove, @PostRemove
   - Интерфейс Callback
   - Класс EntityCallback

2. **[Listener Callbacks](./Listener Callbacks/README.md)** - Callbacks через слушатели
   - Класс ListenerCallback
   - Регистрация ListenerCallback
   - Практические примеры

3. **[Hibernate Event Listeners](./Hibernate Event Listeners/README.md)** - Слушатели событий Hibernate
   - Hibernate Event model
   - Типы событий
   - Регистрация слушателей

4. **[Hibernate Interceptors](./Hibernate Interceptors/README.md)** - Перехватчики Hibernate
   - Интерфейс Interceptor
   - EmptyInterceptor (deprecated)
   - Регистрация интерцепторов
   - Практические примеры

### Ключевые концепции

- **Entity Callbacks** - аннотации для жизненного цикла сущностей
- **Listener Callbacks** - программные callbacks через интерфейсы
- **Event Listeners** - слушатели событий Hibernate
- **Interceptors** - низкоуровневые перехватчики операций
- **Жизненный цикл** - этапы существования сущности

### Файлы для изучения

- `Entity Callbacks/` - Callbacks на уровне сущностей
- `Listener Callbacks/` - Callbacks через слушатели
- `Hibernate Event Listeners/` - Слушатели событий Hibernate
- `Hibernate Interceptors/` - Перехватчики Hibernate

### Связанные разделы

- [Entity Associations](../Entity Associations/README.md)
- [Transactions & Locks](../Transactions & Locks/README.md)
- [N + 1 выборок в Hibernate](../N + 1 выборок в Hibernate/README.md)

## Статус разработки

✅ **Завершено:**
- Структура раздела создана
- Entity Callbacks - Callbacks на уровне сущностей
- Listener Callbacks - Callbacks через слушатели
- Hibernate Event Listeners - Слушатели событий Hibernate
- Hibernate Interceptors - Перехватчики Hibernate

🔄 **В процессе:**
- Интеграция с другими модулями

📋 **Планируется:**
- Дополнительные примеры
- Интеграция с Spring
- Создание тестов и упражнений 