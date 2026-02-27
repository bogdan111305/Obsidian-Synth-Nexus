2022-09-0317:14
Tags: #springbase #spring #dmdev 
_
# Спринг введение 

Spring во много реализует спецификации *JavaEE* 

## Invertion of Control

**Inversion of Control** - принцип программирования при котором управление выполнением программы передаётся фреймворку, а не программисту. 

В Spring реализацией **IoC** выступает механизм **Dependency Injection** 

Необходимо предоставить какое-то количество **Bean Definition** исходя из которых IoC контейнер будет создавать **Bean**

**Bean** - это объект, который создан spring контейнером (context), исходя из определения Bean Definition 

IoC container - представляет собой реализацию **BeanFactory** либо **ApplicationContext**
![[Pasted image 20220903210304.png]]

## 3 способа определения Bean Definitions
- XML-based
- Annotation-based
- Java-based