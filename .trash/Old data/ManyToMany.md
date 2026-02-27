2023-07-11 20:26
Tags: #hibernate #note



```Java
  
  \\ Первая сторона 
  
@ManyToMany  
@JoinTable(  
        name = "kit_word",  
        joinColumns = @JoinColumn(name = "kit_id"),  
        inverseJoinColumns = @JoinColumn(name = "word_id")  
)  
private Set<Word> words = new HashSet<>();

  
 \\ Вторая сторона

@ManyToMany(mappedBy = "words",fetch = FetchType.LAZY)  
private Set<Kit> kits = new HashSet<>();

```


Надо сделать заметки по хибернейту. 


