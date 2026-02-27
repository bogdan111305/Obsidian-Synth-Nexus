2022-09-27 22:01
Tags: #spring #springbase #dmdev #springXML 
# Component scan
![[Pasted image 20220927220235.png]]

необходимо указать пакет, где будут сканироваться бины

## Bean Definition Readers
классы которые занимаются считыванием наших bean definitions в зависимости от типа конфигурации: 

![[Pasted image 20220927220428.png]]

по сути они и определяют 3 вида конфигурации: XMLBased, AnnotationBased и JavaBased


**BeanDefinitionReader** реализуют 3 основных класса: 
- XMLBEanDefinitionReader - парсит xml файл и считывает definitions бинов
- GroovyBeanDefinitionReader - парсит groovy скрипт и считывает его definitions 
- PropertiesBeanDefinitions - *deprecated* раньше бины можно было прописывать даже в property файле 

**ClassPathBeanDefinitionScanner** и **AnnotatedBeanDefinitionReader**: 
- благодаря сканеру появляется возможность указать шаблон пути к нашим классам, которые будут bean definitions  
- с помощью AnnotatedBeanDefinitionReader можно зарегистрировать бины в ручную

**ConfigurationClassBeanDefinitionReader**: 
- конфигурация с помощью аннотаций *@Configuration* и *@Bean*


ComponentScan использует под капотом не только рефлексию но и IO, так как у рефлексии нет функционала поиска классов по пакетам. 
Тут включается поиск с помощью IO на жёстком диске, по указанному шаблону classpath который мы передали


``` Java
@Override  
@Nullable  
public BeanDefinition parse(Element element, ParserContext parserContext) {  
   String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);  
   basePackage = parserContext.getReaderContext()
	   .getEnvironment().resolvePlaceholders(basePackage);  
   String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,  
         ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);  
  
   // Actually scan for bean definitions and register them.  
   ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);  
   Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);  
   registerComponents(parserContext.getReaderContext(), beanDefinitions, element);  
  
   return null;
}
```

### TypeFilters

![[Pasted image 20220929215245.png]]


Пример добавления фильтров. Для эксперимента, мы отключили дефолтный фильтр.

![[Pasted image 20220929215804.png]]

- annotation - создали фильтр, который забирает классы аннотированные @Component (по сути вернули дефолтный)
- assignable - указали интерфейс, реализации которого будет искать фильтр
- regex - указали регулярку, по которой будут искаться классы фильтром

*scoped-proxy* - создавать ли proxy на основании классов (по умолчанию no)