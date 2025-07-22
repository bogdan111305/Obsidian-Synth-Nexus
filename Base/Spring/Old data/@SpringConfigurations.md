2022-10-05 22:35
Tags: #spring #springboot #dmdev 
_
# @SpringConfigurations

При описании конфигурации чаще всего используются [properties файлы](Properties.md)  которые также могут быть в  [Yaml](Yaml.md)  формате. Что же с ними происходит дальше? 

На самом деле те properties которые там указаны в итоге <i style="color:#f3ff00">становятся значениями для самых обычных бинов</i>, которые как правило являются конфигурационными.

Который должен представлять из себя:
	- pojo класс
	- иметь пустой конструктор
	- сеттеры и геттеры

Представим, что у нас есть следующий yaml файл: 

``` yaml
db:  
  username: ${username.value:postgres}  
  password: pass  
  driver: PostgresDriverUrl:postgres:5432  
  hosts: localhost,127.0.0.1  
  properties:  
    first: 123  
    second: 567  
    third.value: Third  
  pool:  
    size: 12  
    timeout: 10  
  pools:      
    - size: 1  
      timeout: 1  
    - size: 2  
      timeout: 2  
    - size: 3  
      timeout: 3
```

На самом деле всё что надо сделать, это написать pojo класс и связать его с конфигурациями, и для этого есть 3 способа: 

## Через @Bean и @ConfigurationProrperties 

Pojo класс: 
``` Java
@Data  
@NoArgsConstructor  
public class DatabaseProperties {  
  
    private String username;  
    private String password;  
    private String driver;  
    private String hosts;  
    private Map<String, Object> properties;  
    private PoolProperties pool;  
    private List<PoolProperties> pools;  
  
    @Data  
    @NoArgsConstructor    public static class PoolProperties {  
        private Integer size;  
        private Integer timeout;  
    }  
}
```

Создаём обычный бин, но добавляем аннотацию **@ConfigurationProperties**
``` Java
@Bean  
@ConfigurationProperties("db")  
public DatabaseProperties databaseProperties(){  
    return new DatabaseProperties();  
}
```

## @ConfigurationsProperties над классом 

``` Java
@Data  
@NoArgsConstructor  
@ConfigurationProperties("db")  
public class DatabaseProperties {  
  
    private String username;  
    private String password;  
    private String driver;  
    private String hosts;  
    private Map<String, Object> properties;  
    private PoolProperties pool;  
    private List<PoolProperties> pools;  
  
    @Data  
    @NoArgsConstructor    public static class PoolProperties {  
        private Integer size;  
        private Integer timeout;  
    }  
}
```

Здесь есть 2 пути: 
	- указать что это бин @Component
	- указать аннотацию @ConfigurationPropertiesScan над любым конфигурационным классом
	
**@ConfigurationPropertiesScan** будет сканировать все пакеты на поиск аннотации @ConfigurationProperties и создавать из них бины


Для того чтобы DatabaseProperties был Immutable сработает только вариант с @Component и и полным конструктором, для чего нужна аннотация *@ConstructorBinding*:
(@ConfigurationPropertiesScan также должна быть)

``` Java
@ConstructorBinding  
@ConfigurationProperties(prefix = "db")  
public record DatabaseProperties( String username,  
                                  String password,  
                                  String driver,  
                                  String hosts,  
                                  Map<String, Object> properties,  
                                  PoolProperties pool,  
                                  List<PoolProperties> pools) {  
    public static record PoolProperties(Integer size,  
                                        Integer timeout) {  
    }  
}
```

