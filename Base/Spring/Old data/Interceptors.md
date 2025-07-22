2022-10-09 23:24
Tags: #hibernate #auditing #dmdev 
_
# Hibernate Interceptors

Этот механизм чуть сложнее, и из-за своей монструозности используется реже чем listener

Однако он позволяет создать некое поведение не привязанное даже к event-ам. На всё приложение.

В самом интефейсе Interceptor огромное количество методов и для упращения написали дефолтную реализацию *EmptyInterceptor* (deprecated с 6 версии)

```Java 
public class GlobalInterceptor extends EmptyInterceptor {  
  
    @Override  
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState, Object[] previousState, String[] propertyNames, Type[] types) throws CallbackException {  
        return super.onFlushDirty(entity, id, currentState, previousState, propertyNames, types);  
    }  
}
```

## Регистрация Interceptor-а

Глобальный: 
``` Java
Configuration configuration = getConfiguration();  
configuration.setInterceptor(new GlobalInterceptor());
```

В некотоых случаях можно добавить interceptor для конкретной сессии: 

``` Java
session  
        .sessionWithOptions()  
        .interceptor(new GlobalInterceptor());
```
 