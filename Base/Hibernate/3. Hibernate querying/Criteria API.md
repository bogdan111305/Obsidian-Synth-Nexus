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

## 11. Senior-insights: продвинутые возможности <a name="senior"></a>

### Subqueries: Subquery<T> и корреляция

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

// Subquery: найти пользователей, у которых есть хоть один заказ > 1000
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> user = cq.from(User.class);

Subquery<Long> subquery = cq.subquery(Long.class);
Root<Order> order = subquery.from(Order.class);
subquery.select(order.get(Order_.userId))
        .where(
            cb.greaterThan(order.get(Order_.total), new BigDecimal("1000")),
            cb.equal(order.get(Order_.userId), user.get(User_.id)) // корреляция
        );

cq.where(cb.exists(subquery));
List<User> result = em.createQuery(cq).getResultList();
```

### CriteriaUpdate и CriteriaDelete

```java
// CriteriaUpdate — типобезопасный UPDATE
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaUpdate<Product> update = cb.createCriteriaUpdate(Product.class);
Root<Product> root = update.from(Product.class);

update.set(root.get(Product_.price), cb.prod(root.get(Product_.price), 1.1))
      .where(cb.equal(root.get(Product_.category), "electronics"));

int updated = em.createQuery(update).executeUpdate();

// CriteriaDelete — типобезопасный DELETE
CriteriaDelete<Order> delete = cb.createCriteriaDelete(Order.class);
Root<Order> orderRoot = delete.from(Order.class);

delete.where(
    cb.lessThan(orderRoot.get(Order_.createdAt), LocalDate.now().minusYears(2))
);

int deleted = em.createQuery(delete).executeUpdate();
```

> [!WARNING]
> `CriteriaUpdate` и `CriteriaDelete` — bulk операции. Они не инвалидируют L2 Cache и не вызывают lifecycle callbacks, аналогично HQL bulk UPDATE/DELETE.

### exists / not exists через Subquery

```java
// Найти отделы БЕЗ ни одного сотрудника (NOT EXISTS)
CriteriaQuery<Department> cq = cb.createQuery(Department.class);
Root<Department> dept = cq.from(Department.class);

Subquery<Long> sub = cq.subquery(Long.class);
Root<Employee> emp = sub.from(Employee.class);
sub.select(emp.get(Employee_.id))
   .where(cb.equal(emp.get(Employee_.department), dept)); // корреляция

cq.where(cb.not(cb.exists(sub)));

List<Department> emptyDepts = em.createQuery(cq).getResultList();
```

### Multiselect и Tuple projections

```java
// Tuple: выбрать несколько полей без DTO
CriteriaQuery<Tuple> tq = cb.createTupleQuery();
Root<User> user = tq.from(User.class);

tq.multiselect(
    user.get(User_.id).alias("userId"),
    user.get(User_.email).alias("email"),
    user.get(User_.company).get(Company_.name).alias("companyName")
);

List<Tuple> tuples = em.createQuery(tq).getResultList();
for (Tuple t : tuples) {
    Long id = t.get("userId", Long.class);
    String email = t.get("email", String.class);
    String companyName = t.get("companyName", String.class);
}

// DTO через construct:
CriteriaQuery<UserDto> dtoQuery = cb.createQuery(UserDto.class);
Root<User> u = dtoQuery.from(User.class);
dtoQuery.select(cb.construct(
    UserDto.class,
    u.get(User_.id),
    u.get(User_.email),
    u.get(User_.company).get(Company_.name)
));
List<UserDto> dtos = em.createQuery(dtoQuery).getResultList();
```

### Сравнение с QueryDSL: когда что выбирать

| Критерий | Criteria API | QueryDSL |
|----------|-------------|---------|
| Типобезопасность | Да (через метамодель) | Да (через Q-классы) |
| Читаемость | Низкая (verbose) | Высокая (fluent API) |
| Поддержка JPA | Стандарт JPA | Внешняя библиотека |
| Сложные запросы | Громоздко | Элегантно |
| Поддержка | Всегда (JPA стандарт) | Требует зависимость |
| Дополнительная генерация | `hibernate-jpamodelgen` | `querydsl-apt` |

```java
// Criteria API — более verbose:
cq.where(
    cb.and(
        cb.equal(root.get(User_.active), true),
        cb.greaterThan(root.get(User_.age), 18)
    )
);

// QueryDSL — более читаемо:
queryFactory.selectFrom(QUser.user)
    .where(QUser.user.active.isTrue()
        .and(QUser.user.age.gt(18)))
    .fetch();
```

**Когда выбирать Criteria API**:
- Проект не хочет внешних зависимостей
- Нужна стандартная JPA совместимость
- Простые динамические фильтры

**Когда выбирать QueryDSL**:
- Сложные динамические запросы с множеством условий
- Важна читаемость кода
- Команда готова добавить зависимость

---

**Дополнительные ресурсы:**
- [Hibernate Criteria API Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#criteria)
- [JPA Criteria API Guide](https://jakarta.ee/specifications/persistence/3.0/jakarta-persistence-spec-3.0.html#criteria)

**Связанные файлы:**
- [[HQL]] — HQL/JPQL запросы
- [[Querydsl]] — QueryDSL подробнее
