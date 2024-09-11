# Pessimistic и Optimistic Locking

В предыдущем модуле мы рассматривали Pessimistic Locking и Optimistic Locking как механизмы для
гарантии консистентности при операциях `UPDATE` и `INSERT`. Давайте на примере `Organization`
посмотрим, какие аномалии возможны в реальном приложении и какие есть способы их преодоления.

Во-первых, немного расширим сущность `Organization`: теперь там будет храниться список `Department`:

```java
public class Organization {

    private final UUID id;
    private final String name;
    private final List<Department> departments;
    private final Subscription requiredSubscription;

    /* constructor, equals, hashCode */

    public static Organization newOrganization(String name) {
        return new Organization(UUID.randomUUID(), name, emptyList(), Subscription.REGULAR);
    }

    public Organization createDepartment(String name, String description) {
        List<Department> newDepartments = new ArrayList<>(this.departments);
        newDepartments.add(new Department(this.id, name, description));
        return new Organization(
            this.id,
            this.name,
            Collections.unmodifiableList(newDepartments),
            newDepartments.size() > 2 ? Subscription.GOLD : Subscription.REGULAR
        );
    }
}

public class Department {

    private final UUID id;
    private final UUID organizationId;
    private final String name;
    private final String description;

    /* constructor, equals, hashCode */

    public static Department newDepartment(UUID organizationId, String name, String description) {
        return new Department(UUID.randomUUID(), name, description);
    }
}

public enum Subscription {
    REGULAR, GOLD
}
```

В `Organization` хранится список `Department`. Также есть метод `Organization.createDepartment`,
который добавляет `Department` и возвращает новый инстанс `Organization`. Интерес здесь в
поле `requiredSubscription`. Предположим, что наши пользователи платят ежемесячную подписку, чтобы
управлять организациями и департаментами. В то же время, если в их организации 2 или менее
департаментов, им достаточно `REGULAR` подписки. Иначе требуется уже `GOLD` уровень. Поэтому мы
выставляем актуальный при вызове `createDepartment`.

Предположим, в `OrganizationRepository` есть метод `update`, который принимает на
вход `Organization` и выполняет следующие действия:

1. Обновляет данные по организации (`name`, `requiredSubscription`)
2. Если департамента нет в таблице, добавляет его.
3. Если департамент есть, обновляет по нему данные (вдруг мы поменяли `name` или какое-то другое
   поле).

Посмотрите на пример кода ниже:

```java
public class OrganizationRepositoryImpl implements OrganizationRepository {

    private final Jdbi jdbi;

    /* конструктор + остальные методы */

    @Override
    public void update(Organization organization) {
        jdbi.useTransaction((Handle handle) -> {
            handle.createUpdate(
                    "UPDATE organization SET name = :name, required_subscription = :sub WHERE id = :id"
                )
                .bind("name", organization.getName())
                .bind("sub", organization.getRequiredSubscription())
                .bind("id", organization.getId())
                .execute();
            for (Department department : organization.getDepartments()) {
                boolean departmentPresent =
                    handle.createQuery("SELECT COUNT(id) FROM department WHERE id = :id")
                        .bind("id", department.getId())
                        .mapToBoolean(Boolean.class)
                        .first();
                if (departmentPresent) {
                    handle.createUpdate(
                            "UPDATE department SET name = :name, description = :description WHERE id = :id"
                        )
                        .bind("name", department.getName())
                        .bind("description", department.getDescription())
                        .bind("id", id)
                        .execute();
                } else {
                    handle.createUpdate(
                            "INSERT INTO department (id, organization_id, name, description) VALUES (:id, :organization_id, :name, :description)"
                        )
                        .bind("id", department.getId())
                        .bind("organization_id", department.getOrganizationId())
                        .bind("name", department.getName())
                        .bind("description", department.getDescription())
                        .execute();
                }
            }
        });
    }
}
```

Алгоритм здесь следующий:

1. Сначала обновляем данные по `Organization` с помощью `UPDATE organization...`.
2. Далее проходимся по списку `Department` в `Organization`.
3. Если `Department` присутствует в БД по ID, то выполняем `UPDATE`. Иначе - `INSERT`.

