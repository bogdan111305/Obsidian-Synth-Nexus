# Entity Graphs в Hibernate: современное решение проблемы N+1

## Оглавление
1. [Введение: зачем нужны Entity Graphs](#введение)
2. [Аннотации и статические графы](#аннотации)
3. [Динамические Entity Graphs](#динамика)
4. [Subgraph и вложенные связи](#subgraph)
5. [Сравнение Entity Graph и JOIN FETCH](#сравнение)
6. [Ограничения и edge cases](#ограничения)
7. [Типовые ошибки и подводные камни](#ошибки)
8. [Best practices](#best)
9. [Чек-лист для проекта](#чеклист)
10. [Вопросы для собеседования](#вопросы)

---

## <a name="введение"></a>Введение: зачем нужны Entity Graphs

Entity Graphs — мощный инструмент JPA/Hibernate для гибкого управления загрузкой связанных сущностей. Они позволяют явно указать, какие связи и атрибуты должны быть загружены в одном запросе, устраняя проблему N+1 и давая полный контроль над fetch-стратегиями без изменения модели.

---

## <a name="аннотации"></a>Аннотации и статические графы

- Используйте `@NamedEntityGraph` для описания часто используемых графов на уровне модели.
- **Пример:**
```java
@NamedEntityGraph(
    name = "User.withProfile",
    attributeNodes = @NamedAttributeNode("profile")
)
@Entity
public class User {
    @Id
    private Long id;
    @OneToOne(fetch = FetchType.LAZY)
    private Profile profile;
}
```
- **Использование:**
```java
EntityGraph<User> graph = em.getEntityGraph("User.withProfile");
List<User> users = em.createQuery("SELECT u FROM User u", User.class)
    .setHint("javax.persistence.fetchgraph", graph)
    .getResultList();
```

---

## <a name="динамика"></a>Динамические Entity Graphs

- Позволяют строить графы на лету, без аннотаций.
- **Пример:**
```java
EntityGraph<User> graph = em.createEntityGraph(User.class);
graph.addAttributeNodes("profile");
List<User> users = em.createQuery("SELECT u FROM User u", User.class)
    .setHint("javax.persistence.fetchgraph", graph)
    .getResultList();
```
- Можно добавлять subgraph для вложенных связей (см. ниже).

---

## <a name="subgraph"></a>Subgraph и вложенные связи

- Subgraph позволяет загружать вложенные связи (например, User -> Profile -> Address).
- **Пример:**
```java
EntityGraph<User> graph = em.createEntityGraph(User.class);
Subgraph<Profile> profileSubgraph = graph.addSubgraph("profile");
profileSubgraph.addAttributeNodes("address");
List<User> users = em.createQuery("SELECT u FROM User u", User.class)
    .setHint("javax.persistence.fetchgraph", graph)
    .getResultList();
```
- В аннотациях:
```java
@NamedEntityGraph(
    name = "User.withProfileAndAddress",
    attributeNodes = {
        @NamedAttributeNode(value = "profile", subgraph = "profile-address")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "profile-address",
            attributeNodes = @NamedAttributeNode("address")
        )
    }
)
```

---

## <a name="сравнение"></a>Сравнение Entity Graph и JOIN FETCH

| Характеристика         | Entity Graph         | JOIN FETCH         |
|------------------------|---------------------|--------------------|
| Гибкость               | Высокая (динамика)  | Только в запросе   |
| Влияние на модель      | Нет                 | Нет                |
| Поддержка вложенных    | Да (subgraph)       | Нет                |
| Несколько коллекций    | Можно, но осторожно | Декартово произведение |
| Использование с HQL    | Да (setHint)        | Да                 |
| Использование с Criteria| Да                  | Да                 |
| Ограничения            | Не все провайдеры JPA поддерживают subgraph одинаково | JOIN FETCH не работает с коллекциями одновременно |

- **Best practice:** для сложных графов и динамики — Entity Graph, для простых случаев — JOIN FETCH.

---

## <a name="ограничения"></a>Ограничения и edge cases
- Не используйте Entity Graph для загрузки нескольких больших коллекций одновременно — возможен взрыв данных (декартово произведение).
- Не все JPA-провайдеры одинаково поддерживают subgraph и вложенные графы.
- fetchgraph загружает только явно указанные атрибуты, loadgraph — также EAGER-атрибуты.
- Entity Graph не работает с native SQL-запросами.
- Не забывайте про LAZY/EAGER: Entity Graph может переопределить только fetch-стратегию, но не тип коллекции.

---

## <a name="ошибки"></a>Типовые ошибки и подводные камни
- Использование fetchgraph без указания всех нужных связей — часть данных не загрузится.
- Одновременная загрузка нескольких коллекций — декартово произведение.
- Неаккуратное использование subgraph — избыточные данные.
- Не работает с native SQL.
- Не все провайдеры поддерживают subgraph одинаково.
- Ошибки в именах атрибутов — граф не применяется, Hibernate молчит.

---

## <a name="best"></a>Best practices
- Используйте Entity Graph для сложных графов и динамических сценариев.
- Для повторяющихся сценариев — @NamedEntityGraph.
- Для массовых выборок — fetchgraph, для частичных — loadgraph.
- Проверяйте сгенерированный SQL и количество запросов.
- Не загружайте несколько коллекций одновременно.
- Включайте логирование SQL и используйте Hibernate Statistics.
- Тестируйте сценарии с большими объёмами данных.

---

## <a name="чеклист"></a>Чек-лист для проекта
- [ ] Для сложных графов используете Entity Graph
- [ ] Для повторяющихся сценариев — @NamedEntityGraph
- [ ] Не загружаете несколько коллекций одновременно
- [ ] Проверяете SQL и количество запросов
- [ ] Используете интеграционные тесты для контроля N+1
- [ ] Проверяете поддержку subgraph вашим JPA-провайдером

---

## <a name="вопросы"></a>Вопросы для собеседования
1. Чем Entity Graph отличается от JOIN FETCH?
2. Как работает subgraph в Entity Graph?
3. Какие ограничения у Entity Graph?
4. Как использовать динамические Entity Graph?
5. Какой fetchgraph выбрать: fetchgraph или loadgraph?
6. Как избежать декартова произведения при загрузке коллекций?
7. Как тестировать корректность работы Entity Graph?
8. Как Entity Graph влияет на LAZY/EAGER?
9. Как использовать Entity Graph с Criteria API?
10. Какие типовые ошибки при использовании Entity Graph?

---

**Рекомендуемые материалы:**
- [Официальная документация Hibernate: Entity Graphs](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#fetching-entity-graph)
- [Baeldung: JPA Entity Graphs](https://www.baeldung.com/jpa-entity-graphs)
- [Vlad Mihalcea: High-Performance Java Persistence](https://vladmihalcea.com/tutorials/hibernate/)