2022-10-10 21:34
Tags: #spring #springboot #dmdev 
_
# Logger starter


 ![[Pasted image 20221010214846.png]]


в spring используется sl4j api

стартер для логгера: 
`spring-boot-starter-logging`

``` Java
@Slf4j  
public class SomeService {  
    public void someMeth(){  
        log.info("someMeth");  
    }  
}
```

Некоторые конфигурации логера: 

``` yaml
logging:  
  charset:  
    console: utf-8  
#будет выводить у указанных логгеров указанные уровни и выше по опасности  
  level:  
    root: WARN  
    org.motorin.spring: DEBUG  
    org.motorin.spring.service.SomeService: INFO
# также можно указывать паттерны форматирования
  pattern:  
	console:
# по умолчанию логи идут только в консоль, добавляем создание файла
  file:  
	name: motorin.log  
	path: /
```

## Custom Log Configuration

Для того чтобы полностью создать свою конфигурацию и она перекрывала дефолтную конфигурацию спринга.

Нужно создать файл в зависимости от типа нашего логирования:
![[Pasted image 20221010221122.png]]

[Документация](https://docs.spring.io/spring-boot/docs/2.1.18.RELEASE/reference/html/boot-features-logging.html)
[Конфигурация Logback](https://logback.qos.ch/manual/configuration.html)