> Стоит отметить, что подобная комбинация запросов не очень эффективна, потому что мы явно
> проверяем,
> что присутствует в БД, выполняя дополнительный запрос `SELECT COUNT(id)`.
> Вместо этого мы могли бы явно на уровне сущности `Organization` отслеживать тип
> изменений (`Departament` добавлен или отредактирован), чтобы генерировать только нужные запросы.
> Пока мы закроем глазу на эту проблему. В следующем семестре мы погрузимся во
> фреймворк [Hibernate](https://hibernate.org/), где эта проблема решена.

Все операции выполняются в одной транзакции, так что кажется, что никаких проблем нет. Но давайте
рассмотрим ситуацию, в которой два потока (то есть два паралелльных запроса от клиента) пытаются
отредактировать `Organization` одновременно, но с разными условиями.

Будем исходить из того, что в `Organization` уже есть один `Department`. То
есть `Organization.requiredSubsription` равен `REGULAR`.
При этом каждый клиент в своем потоке добавил новый `Department` в `Organization`. С точки зрения
доменной модели, поле `requiredSubsription` по-прежнему равно `REGULAR` (посмотрите еще раз на
код `Organization.createOrganization` выше).

И здесь получается интересная ситуация. Как вы помните из предыдущего модуля, при
выполнении `UPDATE` запись автоматически блокируется до конца транзакции, чтобы
избежать `DIRTY WRITE`. Значит, выполнение двух запросов одновременно может принять такой оборот:

| Поток клиента 1                                                   | Поток клиента 2                                                       |
|-------------------------------------------------------------------|-----------------------------------------------------------------------|
| `SELECT * FROM organization WHERE id = ?`                         | `SELECT * FROM organization WHERE id = ?`                             |
| `UPDATE organization SET required_subscription = REGULAR`         | Ждет, пока блокировка снимется.                                       |
| `INSERT INTO department...` - теперь в организации 2 департамента | Ждет, пока блокировка снимется.                                       |
| `COMMIT`                                                          | Ждет, пока блокировка снимется.                                       |
|                                                                   | `UPDATE organization SET required_subscription = REGULAR`             |
|                                                                   | `INSERT INTO department...` - теперь в организации **3** департамента |
|                                                                   | `COMMIT`                                                              |

Проблема в том, что каждый поток проверял выполнение условия бизнес-кейса в
методе `Organization.createOrganization` только в рамках своего потока. Но не учитывался факт того,
что кто-то может вклиниться и выполнить такую же операцию параллельно.

В итоге у нас 3 `Department` в рамках `Organization`, но `requiredSubscription` равен `REGULAR`.
Хотя по условию, которое мы выставили в `Organization.createOrganization`, уровень в этом случае
должен быть равен `GOLD`.

Есть два способа решить эту проблему.

## Pessimistic locking

Самый простой вариант - начать блокировку раньше, чем при выполнении операции `UPDATE`. То есть в
момент выборки `Organization` по ID. В прошлом модуле мы выяснили, что за это отвечает
выражение `FOR UPDATE`. Посмотрите на пример того, как это можно написать на Java:

```java
public class OrganizationRepositoryImpl implements OrganizationRepository {

    private final Jdbi jdbi;

    /* конструктор + остальные методы */

    @Override
    public Organization findByIdForUpdate(UUID organizationId) {
        return jdbi.inTransaction((Handle handle) -> {
            Map<String, Object> orgMap =
                // FOR UPDATE блокирует запись для нас до конца жизни транзакции
                handle.createQuery(
                        "SELECT id, name, required_subscription FROM organization WHERE id = :id FOR UPDATE"
                    )
                    .bind("id", organizationId)
                    .mapToMap()
                    .first();
            List<Map<String, Object>> departmentsMapList =
                handle.createQuery("SELECT * FROM department WHERE organization_id = :orgId")
                    .bind("orgId", organizationId)
                    .mapToMap()
                    .stream()
                    .toList();
            return new Organization(/* маппинг результатов к объекту Organization */);
        });
    }

    /* остальные методы репозитория */
}
```

Тогда код в сервисном слое может выглядеть так:

```java
public class OrganizationService {

    private final OrganizationRepository organizationRepository;
    private final TransactionManager transactionManager;

    public void createDepartment(UUID organizationId, String name, String description) {
        transactionManager.useTransaction(() -> {
            Organization organization = organizationRepository.findByIdForUpdate(organizationId);
            organizationRepository.update(
                organization.createDepartment(name, description)
            );
        });
    }
}
```

Теперь проблемы, о которой мы писали выше, не возникнет. Потому что в случае одновременного запроса
от двух клиентов ситуация будет такой:

| Поток клиента 1                                                   | Поток клиента 2                                                                                                                                                  |
|-------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `SELECT * FROM organization WHERE id = ? FOR UPDATE`              | Ждет, пока блокировка снимется.                                                                                                                                  |
| `UPDATE organization SET required_subscription = REGULAR`         | Ждет, пока блокировка снимется.                                                                                                                                  |
| `INSERT INTO department...` - теперь в организации 2 департамента | Ждет, пока блокировка снимется.                                                                                                                                  |
| `COMMIT`                                                          | Ждет, пока блокировка снимется.                                                                                                                                  |
|                                                                   | `SELECT * FROM organization WHERE id = ? FOR UPDATE` - здесь получаем организацию уже с 2 департаментами, потому что предыдущая транзакция закоммитила изменения |
|                                                                   | `UPDATE organization SET required_subscription = GOLD`                                                                                                           |
|                                                                   | `INSERT INTO department...` - теперь в организации 3 департамента, но `required_subscription` выставлен правильно.                                               |
|                                                                   | `COMMIT`                                                                                                                                                         |

Проще говоря, запросы на обновление `Organization` теперь выполняются последовательно, поэтому мы
гарантируем соблюдение инвариантов.

Главный плюс Pessimistic Locking - простота разработки. Минус же - проблемы
с [доступностью](https://en.wikipedia.org/wiki/Reliability,_availability_and_serviceability#:~:text=Availability%20means%20the%20probability%20that,time%20it%20should%20be%20operating.)
(или _availability_). Представьте, что кто-то решил обновить только название организации, но не трогать
количество департаментов. Или же обновить название департамента, но не менять их количество. Здесь
не возникнет проблем с правильным значением для `required_subscription`, но при повсеместном
использовании Optimistic Locking каждый из этих запросов будет ждать завершения другого, что
увеличит общее время ответа.

В рамках финального задания, а также практических работ нет ничего плохого в использовании
Pessimistic Locking: вы не наткнетесь на проблемы из-за этого. Тем не менее, мы расскажем вам про
альтернативу.

## Optimistic Locking

В прошлом модуле мы рассмотрели концепцию Optimistic Locking. Разберемся, как реализовать ее в
приложении для решения обозначенной проблемы.

### Уровень изоляции REPEATABLE READ

Первый и самый простой вариант - поменять уровень изоляции транзакции на `REPEATABLE_READ`. Для
этого немного доработаем `TransactionManager`:

```java
import org.jdbi.v3.core.transaction.TransactionLevel;

public interface TransactionManager {

    <R> R inTransaction(TransactionIsolationLevel level, Supplier<R> supplier);

    void useTransaction(TransactionIsolationLevel level, Runnable runnable);
}

public class JdbiTransactionManager {

    private final Jdbi jdbi;

    /* конструктор... */

    @Override
    public <R> R inTransaction(TransactionIsolationLevel level, Supplier<R> supplier) {
        return jdbi.inTransaction(level, (Handle handle) -> supplier.get());
    }

    @Override
    public void useTransaction(TransactionIsolationLevel level, Runnable runnable) {
        jdbi.useTransaction(level, (Handle handle) -> runnable.run());
    }
}
```

> Здесь мы немного нарушаем принципы абстракции, потому что в интерфейсе `TransactionManager` явно
> импортится класс `TransactionLevel`,
> который поставляется с библиотекой `JDBI`. Как вариант, можно объявить отдельный `enum` и
> конвертировать значения между ними.
> Но для упрощения мы этого не делаем.

Тогда код в сервисном слое будет выглядеть так:

```java
public class OrganizationService {

    private final OrganizationRepository organizationRepository;
    private final TransactionManager transactionManager;

    public void createDepartment(UUID organizationId, String name, String description) {
        transactionManager.useTransaction(TransactionLevel.REPEATABLE_READ, () -> {
            Organization organization = organizationRepository.findById(
                organizationId); // больше не нужен FOR UPDATE
            organizationRepository.update(
                organization.createDepartment(name, description)
            );
        });
    }
}
```

Обратите внимание, что теперь `Organization` по ID мы находим через обычный `SELECT`
без `FOR UPDATE`. В явной блокировке более нет смысла, потому что в момент коммита транзакции,
благодаря уровню `REPEATABLE_READ`, БД сама проверит, успел ли кто-то поменять в БД одну из строк,
которые мы затронули в рамках транзакции. Если да, то кинет исключение.

Поскольку паттерн Optimistic Locking предполагает возможное завершение операции с ошибкой, нам нужен
настроить retry, чтобы попытаться выполнить операцию несколько раз в рамках запроса от клиента.
Можно написать утилиту самим, но воспользуемся для этого
готовой [библиотекой Resilience4j](https://resilience4j.readme.io/docs/getting-started). Она
включает в себе много инструментов, но нас интересует только Retry. Сначала добавим зависимость:

```xml

<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-retry</artifactId>
    <version>2.0.2</version>
</dependency>
```

В случае нарушения Optimistic Locking при уровне изоляции `REPEATABLE_READ` JDBI бросит
исключение `org.jdbi.v3.core.statement.UnableToExecuteStatementException`. Так что его мы и будем
отлавливать.

Можно добавить retry напрямую в `OrganizationService`, но как вы догадываетесь, это не очень
правильный подход с точки зрения архитектуры. Нет гарантии, что нам всегда будет нужен retry. Так
что нецелесообразно так зашивать его в код. Решение - паттерн декоратор, который мы разбирали ранее
в курсе.

Сначала вынесем `OrganizationService` в отдельный интерфейс. Реализация `OrganizationServiceImpl`
будет выполнять код, который мы написали выше. Декоратор же `RetryableOrganizationService` выглядит
так:

```java
import io.github.resilience4j.retry.Retry;

public class RetryableOrganizationService implements OrganizationService {

    private final OrganizationService delegate;
    private final Retry retry;

    @Override
    public void createDepartment(UUID organizationId, String name, String description) {
        return retry.executeSupplier(
            () -> delegate.createDepartment(organizationId, name, description)
        );
    }
}
```

Конфигурация retry при создании объекта `RetryableOrganizationService` может выглядеть следующим
образом:

```java
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;

import org.jdbi.v3.core.statement.UnableToExecuteStatementException;

public class Main {

    public static void main(String[] args) {
        RetryableOrganizationService retryableOrganizationService = new RetryableOrganizationService(
            new OrganizationServiceImpl(/* конфигурация OrganizationServiceImpl */),
            Retry.of(
                "retry-db",
                RetryConfig.custom()
                    .maxAttempts(3)
                    .retryExceptions(UnableToExecuteStatementException.class)
                    .build()
            )
        );
        /* дальнейшие инициализации... */
    }
} 
```

> Возможно, вы уже догадались, что `TransactionManager` правильнее было вынести в декоратор подобным образом,
> а не добавлять его напрямую в `OrganizationServiceImpl`. Но чтобы не усложнять код, мы этого не делаем.
> В работе, если вы внедрите `TransactionManager`, допускается поступать также.

### Ручная реализация паттерна

Хотя подход с выставлением уровня изоляции удобен, у него есть недостаток. БД автоматически будет
проверять все записи, которые вы обновили подобным образом. Это не всегда нужно: будут
ложно-положительные срабатывания. Поэтому рассмотрим другой вариант с использованием колонки версии.

Мы уже рассматривали этот подход в прошлом модуле на примере SQL. Теперь посмотрим, как его
имплементировать в реальном Java-приложении. Сначала добавьте в вашу таблицу числовую колонку,
которая в Java будет маппиться на тип `long` (обычно такую колонку называют `version`).

Теперь немного отредактируем сущность `Organization`:

```java
public class Organization {

    private final UUID id;
    private final String name;
    private final List<Department> departments;
    private final Subscription requiredSubscription;
    private final long version;

    /* constructor, equals, hashCode */

    public static Organization newOrganization(String name) {
        return new Organization(UUID.randomUUID(), name, emptyList(), Subscription.REGULAR, 0);
    }

    public Organization createDepartment(String name, String description) {
        /* логика создания нового department */
    }
}
```

При создании новой `Organization` поле `version` равно `0`. Но когда мы вычитываем его из БД через
репозиторий, то заполняем тем значением, которое на самом деле хранится в строчке таблицы.

Теперь снова вернемся к методу `OrganizationRepository.update`. Внесем в него небольшие исправления:

```java
public class OrganizationRepositoryImpl implements OrganizationRepository {

    private final Jdbi jdbi;

    /* конструктор + остальные методы */

    @Override
    public void update(Organization organization) {
        jdbi.useTransaction((Handle handle) -> {
            int updatedRowsCount = handle.createUpdate(
                    // инкрементируем версию, если никто этого не делал ранее
                    "UPDATE organization SET name = :name, required_subscription = :sub, version = version + 1"
                    + "WHERE id = :id AND version = :version"
                )
                                       .bind("name", organization.getName())
                                       .bind("sub", organization.getRequiredSubscription())
                                       .bind("id", organization.getId())
                                       .bind("version", organization.getVersion())
                                       .execute();
            if (updatedRowsCount == 0) {
                throw new OptimisticLockingException(
                    "Couldn't update Organization due to Optimistic Locking failure"
                );
            }
            /* логика создания/обновления Department не меняется с предыдущего примера */
        });
    }
}
```

В список полей, которые мы меняем в `organization`, добавилась `version`. Мы просто инкрементируем
значение на 1. То есть каждое обновление сопровождается увеличением версии. Интерес же скрыт в
блоке `WHERE`. Помимо значения `id`, мы проверяем еще и `version`. Иначе говоря, мы пытаемся найти
строчку, у которой заданный `id` **И** поле `version` равно тому значению, которое мы заполнили в
сущности `Organization` при старте бизнес-операции
(то есть при чтении строчки `organization` из БД).

Если кто-то успел вклиниться и обновить `organization` до того, как мы выполнили `UPDATE`, но после
этого, как мы уже прочитали сущность `Organization`, то наш `UPDATE` не сработает. Потому что `id`
сущности остался таким же, а вот значение `version` поменялся (другой поток успел его
заинкрементировать). Соответственно, строчка в БД не будет найдена по условию `WHERE`, и `UPDATE` не
выполнится.

Хорошо, мы поняли, как предотвратить нежелательную операцию. Но как мы узнаем о том, выполнился
ли `UPDATE` в данном случае или нет? Как вариант, вы можете сделать повторный `SELECT`, но уже
только по `id`, и посмотреть, поменялось ли значение или нет. Тем не менее, спецификация JDBC дает
более простое решение.

Заметили, что `handle.createUpdate` возвращает значение `int`? Это количество строчек в таблице,
которое получилось обновить данным `UPDATE`-ом. Если мы не нашли нужной строчки по комбинации `id`
и `version`, то вернется `0`. Следовательно, чтобы определить факт нарушения Optimistic Locking,
достаточно сравнить значение с `0` и кинуть исключение, если что-то пошло не так.

> Здесь `OptimisticLockingException` - это некоторое кастомное исключение.

Теперь можно пересмотреть подход к retry'ам. Посмотрите на пример кода ниже:

```java
public class RetryableOrganizationRepository implements OrganizationRepository {

    private final OrganizationRepository delegate;
    private final Retry retry;

    /* конструктор + остальные методы */

    @Override
    public void update(Organization organization) {
        return retry.executeSupplier(
            () -> delegate.update(organization)
        );
    }
}

public class OrganizationService {

    private final OrganizationRepository organizationRepository;

    public void createDepartment(UUID organizationId, String name, String description) {
        Organization organization = organizationRepository.findById(organizationId);
        organizationRepository.update(
            organization.createDepartment(name, description)
        );
    }
}
```

В `OrganizationService` больше нет необходимости оборачивать и запрос `SELECT` и `UPDATE` в одну
транзакцию. Pessimistic Locking мы теперь не используем, а благодаря Optimistic Locking в
методе `update` будет выполнена нужная проверка на concurrency access + retry.