2022-09-0522:01
Tags: #spring #springbase #dmdev #bean #beeLifecycle
_
# Lifecycle Callbacks

![[Pasted image 20220905220323.png]]

**Inintialization callback** - предоставляет 3 варианта что-то подкрутить в бине после инициализации: 

1. @PostConstruct 
1. afterPropertiesSet()
1. init-method (XML)


**Destruction callbacks** - 3 варианта что-то подкрутить в бине при его разрушении

1. @PreDestroy
2. destroy()
3. destroy-method (XML)

Важно: destroy callbacks - не вызываются у бинов со scope *prototype* - так как контекст не хранит на них ссылки 

