# Spring Interceptors

Spring Interceptors предоставляют мощный механизм для перехвата HTTP запросов на различных этапах их обработки. Это позволяет добавлять дополнительную логику без изменения основного кода контроллеров.

## Содержание

1. [Основы интерцепторов](#основы-интерцепторов)
2. [HandlerInterceptor интерфейс](#handlerinterceptor-интерфейс)
3. [Жизненный цикл интерцептора](#жизненный-цикл-интерцептора)
4. [Конфигурация интерцепторов](#конфигурация-интерцепторов)
5. [Практические примеры](#практические-примеры)
6. [Лучшие практики](#лучшие-практики)

## Основы интерцепторов

### Что такое Spring Interceptors?

Spring Interceptors - это компоненты, которые позволяют перехватывать HTTP запросы на различных этапах их обработки в Spring MVC. Они работают как фильтры, но на уровне Spring MVC, а не на уровне сервлетов.

### Зачем нужны интерцепторы?

- **Логирование** - запись информации о запросах
- **Аутентификация** - проверка авторизации пользователей
- **Аудит** - отслеживание действий пользователей
- **Кэширование** - кэширование ответов
- **Мониторинг** - сбор метрик производительности

## HandlerInterceptor интерфейс

### Основной интерфейс

```java
public interface HandlerInterceptor {
    
    /**
     * Выполняется ДО обработки контроллера
     */
    default boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {
        return true;
    }
    
    /**
     * Выполняется ПОСЛЕ обработки контроллера, но ДО рендеринга view
     */
    default void postHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler, 
                           ModelAndView modelAndView) throws Exception {
    }
    
    /**
     * Выполняется ПОСЛЕ завершения запроса (включая рендеринг view)
     */
    default void afterCompletion(HttpServletRequest request, 
                                HttpServletResponse response, 
                                Object handler, 
                                Exception ex) throws Exception {
    }
}
```

### Возвращаемые значения

- **preHandle()** возвращает `boolean`:
  - `true` - продолжить обработку запроса
  - `false` - прервать обработку запроса

## Жизненный цикл интерцептора

### Последовательность выполнения

```
HTTP Request
    ↓
preHandle() [Interceptor 1]
    ↓
preHandle() [Interceptor 2]
    ↓
Controller Method
    ↓
postHandle() [Interceptor 2]
    ↓
postHandle() [Interceptor 1]
    ↓
View Rendering
    ↓
afterCompletion() [Interceptor 2]
    ↓
afterCompletion() [Interceptor 1]
    ↓
HTTP Response
```

### Пример реализации

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) throws Exception {
        
        long startTime = System.currentTimeMillis();
        request.setAttribute("startTime", startTime);
        
        logger.info("Request URL: {}", request.getRequestURL());
        logger.info("Request Method: {}", request.getMethod());
        logger.info("Handler: {}", handler);
        
        return true; // Продолжаем обработку
    }
    
    @Override
    public void postHandle(HttpServletRequest request, 
                          HttpServletResponse response, 
                          Object handler, 
                          ModelAndView modelAndView) throws Exception {
        
        logger.info("Post Handle - Response Status: {}", response.getStatus());
        
        if (modelAndView != null) {
            logger.info("View Name: {}", modelAndView.getViewName());
        }
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) throws Exception {
        
        long startTime = (Long) request.getAttribute("startTime");
        long endTime = System.currentTimeMillis();
        long duration = endTime - startTime;
        
        logger.info("Request completed in {} ms", duration);
        
        if (ex != null) {
            logger.error("Exception occurred: {}", ex.getMessage());
        }
    }
}
```

## Конфигурация интерцепторов

### WebMvcConfigurer

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Autowired
    private LoggingInterceptor loggingInterceptor;
    
    @Autowired
    private AuthenticationInterceptor authInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Добавляем интерцепторы в определенном порядке
        registry.addInterceptor(loggingInterceptor)
                .addPathPatterns("/**")           // Применяем ко всем путям
                .excludePathPatterns("/static/**", "/error"); // Исключаем статические ресурсы
        
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**", "/admin/**") // Только для API и админки
                .order(1); // Порядок выполнения (меньше = раньше)
    }
}
```

### Path Patterns

```java
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor())
                // Включаем только определенные пути
                .addPathPatterns("/api/**", "/users/**")
                // Исключаем определенные пути
                .excludePathPatterns("/api/public/**", "/users/login")
                // Порядок выполнения
                .order(1);
        
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/admin/**", "/api/secure/**")
                .order(2);
    }
}
```

## Практические примеры

### 1. Интерцептор аутентификации

```java
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {
    
    @Autowired
    private JwtTokenProvider jwtTokenProvider;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) throws Exception {
        
        // Пропускаем OPTIONS запросы (CORS)
        if ("OPTIONS".equals(request.getMethod())) {
            return true;
        }
        
        // Получаем токен из заголовка
        String token = getTokenFromRequest(request);
        
        if (token == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\":\"No token provided\"}");
            return false;
        }
        
        try {
            // Валидируем токен
            if (!jwtTokenProvider.validateToken(token)) {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.getWriter().write("{\"error\":\"Invalid token\"}");
                return false;
            }
            
            // Устанавливаем пользователя в контекст
            String username = jwtTokenProvider.getUsernameFromToken(token);
            request.setAttribute("currentUser", username);
            
            return true;
            
        } catch (Exception e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\":\"Token validation failed\"}");
            return false;
        }
    }
    
    private String getTokenFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### 2. Интерцептор кэширования

```java
@Component
public class CacheInterceptor implements HandlerInterceptor {
    
    @Autowired
    private CacheManager cacheManager;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) throws Exception {
        
        // Проверяем кэш только для GET запросов
        if (!"GET".equals(request.getMethod())) {
            return true;
        }
        
        String cacheKey = generateCacheKey(request);
        Cache cache = cacheManager.getCache("http-responses");
        
        if (cache != null) {
            Cache.ValueWrapper cachedResponse = cache.get(cacheKey);
            if (cachedResponse != null) {
                // Возвращаем кэшированный ответ
                CachedResponse response = (CachedResponse) cachedResponse.get();
                writeCachedResponse(response, response);
                return false; // Прерываем обработку
            }
        }
        
        // Сохраняем ключ для postHandle
        request.setAttribute("cacheKey", cacheKey);
        return true;
    }
    
    @Override
    public void postHandle(HttpServletRequest request, 
                          HttpServletResponse response, 
                          Object handler, 
                          ModelAndView modelAndView) throws Exception {
        
        String cacheKey = (String) request.getAttribute("cacheKey");
        if (cacheKey != null && response.getStatus() == 200) {
            // Кэшируем успешный ответ
            Cache cache = cacheManager.getCache("http-responses");
            if (cache != null) {
                CachedResponse cachedResponse = new CachedResponse(response);
                cache.put(cacheKey, cachedResponse);
            }
        }
    }
    
    private String generateCacheKey(HttpServletRequest request) {
        return request.getRequestURI() + "?" + request.getQueryString();
    }
}
```

### 3. Интерцептор мониторинга

```java
@Component
public class MetricsInterceptor implements HandlerInterceptor {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) throws Exception {
        
        long startTime = System.currentTimeMillis();
        request.setAttribute("startTime", startTime);
        
        // Увеличиваем счетчик запросов
        meterRegistry.counter("http.requests.total", 
                            "method", request.getMethod(),
                            "uri", request.getRequestURI())
                    .increment();
        
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) throws Exception {
        
        long startTime = (Long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - startTime;
        
        // Записываем время выполнения
        meterRegistry.timer("http.request.duration",
                          "method", request.getMethod(),
                          "uri", request.getRequestURI(),
                          "status", String.valueOf(response.getStatus()))
                    .record(duration, TimeUnit.MILLISECONDS);
        
        // Увеличиваем счетчик ошибок
        if (ex != null) {
            meterRegistry.counter("http.errors.total",
                                "method", request.getMethod(),
                                "uri", request.getRequestURI())
                        .increment();
        }
    }
}
```

### 4. Интерцептор аудита

```java
@Component
public class AuditInterceptor implements HandlerInterceptor {
    
    @Autowired
    private AuditService auditService;
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) throws Exception {
        
        // Создаем запись аудита
        AuditEntry audit = AuditEntry.builder()
                .timestamp(Instant.now())
                .method(request.getMethod())
                .uri(request.getRequestURI())
                .status(response.getStatus())
                .userAgent(request.getHeader("User-Agent"))
                .ipAddress(getClientIpAddress(request))
                .user((String) request.getAttribute("currentUser"))
                .exception(ex != null ? ex.getMessage() : null)
                .build();
        
        // Сохраняем асинхронно
        CompletableFuture.runAsync(() -> {
            auditService.save(audit);
        });
    }
    
    private String getClientIpAddress(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

## Лучшие практики

### 1. Правильный порядок интерцепторов

```java
@Configuration
public class InterceptorOrderConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 1. Логирование (самый ранний)
        registry.addInterceptor(new LoggingInterceptor())
                .addPathPatterns("/**")
                .order(1);
        
        // 2. Аутентификация
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**", "/admin/**")
                .order(2);
        
        // 3. Кэширование
        registry.addInterceptor(new CacheInterceptor())
                .addPathPatterns("/api/public/**")
                .order(3);
        
        // 4. Мониторинг (самый поздний)
        registry.addInterceptor(new MetricsInterceptor())
                .addPathPatterns("/**")
                .order(4);
    }
}
```

### 2. Обработка исключений

```java
@Component
public class SafeInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) throws Exception {
        
        try {
            // Логика интерцептора
            return true;
        } catch (Exception e) {
            // Логируем ошибку, но не прерываем обработку запроса
            logger.error("Error in interceptor: {}", e.getMessage(), e);
            return true;
        }
    }
}
```

### 3. Условная логика

```java
@Component
public class ConditionalInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) throws Exception {
        
        // Проверяем условия перед выполнением
        if (shouldSkip(request)) {
            return true;
        }
        
        // Выполняем логику только при необходимости
        performLogic(request);
        return true;
    }
    
    private boolean shouldSkip(HttpServletRequest request) {
        return request.getRequestURI().startsWith("/health") ||
               request.getRequestURI().startsWith("/metrics");
    }
}
```

### 4. Тестирование интерцепторов

```java
@SpringBootTest
class AuthenticationInterceptorTest {
    
    @Autowired
    private WebApplicationContext context;
    
    private MockMvc mockMvc;
    
    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders.webAppContextSetup(context).build();
    }
    
    @Test
    void shouldAllowRequestWithValidToken() throws Exception {
        String validToken = "valid.jwt.token";
        
        mockMvc.perform(get("/api/secure/data")
                .header("Authorization", "Bearer " + validToken))
                .andExpect(status().isOk());
    }
    
    @Test
    void shouldRejectRequestWithoutToken() throws Exception {
        mockMvc.perform(get("/api/secure/data"))
                .andExpect(status().isUnauthorized());
    }
}
```

## Связанные темы

- [Stateful и Stateless](../Stateful и Stateless/README.md)
- [Spring Security](../../3. Spring Security/README.md)
- [Spring MVC](../Spring MVC/README.md) 