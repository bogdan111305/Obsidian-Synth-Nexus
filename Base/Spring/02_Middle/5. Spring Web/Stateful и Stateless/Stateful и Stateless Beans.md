# Stateful и Stateless Beans

В Spring Framework и веб-разработке в целом важным концептуальным различием является подход к управлению состоянием приложения. Stateful и Stateless подходы имеют различные характеристики, преимущества и недостатки.

## Содержание

1. [Основные концепции](#основые-концепции)
2. [Stateful подход](#stateful-подход)
3. [Stateless подход](#stateless-подход)
4. [Сравнение подходов](#сравнение-подходов)
5. [Практические примеры](#практические-примеры)
6. [Лучшие практики](#лучшие-практики)

## Основные концепции

### Что такое Stateful и Stateless?

- **Stateful** - сервер сохраняет состояние между запросами
- **Stateless** - сервер не сохраняет состояние, вся информация передается клиентом

### Ключевые различия

| Аспект | Stateful | Stateless |
|--------|----------|-----------|
| **Состояние** | Сохраняется на сервере | Передается клиентом |
| **Масштабируемость** | Ограничена | Высокая |
| **Сложность** | Проще для разработки | Сложнее для разработки |
| **Производительность** | Быстрее (нет повторной аутентификации) | Медленнее (повторная валидация) |
| **Отказоустойчивость** | Проблемы при сбоях | Высокая |

## Stateful подход

### Характеристики Stateful

- **Сессии** - информация хранится в сессии на сервере
- **Память** - состояние потребляет память сервера
- **Связность** - клиент привязан к конкретному серверу
- **Простота** - легче разрабатывать и отлаживать

### Примеры Stateful подходов

#### 1. Session-based аутентификация

```java
@RestController
public class StatefulAuthController {
    
    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request, 
                                      HttpSession session) {
        
        // Проверка учетных данных
        User user = userService.authenticate(request.getUsername(), request.getPassword());
        
        if (user != null) {
            // Сохранение в сессии
            session.setAttribute("currentUser", user);
            session.setAttribute("loginTime", System.currentTimeMillis());
            
            return ResponseEntity.ok("Login successful");
        } else {
            return ResponseEntity.status(401).body("Invalid credentials");
        }
    }
    
    @GetMapping("/profile")
    public ResponseEntity<User> getProfile(HttpSession session) {
        // Получение из сессии
        User user = (User) session.getAttribute("currentUser");
        
        if (user != null) {
            return ResponseEntity.ok(user);
        } else {
            return ResponseEntity.status(401).body(null);
        }
    }
    
    @PostMapping("/logout")
    public ResponseEntity<String> logout(HttpSession session) {
        // Очистка сессии
        session.invalidate();
        return ResponseEntity.ok("Logout successful");
    }
}
```

#### 2. Stateful сервис

```java
@Component
@Scope("session") // Stateful scope
public class UserSessionService {
    
    private User currentUser;
    private List<String> recentActions = new ArrayList<>();
    private Map<String, Object> sessionData = new HashMap<>();
    
    public void setCurrentUser(User user) {
        this.currentUser = user;
    }
    
    public User getCurrentUser() {
        return currentUser;
    }
    
    public void addAction(String action) {
        recentActions.add(action);
        // Ограничиваем размер списка
        if (recentActions.size() > 10) {
            recentActions.remove(0);
        }
    }
    
    public List<String> getRecentActions() {
        return new ArrayList<>(recentActions);
    }
    
    public void setSessionData(String key, Object value) {
        sessionData.put(key, value);
    }
    
    public Object getSessionData(String key) {
        return sessionData.get(key);
    }
}
```

#### 3. Stateful кэширование

```java
@Component
@Scope("session")
public class UserCacheService {
    
    private Map<String, Object> cache = new ConcurrentHashMap<>();
    private long lastAccessTime;
    
    public void put(String key, Object value) {
        cache.put(key, value);
        lastAccessTime = System.currentTimeMillis();
    }
    
    public Object get(String key) {
        lastAccessTime = System.currentTimeMillis();
        return cache.get(key);
    }
    
    public void clear() {
        cache.clear();
    }
    
    public boolean isExpired(long timeoutMs) {
        return System.currentTimeMillis() - lastAccessTime > timeoutMs;
    }
}
```

### Проблемы Stateful подхода

#### 1. Масштабируемость

```java
// Проблема: при добавлении нового сервера сессии теряются
@Configuration
public class StatefulConfig {
    
    @Bean
    @Scope("session")
    public UserSessionService userSessionService() {
        // Этот бин будет создан для каждой сессии
        // При масштабировании сессии не синхронизируются между серверами
        return new UserSessionService();
    }
}
```

#### 2. Память

```java
@Component
public class MemoryMonitor {
    
    @EventListener
    public void monitorMemory(ApplicationEvent event) {
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        long maxMemory = runtime.maxMemory();
        
        double memoryUsage = (double) usedMemory / maxMemory * 100;
        
        if (memoryUsage > 80) {
            // Предупреждение о высоком использовании памяти
            System.err.println("High memory usage: " + memoryUsage + "%");
        }
    }
}
```

## Stateless подход

### Характеристики Stateless

- **Токены** - вся информация передается в токенах
- **Без состояния** - сервер не хранит состояние между запросами
- **Масштабируемость** - легко добавлять новые серверы
- **Сложность** - требует тщательного проектирования токенов

### Примеры Stateless подходов

#### 1. JWT-based аутентификация

```java
@Component
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.expiration}")
    private long jwtExpiration;
    
    public String generateToken(User user) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpiration);
        
        return Jwts.builder()
                .setSubject(user.getUsername())
                .claim("userId", user.getId())
                .claim("roles", user.getRoles())
                .setIssuedAt(now)
                .setExpiration(expiryDate)
                .signWith(SignatureAlgorithm.HS512, jwtSecret)
                .compact();
    }
    
    public Claims getClaimsFromToken(String token) {
        return Jwts.parser()
                .setSigningKey(jwtSecret)
                .parseClaimsJws(token)
                .getBody();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}

@RestController
public class StatelessAuthController {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        
        User user = userService.authenticate(request.getUsername(), request.getPassword());
        
        if (user != null) {
            String token = tokenProvider.generateToken(user);
            
            return ResponseEntity.ok(new AuthResponse(token, user));
        } else {
            return ResponseEntity.status(401).body(null);
        }
    }
    
    @GetMapping("/profile")
    public ResponseEntity<User> getProfile(@RequestHeader("Authorization") String token) {
        
        if (token != null && token.startsWith("Bearer ")) {
            String jwt = token.substring(7);
            
            if (tokenProvider.validateToken(jwt)) {
                Claims claims = tokenProvider.getClaimsFromToken(jwt);
                Long userId = claims.get("userId", Long.class);
                
                User user = userService.findById(userId);
                return ResponseEntity.ok(user);
            }
        }
        
        return ResponseEntity.status(401).body(null);
    }
}
```

#### 2. Stateless сервис

```java
@Component
public class StatelessUserService {
    
    public UserInfo getUserInfo(String token) {
        // Извлекаем информацию из токена
        Claims claims = tokenProvider.getClaimsFromToken(token);
        
        return UserInfo.builder()
                .userId(claims.get("userId", Long.class))
                .username(claims.getSubject())
                .roles(claims.get("roles", List.class))
                .build();
    }
    
    public boolean hasPermission(String token, String permission) {
        UserInfo userInfo = getUserInfo(token);
        return userInfo.getRoles().contains(permission);
    }
}
```

#### 3. Stateless кэширование

```java
@Component
public class StatelessCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void put(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }
    
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }
    
    public void delete(String key) {
        redisTemplate.delete(key);
    }
    
    public boolean exists(String key) {
        return redisTemplate.hasKey(key);
    }
}
```

### Преимущества Stateless подхода

#### 1. Масштабируемость

```java
@Configuration
public class StatelessConfig {
    
    @Bean
    public LoadBalancer loadBalancer() {
        // Легко добавлять новые серверы
        return LoadBalancer.builder()
                .addServer("server1:8080")
                .addServer("server2:8080")
                .addServer("server3:8080")
                .build();
    }
}
```

#### 2. Отказоустойчивость

```java
@Component
public class FaultTolerantService {
    
    @Autowired
    private List<ServerInstance> servers;
    
    public ResponseEntity<String> processRequest(String token, HttpRequest request) {
        
        // Пробуем разные серверы при сбое
        for (ServerInstance server : servers) {
            try {
                return server.process(request);
            } catch (Exception e) {
                // Логируем ошибку и пробуем следующий сервер
                logger.error("Server {} failed: {}", server.getId(), e.getMessage());
            }
        }
        
        return ResponseEntity.status(503).body("All servers unavailable");
    }
}
```

## Сравнение подходов

### Таблица сравнения

| Критерий | Stateful | Stateless |
|----------|----------|-----------|
| **Масштабируемость** | ❌ Плохая | ✅ Отличная |
| **Производительность** | ✅ Быстрая | ⚠️ Средняя |
| **Сложность разработки** | ✅ Простая | ❌ Сложная |
| **Отказоустойчивость** | ❌ Плохая | ✅ Отличная |
| **Потребление памяти** | ❌ Высокое | ✅ Низкое |
| **Безопасность** | ⚠️ Средняя | ✅ Высокая |

### Выбор подхода

```java
// Stateful подходит для:
// - Простых приложений
// - Внутренних систем
// - Когда важна простота разработки

// Stateless подходит для:
// - Масштабируемых приложений
// - Микросервисной архитектуры
// - Когда важна надежность
```

## Практические примеры

### 1. Гибридный подход

```java
@Component
public class HybridAuthService {
    
    @Autowired
    private JwtTokenProvider jwtProvider;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public AuthResponse authenticate(LoginRequest request) {
        User user = userService.authenticate(request.getUsername(), request.getPassword());
        
        if (user != null) {
            // Создаем JWT токен (stateless)
            String token = jwtProvider.generateToken(user);
            
            // Сохраняем дополнительную информацию в Redis (stateful)
            String sessionKey = "session:" + user.getId();
            SessionData sessionData = new SessionData(user, System.currentTimeMillis());
            redisTemplate.opsForValue().set(sessionKey, sessionData, Duration.ofHours(24));
            
            return new AuthResponse(token, user);
        }
        
        return null;
    }
    
    public boolean validateSession(String token) {
        if (jwtProvider.validateToken(token)) {
            Claims claims = jwtProvider.getClaimsFromToken(token);
            Long userId = claims.get("userId", Long.class);
            
            // Проверяем дополнительную информацию в Redis
            String sessionKey = "session:" + userId;
            SessionData sessionData = (SessionData) redisTemplate.opsForValue().get(sessionKey);
            
            return sessionData != null && !sessionData.isExpired();
        }
        
        return false;
    }
}
```

### 2. Миграция с Stateful на Stateless

```java
// Старый Stateful подход
@Component
@Scope("session")
public class OldStatefulService {
    
    private User currentUser;
    private List<String> userActions = new ArrayList<>();
    
    public void setUser(User user) {
        this.currentUser = user;
    }
    
    public User getCurrentUser() {
        return currentUser;
    }
}

// Новый Stateless подход
@Component
public class NewStatelessService {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public User getCurrentUser(String token) {
        Claims claims = tokenProvider.getClaimsFromToken(token);
        Long userId = claims.get("userId", Long.class);
        
        // Получаем из Redis
        String userKey = "user:" + userId;
        return (User) redisTemplate.opsForValue().get(userKey);
    }
    
    public void addUserAction(String token, String action) {
        Claims claims = tokenProvider.getClaimsFromToken(token);
        Long userId = claims.get("userId", Long.class);
        
        // Сохраняем в Redis
        String actionsKey = "actions:" + userId;
        redisTemplate.opsForList().rightPush(actionsKey, action);
        redisTemplate.expire(actionsKey, Duration.ofHours(24));
    }
}
```

### 3. Мониторинг состояния

```java
@Component
public class StateMonitor {
    
    @Autowired
    private ApplicationContext applicationContext;
    
    @Scheduled(fixedRate = 60000) // Каждую минуту
    public void monitorState() {
        
        // Мониторинг сессий
        if (applicationContext.containsBean("sessionRegistry")) {
            SessionRegistry sessionRegistry = applicationContext.getBean(SessionRegistry.class);
            int activeSessions = sessionRegistry.getAllPrincipals().size();
            
            logger.info("Active sessions: {}", activeSessions);
            
            if (activeSessions > 1000) {
                logger.warn("High number of active sessions: {}", activeSessions);
            }
        }
        
        // Мониторинг памяти
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        long maxMemory = runtime.maxMemory();
        
        double memoryUsage = (double) usedMemory / maxMemory * 100;
        logger.info("Memory usage: {}%", String.format("%.2f", memoryUsage));
    }
}
```

## Лучшие практики

### 1. Выбор подхода

```java
// Используйте Stateful для:
// - Простых веб-приложений
// - Внутренних административных панелей
// - Прототипов и MVP

// Используйте Stateless для:
// - REST API
// - Микросервисов
// - Высоконагруженных приложений
// - Облачных решений
```

### 2. Безопасность

```java
@Component
public class SecurityBestPractices {
    
    // Для Stateful
    public void secureSession(HttpSession session) {
        // Устанавливаем безопасные атрибуты сессии
        session.setMaxInactiveInterval(1800); // 30 минут
        session.setAttribute("lastAccess", System.currentTimeMillis());
    }
    
    // Для Stateless
    public void secureToken(String token) {
        // Проверяем подпись и срок действия
        if (!tokenProvider.validateToken(token)) {
            throw new SecurityException("Invalid token");
        }
        
        Claims claims = tokenProvider.getClaimsFromToken(token);
        if (claims.getExpiration().before(new Date())) {
            throw new SecurityException("Token expired");
        }
    }
}
```

### 3. Производительность

```java
@Component
public class PerformanceOptimization {
    
    // Для Stateful - используем кэширование
    @Cacheable("userSessions")
    public UserSession getUserSession(String sessionId) {
        return sessionRepository.findById(sessionId);
    }
    
    // Для Stateless - используем Redis
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void cacheUserData(String userId, UserData data) {
        redisTemplate.opsForValue().set("user:" + userId, data, Duration.ofMinutes(30));
    }
}
```

### 4. Тестирование

```java
@SpringBootTest
class StatefulStatelessTest {
    
    @Test
    void testStatefulSession() {
        // Тест Stateful подхода
        HttpSession session = mock(HttpSession.class);
        when(session.getAttribute("user")).thenReturn(new User("test"));
        
        StatefulService service = new StatefulService();
        service.setSession(session);
        
        assertThat(service.getCurrentUser()).isNotNull();
    }
    
    @Test
    void testStatelessToken() {
        // Тест Stateless подхода
        String token = tokenProvider.generateToken(new User("test"));
        
        StatelessService service = new StatelessService();
        User user = service.getCurrentUser(token);
        
        assertThat(user).isNotNull();
        assertThat(user.getUsername()).isEqualTo("test");
    }
}
```

## Связанные темы

- [Interceptors](../Interceptors/README.md)
- [Spring Security](../../3. Spring Security/README.md)
- [REST API](../REST API/README.md) 