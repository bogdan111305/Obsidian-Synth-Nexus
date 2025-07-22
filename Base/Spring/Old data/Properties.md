2022-10-05 21:04
Tags: #spring #springboot #dmdev #springbase 
_
# Properties


Как мы знаем, в springboot для properties есть зарезервированное название <i style='color:#f3ff00'>application.properties</i> - по которому properties автоматически подхватываются приложением. Но на самом деле, это не единственный способ задания properties..


	дфылвфдв

## Spring.properties

Это файл в который можно положить конфигурации самого спринга. И получить затем с помощью класса *SpringProperties*.
[полный список зарезервированных значений](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#appendix.application-properties.core)


## Все варианты передачи properties 

Они идут в порядке возрастания приоретета. Т. е. каждый последующий перекрывает предыдущий.
Главное: если комбинируются варианты то приоритетный property влияет только на те, которые в нём прописывались - т.е. не затирает настройки того, что определялось до него и в нём не определяется.

1.  Default properties (specified by setting `SpringApplication.setDefaultProperties`).
2.  [`@PropertySource`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/context/annotation/PropertySource.html) annotations on your `@Configuration` classes. Please note that such property sources are not added to the `Environment` until the application context is being refreshed. This is too late to configure certain properties such as `logging.*` and `spring.main.*` which are read before refresh begins.
3.  Файлы конфигураций (such as `application.properties` files).
4.  A `RandomValuePropertySource` that has properties only in `random.*`.
5.  OS environment variables.
6.  Java System properties (`System.getProperties()`).
7.  JNDI attributes from `java:comp/env`.
8.  `ServletContext` init parameters.
9.  `ServletConfig` init parameters.
10.  Properties from `SPRING_APPLICATION_JSON` (inline JSON embedded in an environment variable or system property).
11.  Command line arguments.
12.  `properties` attribute on your tests. Available on [`@SpringBootTest`](https://docs.spring.io/spring-boot/docs/2.7.4/api/org/springframework/boot/test/context/SpringBootTest.html) and the [test annotations for testing a particular slice of your application](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing.spring-boot-applications.autoconfigured-tests).
13.  [`@TestPropertySource`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/test/context/TestPropertySource.html) annotations on your tests.
14.  [Devtools global settings properties](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools.globalsettings) in the `$HOME/.config/spring-boot` directory when devtools is active.

варианты передачи фвйлов конфигураций: 

1.  [`Application properties`](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files) packaged inside your jar (`application.properties` and YAML variants).   
2.  [`Profile-specific application properties`](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files.profile-specific) packaged inside your jar (`application-{profile}.properties` and YAML variants).
3.  [`Application properties`](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files) outside of your packaged jar (`application.properties` and YAML variants).
4.  [`Profile-specific application properties`](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files.profile-specific) outside of your packaged jar (`application-{profile}.properties` and YAML variants).


### Передача properties аргументы

**В случае programm arguments:**
с помощью  `--`  spring поймет что это аргументы конфигурации
`--spring.profiles.active=qa`

В случае VM options: 
с помощью `-D`
`-Ddb.username=dmdev`

### Property placeholder

Подставка значений как выражений

```
app.name=MyApp 
app.description=${app.name}
app.description=${username:Unknown}  //дефолтное значение
```


### Путь к файлу .properties

можно указать через запятую
`--spring.config.locations=classpath:application.properties, optional:file:absolutePath/appplication.properties`