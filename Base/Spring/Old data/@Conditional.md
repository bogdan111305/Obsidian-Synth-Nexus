Tags: #spring #springboot #dmdev 
# @Conditional

``` Java
@Target({ElementType.TYPE, ElementType.METHOD})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Conditional {  
  
	Class<? extends Condition>[] value();  
  
}
```

Как мы видим, аннотацию можно ставить над типом или методом.

И на самом деле под методом имеется ввиду метод помеченный аннотацией *@Bean*

<i style="color:yellow;" >То есть с помощью этой аннотации мы можем определить добавлять бин в контекст или нет
</i>

Как мы видим в качестве value аннотация ожидает некий *condition*

## Condition

Condition - это всего лишь функциональный интерфейс, единственный метод которого возвращает true или false

``` Java
@FunctionalInterface  
public interface Condition {  
 boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);  
}
```

Аргументы: 
- CondirionContext - контекст, с помощью которого можно дотянуться до: 
	- BeanDefinitionRegistry - позволит получить доступ, ко всем BeanDefinitions, которые нам нужны
	- BeanFactory - любые готовы бины
	- Environment
	- ResourceLoader - любой файл в classPath
	- ClassLoader
- Metadata - метаинформация об аннотациях, которые стоят над классом или методом

### Пример использования 

Например мы создаём конфигурацию, которая должна отрабатывать, только если в ClassPath есть postgres драйвер: 

``` Java
@Conditional(JpaCondition.class)  
@Configuration  
public class JpaConfiguration {  
 //some logic
}
```

Если мы хотим использовать JpaCondition, то он должен реализовывать Codition интерфейс: 

``` Java
public class JpaCondition implements Condition {  
    @Override  
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {  
        ClassLoader classLoader = context.getClassLoader();  
        try {  
            classLoader.loadClass("org.postgresql.Driver");  
            return true;        
        } catch (ClassNotFoundException e) {  
            return false;  
        }  
    }  
}
```

Кстати говоря, @Profiles на самом деле тоже работает через механизм @Conditional