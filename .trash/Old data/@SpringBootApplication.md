2022-10-04 23:28
Tags: #spring #springboot #dmdev 
_
# @SpringBootApplication

Конфигурация main класса. Должна лежать в root пакете.


На самом деле в spring-boot main классе всё просто, он точно также создаёт контекст, просто через метод-обёртку:
``` Java
@SpringBootApplication  
public class ApplicationRunner {  
    public static void main(String[] args) {  
        ConfigurableApplicationContext context =  
                SpringApplication.run(ApplicationRunner.class, args);  
    }  
}
```

автоматически подгружаются properties файлы с названием *application*

