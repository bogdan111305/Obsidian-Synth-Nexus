2022-09-05 22:48
Tags: #spring #springbase #dmdev #beanLifecycle
_
# Bean Factory Post Processor


> BeanFactory - это по сути *ядро логики DI* в spring. 
> Главный трудяга, он - тот, кто собирает обрабатывает bean definitions, pojo, вызывает BPP, если они есть и нужны, и затем кладёт всё в container бинов, откуда затем мы можем наши бины доставать! 

\-  из доклада Евгения Борисова


### Интерфейс 

Это **функциональный интерфейс** с одним методом, который принимает на вход beanFactory: 

```java
@FunctionalInterface  
public interface BeanFactoryPostProcessor {  
      void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory);    
}
``` 

отрабатывает перед созданием бинов 

![[Pasted image 20220905230635.png]]

т.е. если мы подключаем некий BeanFactoryPostProcessor его задачей будет сделать что-то с bean definitions прежде чем они начнут инициализироваться.

Например, если мы используем Spel и вставляем property, то нужен кто-то, кто обработает Spel выражение и подянет необъодимые property - этим и может занятся BFPP

### Откуда он берётся? 

На самом деле это обычный bean, который иницилизируется раньше отслальных. Bean Factory понимает, что его надо проинциализировать, когда видит что он реализует интерфейс **BeanFactoryPostProcessor**

### Как Spring читает интерфейсы классов

на самом деле spring использует обычный **java reflection** и его метод *isAssinableFrom()*

### Порядок выполнения BFPP

spring позволяет задавать приоритет для бинов, которые находятся в одном логическом списке

Для того чтобы определить порядок, выполнения нашего бина в очереди, он должен реализовать интерфейс *Ordered*
``` java
public class VerifyPropertyBeanFactoryPostProcessor implements BeanFactoryPostProcessor, Ordered {  
  
    @Override  
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {  
        System.out.println();  
    }  
  
    @Override  
    public int getOrder() {  
        return Ordered.HIGHEST_PRECEDENCE;  //наивысший приоритет  
    }  
}
```

Ещё: интерфейс *PriorityOrderd* наследует *Ordered*, однако бины которые его реализовали всегда будут приоритетнее тех, кто реализовал просто Orderd