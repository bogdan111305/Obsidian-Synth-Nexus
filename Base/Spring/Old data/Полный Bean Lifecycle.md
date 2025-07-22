2022-09-06 22:20
Tags:  #spring #springbase #dmdev #beanLifecycle
_
# Bean lifecicle


![[Pasted image 20220906222008.png]]

**Приходит набор Bean Definitions **
2. При попадании в IoC Container начинают отрабатывать BeanFactoryPostProcessors
3. Bean Definitions сортируются, так как одни бины зависят от других
4. По одному с помощью for-each начинается этап инициализации
	1. Вызывается конструктор 
	2. Вызываются сеттеры, после этого можно по сути можно работать с бином
	3. Bean попадает ко всем определённым BPP где вызывается *beforeInitialization()* метод
	4. Вызывается InitializationCallback - и на самом деле это тоже BPP который идёт последним
	5. У BPP в том же самом порядке вызываются *afterInitialization()*
5. Имеем готовые бины
6. Если бин singleton то перед его стиранием вызовется метод destroy