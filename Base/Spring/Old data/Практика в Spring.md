#liquebase

## Конфигурация

```groovy
dependencies {
	immplementation 'org.liquibase:liquibase-core'
}
```


### SpringAutoConfiguration

``` Java
@Configuration(proxyBeanMethods = false)  
@ConditionalOnClass({ SpringLiquibase.class, DatabaseChange.class })   
@ConditionalOnProperty(prefix = "spring.liquibase", name = "enabled", matchIfMissing = true)  
@Conditional(LiquibaseDataSourceCondition.class)  
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, HibernateJpaAutoConfiguration.class })  
@Import(DatabaseInitializationDependencyConfigurer.class)  
public class LiquibaseAutoConfiguration { 

	@Configuration(proxyBeanMethods = false)  
	@ConditionalOnClass({ SpringLiquibase.class, DatabaseChange.class })  
	@ConditionalOnProperty(prefix = "spring.liquibase", name = "enabled", matchIfMissing = true)  
	@Conditional(LiquibaseDataSourceCondition.class)  
	@AutoConfigureAfter({ DataSourceAutoConfiguration.class, HibernateJpaAutoConfiguration.class })  
	@Import(DatabaseInitializationDependencyConfigurer.class)  
	public class LiquibaseAutoConfiguration {
		private final LiquibaseProperties properties;
		
		...
	}
	
	...
}
```

**SpringLiquibase** - класс, который идёт из зависимостей которые мы подключили, как и большинство от которых зависит автоконфигурация. 

Также она зависит от натсроенного *dataSource* и  *hibernate*

Сама конфигурация сразу начинает оперировать *LiquebaseProperies*

``` Java
@ConfigurationProperties(prefix = "spring.liquibase", ignoreUnknownFields = false)  
public class LiquibaseProperties {  

	 //Change log configuration path.
      private String changeLog = "classpath:/db/changelog/db.changelog-master.yaml";
...
}
```
а  там можем найти путь, по которому будет искаться основной changelog

все это peoperties настраиваются с помощью spring.liquibase


### Работа

![[Pasted image 20230302225354.png]]

``` YAML
databaseChangeLog:  
  - include:  
      file: db/changelog/db.changelog-1.0.sql  
  - include:  
      file: db/changelog/db.changelog-2.0.sql
```

``` SQL
--liquibase formatted sql  
  
--changeset motorin:1  
CREATE TABLE IF NOT EXISTS company  
(  
    id SERIAL PRIMARY KEY ,  
    name VARCHAR(64) NOT NULL UNIQUE  
);  
--rollback DROP TABLE company  
  
--changeset motorin:2  
CREATE TABLE IF NOT EXISTS company_locales  
(  
    company_id INT REFERENCES company (id),  
    lang VARCHAR(2),  
    description VARCHAR(255) NOT NULL ,  
    PRIMARY KEY (company_id, lang)  
);  
--rollback DROP TABLE company_locales
```


### Что делать если появилась необходимость изменить скрипт?  


Если мы уже накатили таблицы, и решили их изменить, то при попытке накатить заного выбросится исключение. 

Чтобы обойти исключение, нужно удалить  запись из changelog таблицы об этом chageset-е  (у которого изменился скрипт), так ка именно она говорит что md5 хэш не совпадает и схема изменена.