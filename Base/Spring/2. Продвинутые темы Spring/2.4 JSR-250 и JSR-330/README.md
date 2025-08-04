# 2.4 JSR-250 и JSR-330

Этот раздел содержит документацию по стандартам Java для dependency injection, которые поддерживаются Spring Framework.

## Содержание раздела

### Основные темы

1. **[JSR-250 и JSR-330.md](./JSR-250 и JSR-330.md)** - Стандарты Java для DI
   - JSR-250 аннотации (@PostConstruct, @PreDestroy, @Resource)
   - JSR-330 аннотации (@Inject, @Named, @Qualifier)
   - Сравнение с Spring аннотациями
   - Практические примеры

### Ключевые концепции

- **JSR-250** - Common Annotations для Java Platform
- **JSR-330** - Dependency Injection для Java
- **@PostConstruct/@PreDestroy** - жизненный цикл бинов
- **@Inject/@Named** - внедрение зависимостей
- **@Resource** - внедрение по имени

### Файлы для изучения

- `JSR-250 и JSR-330.md` - Стандарты Java для DI

### Связанные разделы

- [2.3 SpEL и Environment](../2.3 SpEL и Environment/README.md)
- [Dependency Injection](../../1. Основы Spring/1.1 IoC и Dependency Injection/README.md)
- [BeanPostProcessor](../2.1 BeanPostProcessor/README.md)

## Статус разработки

✅ **Завершено:**
- Структура раздела создана
- JSR-250 и JSR-330.md - Стандарты Java для DI

🔄 **В процессе:**
- Интеграция с другими модулями

📋 **Планируется:**
- Дополнительные примеры
- Интеграция с Spring Security
- Создание тестов и упражнений 