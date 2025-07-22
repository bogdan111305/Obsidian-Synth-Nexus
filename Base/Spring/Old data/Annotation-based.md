2022-09-06 21:31
Tags: #spring #springbase #springAnnotationConf 
_
# Annotation based configuration

Если мы комбинируем XML конфигурацию с аннотациями в XML необходимо добавить бин: 

```XML
<bean class="org.springframework.context.annotation.CommonAnnotationBeanPostProcessor"/>
```

или другой способ: 

```XML
<bean: annotation-config/>
```

[[Annotation-config]] добавляет сразу несколько бинов, которые будут обрабатывать остальные основные аннотации, которые будут использоваться в проекте


### @PostConstruct и @PreDestroy

Вместо *init* и *destroy* методов

Так как spring c java 9 начал разбиваться на модули - эти аннотации перекочевали в зависимость: **Jacarta annotation API**

