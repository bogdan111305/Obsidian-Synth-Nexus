2022-10-09 22:59
Tags: #hibernate #auditing #dmdev 
_
# Listeneres

Отличие этого listener-а, от построенного на механизме Callback в том что он привязан не к entity а конкретному соббытию или event-у  

Общая схемма паттерна: 
![[Pasted image 20221009230945.png]]

На самом деле, большинство вещей в hibernate подстроено на шблоне проектирования Listeners. Даже обычный метод сессии *save()*

На фоне entity и события создаётся event, и оповещаются все прослушиывающие listener-ы 
``` Java  
public void persist(Object entity) {  
   checkOpen();  
   firePersist( new PersistEvent( null, object, this ) );  
}

private void firePersist(final PersistEvent event) {  
    fastSessionServices.eventListenerGroup_PERSIST  
        .fireEventOnEachListener( event, PersistEventListener::onPersist );
```

Дело в том, что на каждый из случаев жизни у hibernate подготовлены группы Listener-ов.

Следовательно всё что нам нужно, это: 
- написать listener на необходимое нам соббытие 
- добавить/зарегестрировать мой listener в группе listener-ов


Сами по себе listener-ы это всего лишь функциональные интерфейсы, которые принимают некоторые event-ы: 

например preUpdateEventListener: 
``` Java
public interface PreUpdateEventListener {  
      boolean onPreUpdate(PreUpdateEvent event);  
}
```


Для примера создадим listener, который будет сохранят Audit entity в таблицу Aaudit при изменении или удалении: 
## Создание Listener
``` Java
public class AuditTableListener implements PreDeleteEventListener, PreInsertEventListener {  
  
    @Override  
    public boolean onPreDelete(PreDeleteEvent event) {  
        audit(event, Audit.Operation.DELETE);  
        return false;    
    }  
    @Override  
    public boolean onPreInsert(PreInsertEvent event) {  
        audit(event, Audit.Operation.INSERT);  
        return false;    
    }  
  
    private static void audit(AbstractPreDatabaseOperationEvent event, Audit.Operation operation) {  
        if(event.getEntity().getClass() != Audit.class){  
            Audit audit = Audit.builder()  
                    .entityId((Serializable) event.getId())  
                    .entityName(event.getEntity().getClass().getName())  
                    .entityContent(event.getEntity().toString())  
                    .operation(operation)  
                    .build();  
  
            event.getSession().persist(audit);  
        }  
    }  
}
```

## Регистрация Listener:

Так как listeners должны быть с самого начала необходимо добавлять их с помощью sessionFacroty

``` Java
private static void registerListener(SessionFactory sessionFactory) {  
    SessionFactoryImpl unwrap = sessionFactory.unwrap(SessionFactoryImpl.class);  
    EventListenerRegistry listenerRegistry = unwrap.getServiceRegistry().getService(EventListenerRegistry.class);  
  
    AuditTableListener listener = new AuditTableListener();  
    listenerRegistry.appendListeners(EventType.PRE_INSERT, listener);  
    listenerRegistry.appendListeners(EventType.PRE_DELETE, listener);  
}
```
 