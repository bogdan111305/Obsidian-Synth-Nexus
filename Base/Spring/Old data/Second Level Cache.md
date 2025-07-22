2022-10-16 18:41
Tags: #hibernate #dmdev 
_

Второй уровень кэширования строится на **уровне SessionFactory**.
*по умодчанию выключен и нуждается в дополнительной настройке*

Вот как это выглядит: 
![[image 1.png]]


Кэш разбит на regions - аналоги Persistent Context
Сущность хранится в виде сериализованного объекта.
regions могут настраиваться: объём памяти, как долго сущность будет жить в region и тд.

Некторые понятия связанные с кэшем: 
- Cache Miss  - когда мы обратились к кэшу и не нашли там какого-либо значения
- Cache Put - когда мы положили сущность из бд в кэш
- Cache Hit - когда мы нашли необходимую сущность в кэше

Такой кэш удобен в том случае, когда сущности редко меняются (справочники, словари и тд.)

## Подключение кэша в SessionFactory

2 основных интерфейса: 
`RegionFactoryTemplate` - создаёт region. По умолчанию hibernate не предоставляет реализацию - мы можем написать её сами или подключить уже существующую.
`DomainDataStorageAccess` - для работы с самим кэшем и его настройки.

В качестве реализации оспользуем ehcache
``` Groovy 
implementation 'org.hibernate:hibernate-jcache:6.1.3.Final'  
implementation 'org.ehcache:ehcache-transactions:3.10.1'
```


### Аннотация @Cache 

По сути regions хранят такую же мапу как и PersistentContext, где:
- ключ - класс и id 
- значение - сериализованные значения полей 
![[Pasted image 20221016192454.png]]

Если сессия ищет сущность в SecondLvl кэше, то когда она его найдёт она создаст объект из сериализованных  данных и положит его в свой PersistentContext 

Для того чтобы сущность добавлялась в кэш второго уровня, над её классом надо поставить аннотацию <i style="color:#f3ff00">@Cache</i>
``` Java
@Entity  
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  
public class SomeClass {
```

#### Кэширование при свзяи ManyToOne

Предположим у нас есть User и Comany

В стандартном варианте, когда аннотация только над классом User, Company не кэшируется - кэшируется только id этой Company, и соответственно, чтобы достать Company придётся сделать новый запрос.

Чтобы кэшировалась сама сущность Company, над ней тоже надо поставить аннотацию @Cache, в таком случае произойдёт следующее: 
![[Pasted image 20221016195425.png]]

В user закэшируется id, но так как Company тоже будет закэширован, hibernate легко его отыщет. Без дополнительных запросов в бд.

#### Кэширования коллекций (OneToMany)

Так как у нас маппинг коллекции, то сами id не закешируются, так как фактически бд не хранит в таблице User id user_chats например..

В таком случае @Cache надо ставить и над сущностю маппинга и над полем коллекции

![[Pasted image 20221016201439.png]]

### Regions

Regions - это просто области памяти которые используются для хранения сериализованных значений.
Так же они позволяют производить некоторую настройку: 
- как много сущностей можем сохранить
- как долго они будут храниться

По умолчанию при каждой аннотации @Cache создаётся свой Region и именуется по полному пути класса или поля.
Однако мы можем указать его явно: 
`@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "SomeRegion")`

Пример настройки SomeRegion
``` XML
<?xml version="1.0" encoding="UTF-8"?>  
<ehcache:config  
        xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'  
        xmlns:ehcache='http://www.ehcache.org/v3'  
        xsi:schemaLocation="http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.1.xsd">  
  
    <ehcache:cache alias="SomeRegion" uses-template="simple"/>  
  
    <ehcache:cache-template name="simple">  
        <ehcache:expiry>  
            <ehcache:ttl>10</ehcache:ttl>  
        </ehcache:expiry>  
        <ehcache:heap>1000</ehcache:heap>  
    </ehcache:cache-template>  
  
</ehcache:config>
```

### Стретегии поведения кэширования при транзацкциях

указывается в аннотации @Cache
за стратегию отвечает класс CacheConcurrencyStrategy, а точнее его константы: 
- NONE
- READ_ONLY - из бд можно только читать
- NONSTRICT_READ_WRITE - похожа на следующую, но даёт небольшой промежуток времени, когда новая транзакция может получить старые данные - инвалидирование происходит не сразу. Увеличивает пропускную спопробность
- READ_WRITE - можно взаимодействовать, но происходит lock и ждёт пока первая транзакция которая начала что-то менять инвалидировала кэш и остальные транзакции ждут пока она сделает комит
- TRANSACTIONAL - стратегия для распределённых транзакций

### Кэширование запросов 

В качестве ключа берётся вся строка нашего запроса: 
![[Pasted image 20221016204132.png]]

Кэширование запросов также необходимо включить в конфигурации: 
```XML
<property name="hibernate.cache.use_query_cache">true</property>
```


``` Java
var query = session.createQuery("select d from Dog d where id=:id", Dog.class)
.setParameter("id", 1)
.setCacheable(true) //делаем запрос кэшируемым
//.setCacheRegion("myRegion") //устанавливаем свой region (может быть дефолтным)
//.setHint(QueryHints.CACHEABLE,true) //делаем кэширование через hits, просто другой вариант
.getResultList();
```