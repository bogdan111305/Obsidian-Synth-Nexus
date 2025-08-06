# Querydsl: подробное руководство по типобезопасным запросам

## Оглавление
1. [Что такое Querydsl и зачем он нужен](#intro)
2. [Настройка: подключение плагина, зависимостей, tasks](#setup)
3. [Решение проблем совместимости](#compat)
4. [Базовые примеры запросов](#basic)
5. [Практические задачи](#tasks)
6. [Динамические фильтры и QPredicates](#filters)
7. [Best practices](#best-practices)
8. [Вопросы для собеседования](#вопросы)

---

## 1. Что такое Querydsl и зачем он нужен <a name="intro"></a>

- **Querydsl** — это фреймворк для построения типобезопасных SQL/JPA/JPQL/SQL запросов на Java.
- Позволяет строить сложные, динамические, переносимые запросы с помощью Java-кода, а не строк.
- Использует сгенерированные Q-классы для типобезопасности.

## 2. Настройка: подключение плагина, зависимостей, tasks <a name="setup"></a>

### Maven
```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-jpa</artifactId>
  <version>5.0.0</version>
</dependency>
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-apt</artifactId>
  <version>5.0.0</version>
  <scope>provided</scope>
</dependency>
```
- Включите annotation processing в IDE/Gradle/Maven.

### Gradle
```groovy
dependencies {
    implementation 'com.querydsl:querydsl-jpa:5.0.0'
    annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jpa'
}
```
- Добавьте task для генерации Q-классов.

## 3. Решение проблем совместимости <a name="compat"></a>
- Используйте совместимые версии querydsl и JPA/Hibernate
- Проверьте, что annotation processing включён
- Для Kotlin используйте kapt
- Для Spring Boot: используйте spring-boot-configuration-processor

## 4. Базовые примеры запросов <a name="basic"></a>

### findAll
```java
JPAQueryFactory queryFactory = new JPAQueryFactory(entityManager);
QUser user = QUser.user;
List<User> users = queryFactory.selectFrom(user).fetch();
```

### findAllByFirstName
```java
List<User> users = queryFactory.selectFrom(user)
    .where(user.firstName.eq("John")).fetch();
```

### findLimitedUsersOrderedByBirthday
```java
List<User> users = queryFactory.selectFrom(user)
    .orderBy(user.birthday.asc())
    .limit(10)
    .fetch();
```

### findAllByCompanyName
```java
QCompany company = QCompany.company;
List<User> users = queryFactory.selectFrom(user)
    .join(user.company, company)
    .where(company.name.eq("Acme")).fetch();
```

### findAllPaymentsByCompanyName
```java
QPayment payment = QPayment.payment;
QUser user = QUser.user;
QCompany company = QCompany.company;
List<Payment> payments = queryFactory.selectFrom(payment)
    .join(payment.user, user)
    .join(user.company, company)
    .where(company.name.eq("Acme")).fetch();
```

### findAveragePaymentAmountByFirstAndLastNames
```java
Double avg = queryFactory.select(payment.amount.avg())
    .from(payment)
    .join(payment.user, user)
    .where(user.firstName.eq("John"), user.lastName.eq("Doe"))
    .fetchOne();
```

### findCompanyNamesWithAvgUserPaymentsOrderedByCompanyName
```java
List<Tuple> result = queryFactory.select(company.name, payment.amount.avg())
    .from(company)
    .join(company.users, user)
    .join(user.payments, payment)
    .groupBy(company.name)
    .orderBy(company.name.asc())
    .fetch();
```

### isItPossible (exists, in, not in, подзапросы)
```java
Boolean exists = queryFactory.selectFrom(user)
    .where(user.active.isTrue())
    .fetchFirst() != null;
```

## 5. Динамические фильтры и QPredicates <a name="filters"></a>

### Создание DTO (PaymentFilter)
```java
public class PaymentFilter {
    private String companyName;
    private String userFirstName;
    private BigDecimal minAmount;
    // ...
}
```

### Динамический запрос через Querydsl
```java
BooleanBuilder builder = new BooleanBuilder();
if (filter.getCompanyName() != null) {
    builder.and(company.name.eq(filter.getCompanyName()));
}
if (filter.getUserFirstName() != null) {
    builder.and(user.firstName.eq(filter.getUserFirstName()));
}
if (filter.getMinAmount() != null) {
    builder.and(payment.amount.goe(filter.getMinAmount()));
}
List<Payment> payments = queryFactory.selectFrom(payment)
    .join(payment.user, user)
    .join(user.company, company)
    .where(builder)
    .fetch();
```

### Создание QPredicates
```java
public class QPredicates {
    private final BooleanBuilder builder = new BooleanBuilder();
    public QPredicates add(BooleanExpression expr) {
        if (expr != null) builder.and(expr);
        return this;
    }
    public Predicate build() { return builder; }
}
```

### Использование QPredicates
```java
Predicate predicate = QPredicates.builder()
    .add(company.name.eq(filter.getCompanyName()))
    .add(user.firstName.eq(filter.getUserFirstName()))
    .build();
List<Payment> payments = queryFactory.selectFrom(payment)
    .join(payment.user, user)
    .join(user.company, company)
    .where(predicate)
    .fetch();
```

## 6. Best practices <a name="best-practices"></a>
- Используйте Querydsl для сложных, динамических и типобезопасных запросов
- Генерируйте Q-классы автоматически (annotation processing)
- Для простых запросов используйте HQL/JPQL
- Для сложных фильтров используйте BooleanBuilder/QPredicates
- Следите за производительностью и кэшированием

## 7. Вопросы для собеседования <a name="вопросы"></a>
1. Чем отличается Querydsl от Criteria API?
2. Как работает типобезопасность в Querydsl?
3. Как реализовать динамический фильтр?
4. Как использовать QPredicates?
5. Как реализовать подзапросы в Querydsl?

---

**Дополнительные ресурсы:**
- [Querydsl Documentation](https://querydsl.com/static/querydsl/latest/reference/html_single/)
- [Querydsl JPA Guide](https://github.com/querydsl/querydsl/blob/master/querydsl-docs/src/main/asciidoc/jpa-guide.adoc)
