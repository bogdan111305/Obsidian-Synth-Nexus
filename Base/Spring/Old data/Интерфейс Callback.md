2022-10-09 14:47
Tags: #hibernate #auditing #dmdev  
_
# Интерфейс Callback 

Callbacks нужны для того чтобы перехватывать какие-либо события из жизненного цикла сущности и что-то сделать с самой сущностью.

``` Java
public interface Callback extends Serializable {  
   CallbackType getCallbackType();  
   boolean performCallback(Object entity);  
}
```

CallbackType - тип события которое будет перехватываться. Используются аннотацию - следовательно их использовать можно в двух отдельных местах: 
- самих сущностях
- отдельных листенерах

``` Java
PRE_UPDATE( PreUpdate.class ),  
POST_UPDATE( PostUpdate.class ),  
PRE_PERSIST( PrePersist.class ),  
POST_PERSIST( PostPersist.class ),  
PRE_REMOVE( PreRemove.class ),  
POST_REMOVE( PostRemove.class ),  
POST_LOAD( PostLoad.class )
```


Реализации collbacks создаются и попадают в класс CallbackRegistryImpl: 

``` Java
final class CallbackRegistryImpl implements CallbackRegistryImplementor {  
   private final HashMap<Class<?>, Callback[]> preCreates = new HashMap<>();  
   private final HashMap<Class<?>, Callback[]> postCreates = new HashMap<>();  
   private final HashMap<Class<?>, Callback[]> preRemoves = new HashMap<>();  
   private final HashMap<Class<?>, Callback[]> postRemoves = new HashMap<>();  
   private final HashMap<Class<?>, Callback[]> preUpdates = new HashMap<>();  
   private final HashMap<Class<?>, Callback[]> postUpdates = new HashMap<>();  
   private final HashMap<Class<?>, Callback[]> postLoads = new HashMap<>();
```


Есть 2 варианта использования этого интерфейса: 
- прямо в entity
- в отдельном listener


## Callback в entity

``` Java
@MappedSuperclass  
public abstract class AuditableEntity<T extends Serializable> implements BaseEntity<T> {  
    private Instant createdAt;  
    private String createdBy;  
    
    @PrePersist  
    public void prePersist(){  
        this.setCreatedAt(Instant.now());  
        this.setCreatedBy(SecurityContext.getUser());  
    }  
}
```


Проблема этого подхода в том, что в наших pojo классах приходится прописывать некоторую логику, чот не соответствует определению pojo класса.


## Callback в listener

Накладывает ограничение: мы не можем несколько раз использовать @PrePersist, для подобного поведения придётся создать другой listener.

Когда мы создаём отдельный listener, hibernate не знает о нём.  
Решить эту проблему помогает аннотация <i style="color:#f3ff00">@EntityListener</i> над сущностью которая будет аудироваться:

``` Java
@EntityListeners(AuditListener.class)  
public abstract class AuditableEntity<T extends Serializable> implements BaseEntity<T> {  
    private Instant createdAt;  
    private String createdBy;  
  
    private Instant updatedAt;  
    private String updatedBy;  
}
```

``` JAVA
public class AuditListener {  
    @PrePersist  
    public void prePersist(AuditableEntity<?> entity){  
        entity.setCreatedAt(Instant.now());  
        entity.setCreatedBy("CreateUser");  
    }  
  
    @PreUpdate  
    public void preUpdate(AuditableEntity<?> entity){  
        entity.setUpdatedAt(Instant.now());  
        entity.setUpdatedBy("UpdateUser");  
    }  
}
```