2022-10-02 20:08
Tags: #spring #springbase #dmdev 
_
# Event Listeners

Общая схема паттерна Listener:

![[Pasted image 20221002201800.png]]

как видно из схемы, event как правило должен иметь некий тип event'a и объект, который обрабатывается.

## В Spring

Создание event: 

``` Java
public class EntityEvent extends ApplicationEvent {  
  
    private final AccessType type;  
  
    public EntityEvent(Object source, AccessType type, Object entity) {  
        super(source);  //source - entity объект, операции над которым прослушиваются  
        this.type = type;  
    }  
  
    public AccessType getType() {  
        return type;  
    }  
}
```

Event должен наследовать ApplicationEvent, в базовый конструктор которого будет передаваться source объект.

Создание слушателя: 

``` Java 
@Component  
public class EntityListener {  
    @EventListener //будет обрабатываться EventListenerMethodProcessor
    @Order(10) //задать порядок этого слушателя в списке слушателей
    public void acceptEntity(EntityEvent event){  
        System.out.println("Entity: " + event);  
    }  
}
```

аннотацию *@EventListener* будет обрабатывать **EventListenerMetthodProcerssor** (BFPP) представленный в [[Annotation-config]]

### Отправка event

Чтобы функционал работал, нам необходимо написать код, котрый будет отправлять событие в listener.

За отправку события отвечает некоторый объект - publisher. В spring это **ApplicationEventPublisher.** 

ApplicationContextAwareProcessor [[Bean Post Processor#Aware]] может нам его подтянуть.

Мы бы могли реализовать соответствующий интефейс (ApplicationEventPublisherAware) и сделать setter, но в таком случае наш сервис станет изменяемым обхектом - что не есть хорошо.

Вместо этого мы можем просто попросить его у контекста. 
И на самом деле applicationContext реализует этот интерфейс.

``` Java
@Service  
public class MyService {  
    private final ApplicationEventPublisher publisher;  
  
    @Autowired  
    public MyService(ApplicationEventPublisher publisher) {  
        this.publisher = publisher;  
    }  
  
    public Optional<String> readMeth(){  
        String myEntity = "myEntity";  
        publisher.publishEvent(new EntityEvent(myEntity, AccessType.READ));  
        return Optional.of(myEntity);  
    }  
  
}
```

### Ещё про аннотацию @EventListener

Кроме прочего в аннотации можно указывать condition - условие, при котором будет событие будет обрабатываться. 
Главное чтобы условие вовращало true, false, 0 или 1. 

\#root.event - event объект
\#root.args - аргументы слушателя

```Java
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface EventListener {
	...
	String condition() default "";
	...
}
```

Пример: 

``` Java 
@EventListener(condition = "#root.args[0].acceptType.name == 'READ'")  
public void acceptEntity(EntityEvent event){  
    System.out.println("Entity: " + event);  
}
```