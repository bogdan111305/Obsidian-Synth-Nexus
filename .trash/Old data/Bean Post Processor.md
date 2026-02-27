2022-09-06 21:52
Tags: #spring #springbase #dmdev #beanLifecycle 
_
# Bean Post Processor

Имеет 2 основных метода: 
```Java
public interface BeanPostProcessor {  
	@Nullable  
   default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
      return bean;  //обрабатывает bean перед init методом
   }  
  
    @Nullable  
   default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
      return bean;  //обрабатывает bean после init метода
   }   
}
```

Небольшой нюанс, в первом методе мы можем увидеть и поработать с оригинальным бином, однако во втором он может быть запроксирован.

И имя бина никогда не меняется. 1 

### Aware 

они нужны для того чтобы внедрять какие-то специфичные бины типа ApplicationContext или BeanFactory - это позволяет взаимодейстовать с этими бинами

Внедрением всех этих бинов в реализации Aware интерфейсов занимается специальный BPP: *ApplicationContextAwareProcessor*


как мы видим его метод BeforeInitialization отрабатывает только для бинов реализующих Aware интерфейсы:
```Java
@Override  
@Nullable  
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
   if (!(bean instanceof EnvironmentAware ||
		 bean instanceof EmbeddedValueResolverAware ||  
         bean instanceof ResourceLoaderAware ||
         bean instanceof ApplicationEventPublisherAware ||  
         bean instanceof MessageSourceAware || 
         bean instanceof ApplicationContextAware ||  
         bean instanceof ApplicationStartupAware)) {  
      return bean;  
   }
```
