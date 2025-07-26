# Criteria API в JPA/Hibernate: подробное руководство

## Оглавление
1. [Что такое Criteria API и зачем он нужен](#intro)
2. [Типобезопасность и преимущества](#typesafe)
3. [Подключение и использование jpamodelgen](#jpamodelgen)
4. [Базовые примеры запросов](#basic)
5. [Динамические запросы (Filter)](#dynamic)
6. [Использование DTO (Projection)](#dto)
7. [Использование Tuple (Projection)](#tuple)
8. [Практические задачи](#tasks)
9. [Best practices](#best-practices)
10. [Вопросы для собеседования](#вопросы)

---

## 1. Что такое Criteria API и зачем он нужен <a name="intro"></a>

- **Criteria API** — это типобезопасный, объектно-ориентированный способ построения запросов к базе данных в JPA/Hibernate.
- Позволяет строить динамические запросы, которые сложно выразить в HQL/JPQL.
- Удобен для сложных фильтров, динамических условий, построения запросов на лету.

## 2. Типобезопасность и преимущества <a name="typesafe"></a>

- Использует метамодель (сгенерированные классы `Q*` или `_*_`), что позволяет ловить ошибки на этапе компиляции.
- Нет риска опечаток в названиях полей.
- Легко строить динамические условия.

## 3. Подключение и использование jpamodelgen <a name="jpamodelgen"></a>

- Добавьте зависимость:
```xml
<dependency>
  <groupId>org.hibernate.orm</groupId>
  <artifactId>hibernate-jpamodelgen</artifactId>
  <version>6.4.4.Final</version>
  <scope>provided</scope>
</dependency>
```
- Включите annotation processing в IDE/Gradle/Maven.
- После компиляции появятся классы `User_`, `Company_` и т.д.

## 4. Базовые примеры запросов <a name="basic"></a>

### findAll
```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.select(root);
List<User> users = em.createQuery(cq).getResultList();
```

### findAllByFirstName
```java
cq.where(cb.equal(root.get(User_.firstName), "John"));
```

### findLimitedUsersOrderedByBirthday
```java
cq.orderBy(cb.asc(root.get(User_.birthday)));
List<User> users = em.createQuery(cq).setMaxResults(10).getResultList();
```

### findAllByCompanyName
```java
Join<User, Company> company = root.join(User_.company);
cq.where(cb.equal(company.get(Company_.name), "Acme"));
```

### findAllPaymentsByCompanyName
```java
CriteriaQuery<Payment> cq = cb.createQuery(Payment.class);
Root<Payment> payment = cq.from(Payment.class);
Join<Payment, User> user = payment.join(Payment_.user);
Join<User, Company> company = user.join(User_.company);
cq.where(cb.equal(company.get(Company_.name), "Acme"));
```

### findAveragePaymentAmountByFirstAndLastNames
```java
cq.select(cb.avg(payment.get(Payment_.amount)))
  .where(cb.and(
    cb.equal(user.get(User_.firstName), "John"),
    cb.equal(user.get(User_.lastName), "Doe")
  ));
```

### findCompanyNamesWithAvgUserPaymentsOrderedByCompanyName
```java
CriteriaQuery<Tuple> cq = cb.createTupleQuery();
Root<Company> company = cq.from(Company.class);
Join<Company, User> user = company.join(Company_.users);
Join<User, Payment> payment = user.join(User_.payments);
cq.multiselect(company.get(Company_.name), cb.avg(payment.get(Payment_.amount)))
  .groupBy(company.get(Company_.name))
  .orderBy(cb.asc(company.get(Company_.name)));
```

### isItPossible (exists, in, not in, подзапросы)
```java
Subquery<Long> sub = cq.subquery(Long.class);
Root<User> subRoot = sub.from(User.class);
sub.select(subRoot.get(User_.id)).where(cb.equal(subRoot.get(User_.active), true));
cq.where(root.get(User_.id).in(sub));
```

## 5. Динамические запросы (Filter) <a name="dynamic"></a>

- Используйте Predicate и CriteriaBuilder для динамического добавления условий:
```java
List<Predicate> predicates = new ArrayList<>();
if (filter.getFirstName() != null) {
    predicates.add(cb.equal(root.get(User_.firstName), filter.getFirstName()));
}
if (filter.getCompanyName() != null) {
    Join<User, Company> company = root.join(User_.company);
    predicates.add(cb.equal(company.get(Company_.name), filter.getCompanyName()));
}
cq.where(predicates.toArray(new Predicate[0]));
```

## 6. Использование DTO (Projection) <a name="dto"></a>

- Для выборки в DTO используйте конструктор:
```java
cq.select(cb.construct(UserDto.class, root.get(User_.id), root.get(User_.firstName)));
```

## 7. Использование Tuple (Projection) <a name="tuple"></a>

- Для выборки нескольких полей без DTO:
```java
cq.multiselect(root.get(User_.id), root.get(User_.firstName));
List<Tuple> tuples = em.createQuery(cq).getResultList();
for (Tuple t : tuples) {
    Long id = t.get(0, Long.class);
    String firstName = t.get(1, String.class);
}
```

## 8. Практические задачи <a name="tasks"></a>
- findAll
- findAllByFirstName
- findLimitedUsersOrderedByBirthday
- findAllByCompanyName
- findAllPaymentsByCompanyName
- findAveragePaymentAmountByFirstAndLastNames
- findCompanyNamesWithAvgUserPaymentsOrderedByCompanyName
- isItPossible (exists, in, not in, подзапросы)
- Реализация динамического фильтра
- Использование DTO и Tuple

## 9. Best practices <a name="best-practices"></a>
- Используйте Criteria API для сложных и динамических запросов
- Включайте jpamodelgen для типобезопасности
- Для простых запросов используйте HQL/JPQL
- Для сложных projection используйте DTO или Tuple
- Не забывайте про кэширование и оптимизацию запросов

## 10. Вопросы для собеседования <a name="вопросы"></a>
1. Чем отличается Criteria API от HQL?
2. Как работает типобезопасность в Criteria API?
3. Как реализовать динамический фильтр?
4. Как выбрать только нужные поля (DTO, Tuple)?
5. Как реализовать подзапросы в Criteria API?

---

**Дополнительные ресурсы:**
- [Hibernate Criteria API Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#criteria)
- [JPA Criteria API Guide](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#criteria)
