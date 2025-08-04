# Interceptors

Этот раздел содержит документацию по перехватчикам HTTP запросов в Spring Web Framework.

## Содержание раздела

### Основные темы

1. **[Interceptors.md](./Interceptors.md)** - Перехватчики HTTP запросов в Spring
   - HandlerInterceptor
   - WebMvcConfigurer
   - Жизненный цикл интерцептора
   - Практические примеры

### Ключевые концепции

- **HandlerInterceptor** - интерфейс для перехвата HTTP запросов
- **preHandle** - выполнение до обработки контроллера
- **postHandle** - выполнение после обработки контроллера
- **afterCompletion** - выполнение после завершения запроса
- **WebMvcConfigurer** - конфигурация интерцепторов

### Файлы для изучения

- `Interceptors.md` - Полное руководство по интерцепторам Spring

### Связанные разделы

- [Stateful и Stateless](../Stateful и Stateless/README.md)
- [Spring Security](../../3. Spring Security/README.md)
- [Spring MVC](../Spring MVC/README.md)

## Статус разработки

✅ **Завершено:**
- Структура раздела создана
- Interceptors.md - Перехватчики HTTP запросов

🔄 **В процессе:**
- Интеграция с другими модулями

📋 **Планируется:**
- Дополнительные примеры
- Интеграция с Spring Security
- Создание тестов и упражнений 