2022-10-04 23:22
Tags: #spring #springboot #dmdev #gradle
_
# Spring Boot Gradle

Для того чтобы tasks и конфигурации spring-boot в gradle необходимо добавить 2 плагина: 

``` Groovy
plugins {  
    id 'java'  
    id 'org.springframework.boot' version '2.6.2'  
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'  
}
```


<a href="https://docs.spring.io/dependency-management-plugin/docs/current/reference/html/#introduction">документация</a>