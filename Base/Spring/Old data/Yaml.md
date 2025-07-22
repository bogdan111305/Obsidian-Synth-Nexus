2022-10-05 22:29
Tags: #springboot  #dmdev 
_
# Yaml

позволяет не дублировать наборы в properties 

пример: 

``` YAML
db:
	username: ${username.value:postgres}
	password: pass
	driver: PostgresDriver
	url:postgres:5432
	hosts: localhost,127.0.0.1
	#объект
	pool:         
		size: 12
		timeout: 10
	#массив объектов		
	pools:       
	- size: 1
	  timeout: 1
	- size: 2
	  timeout: 2
	- size: 3
	  timeout: 3
spring.profiles.active: qa
```
