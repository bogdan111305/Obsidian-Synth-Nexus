2022-09-26 22:18
Tags: #spring #springbase #dmdev 
_
# @Autowired, @Value и @Resource
![[Pasted image 20220926221949.png]]



## Resource

![[Pasted image 20220926223202.png]]


Аннотация resource из пакета javakarta, который не входит в spring core.

Применение: 
Можем ставить над типом полем и методом. Важно: нельзя поставить перед конструктором! 

отличие от autowired: может инжектить в зависимости от названия бина, что может быть удобно при неопределённости

``` Java
@Resource(name = "myBean")  
private SomeBean someBean;
```

Эта аннотация нужна только для того, чтобы удовлетворять Java EE спецификации JSR-250

на практике чаще используется именно spring аннотация **autowired**

## Autowired


Можно ставить даже над другой аннотацией. 

Autowired в отличие от @Resource при инжекте полагается на тип поля, а не на его имя.

При неопределённости: 
- поле *required* - фэйлить ли контест если не найдётся бин 
- для указания какой именно бин придётся указать *@Qualyfire*
- autowired заинжектит если id бина и имя поля в который надо инжектить совпадает 

``` Java
//при двух бинах SomeBean Autowired заинжектит тот id которого someBean  
@Autowired(required = false)  
private SomeBean someBean;
```

Autowired может инжектить целые коллекции

## Value

``` Java
//может использоваться spel
@Value("${someValue}")  
private String name;
```

