2022-09-0522:27
Tags: #spring #springbase #dmdev #springXML 
_
# Injections from properties files

Для того чтобы настроить spring для работы с property файлом есть 2 пути: 
1. Прописать отдельный бин 
2. Указать дополнительный namespace

## Использование бина placeholder
Для того чтобы мы могли подтягивать property из файла, необходимо добавить бин, который будет этим заниматься: 

```XML
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
	<property name="locations" value="classpath:application.properties"/>
</bean>
 ```
 также ему необходимо указать *locations* для того чтобы он нашёл properties файл


## Использование namespace

нужно добавить namespace *context* и *context/spring-context-4.0* 

теперь locations можно указать в *context:property-placeholder*
```XML
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
				http://www.springframework.org/schema/beans/spring-beans.xsd
				http://www.springframework.org/schema/context
				http://www.springframework.org/schema/context/spring-context-4.0.xsd">

<context:property-placeholder location="classpath:application.properties"/>

...

</beans>
```


## Внедрение property

Можно использовать **EL** - expression language: ${propertyName}

```XML
<constructor-arg name="username" type="java.lang.String" value="${db.username}"/>
```

Но этого может показаться мало, поэтому спринг предоставляет **SPEL** - который позволяет вставлять целые куски кода, и обращаться сразу через бины: 

```XML
<property name="properties">
	<map>
		<entry key="url" value="postgresurl"/>
		<entry key="password" value="123"/>
		<entry key="driver" value="#{driver.substring(3)}"/>
		<entry key="test" value="#{driver.length() > 10}"/>
		<entry key="test1" value="#{driver.length() > T(Math).random() * 10}"/>
		<entry key="hosts" value="#{'${db.hosts}'.split(',')}"/>
		<entry key="currentUser" value="#{systemProperties['user.name']}"/>
		<entry key="currentUser" value="${user.name}"/>
	</map>
</property>
```