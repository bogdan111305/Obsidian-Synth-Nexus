2022-09-0521:21
Tags: #spring #springbase #dmdev 
_
# Scope of beans

![[Pasted image 20220905212209.png]]


## Common scopes

**singleton** - создаётся только один instance 
**prototype** - столько инстансов сколько нужно
**trhead** - для отдельного потоков свой instance - нужно регистрировать самостоятельно


Такие scope может регистрировать **ApplicationContext** так как он реализует интерфейс *Scope*

Важно различие между singletone и prototype: 

syngleton - *создаются сразу* в соответствии с их bean definitions и при вызове достаются из контейнера

prototype - *создаются* тогда, *когда* мы к ним *обращаемся*. Bean проходит весь цикл инициализации. 


## Custom scopes
мы можем создать собственный scope 
1. реализуем интерфейс *scope*
2. реализуем его метод *registerScope*



## Web scopes

**request**
**session**
**application**
**websocket**

может регистрировать **WebApplicationContext**