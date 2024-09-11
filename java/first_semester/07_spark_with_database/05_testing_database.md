# Тестирование работы с БД

В прошлых уроках мы создали три таблицы: `organization`, `department` и `employee`. Давайте напишем
простой класс, который позволяет добавлять новые записи в `organization` и получать существующие по
ID:

```java
public class Organization {

   private final long id;
   private final String name;

   /* constructor, equals, hashCode */
}

public class OrganizationRepository {

  private final Jdbi jdbi;

  public OrganizationRepository(Jdbi jdbi) {
    this.jdbi = jdbi;
  }

  public long createOrganization(String name) {
    return jdbi.inTransaction((Handle handle) -> {
      ResultBearing resultBearing = handle.createUpdate(
              "INSERT INTO organization (name) VALUES (:name)")
          .bind("name", name)
          .executeAndReturnGeneratedKeys("id");
      Map<String, Object> mapResult = resultBearing.mapToMap().first();
      return ((Long) mapResult.get("id"));
    });
  }

  public Organization findById(long id) {
    return jdbi.inTransaction((Handle handle) -> {
      Map<String, Object> result =
          handle.createQuery("SELECT id, name FROM organization WHERE id = :id")
              .bind("id", id)
              .mapToMap()
              .first();
      return new Organization(
          ((Long) result.get("id")),
          ((String) result.get("name"))
      );
    });
  }
}
```

Разберемся по порядку, что здесь происходит. Сначала мы добавили структуру данных `Organization`,
экземпляр которой будем возвращать по запросу `findById`. Далее идет метод `createOrganization`. В
нем мы принимаем на вход название организации, создаем ее с помощью `INSERT` и возвращаем ID. Здесь
следует обратить внимание на метод `executeAndReturnGeneratedKeys`. Тип колонки `id` в БД мы
объявили как `BIGSERIAL`. То есть значение подставится в момент вставки новой записи, но до этого мы
его не знаем. Протокол JDBC в курсе этой особенности и позволяет возвращать в качестве ответа то
значение, которое база данных сгенерировала (ID, в нашем случае).

> Далее в модуле мы рассмотрим и другие способы работы с ID, подходящие для применения в коде на Java.

Последнее - метод `findById`. Здесь все просто: находим нужную строчку по ID и возвращаем ответ,
обернутый в экземпляр `Organization`.

На этом моменте возникает вопрос: как это все протестировать? На ум приходят два варианта:

1. Замокать зависимость `Jdbi` и написать обычный unit-тест.
2. Развернуть БД локально, создать в тестах экземпляр `OrganizationRepository`, который подключится
   к ней, и проверить работоспособность.

Первый вариант мы отметаем сразу, потому что в нем нет смысла. Даже если мы замокаем `Jdbi`, это не дает
нам никого понимания относительно работоспособности решения. Ведь тут мы отправляем SQL-запросы и
ожидаем получить определенный ответ. Ошибок может быть много:

1. Неправильно составленный запрос.
2. Исключение, которое мы не отловили.
3. Забыли при вставке передать значение колонки, которая отмечена как `not null`.

Эти кейсы невозможно проверить с моками, потому что трудно предугадать ответ БД, пока мы не
попробуем выполнить запрос.

Второй вариант намного лучше. Здесь мы уже тестируем взаимодействие с настоящей базой данных. Тем не
менее, у этого решения тоже есть проблемы. Помните модуль, в котором мы разбирали тесты? Мы
упоминали, что тесты должны запускаться во время сборки Pull Request и не позволять вливать его,
если хотя бы один упал. Так и гарантируется корректность кода. Но достичь такого же эффекта с
тестом, который подключается к локальной БД, не получится.

Действительно, сборка Pull Request выполняется на удаленной машине, которая ничего не знает о БД на
нашем компьютере и не имеет к ней доступа. Исходя из этого, к тестам, которые проверяют
взаимодействие с БД, можно выдвинуть следующие требования:

1. Тесты должны проверять реальные запросы в БД, а не обращения к мокам.
2. Когда мы запускаем билд (хоть локально, хоть на CI), тесты должны прогоняться автоматически, как
   и любые другие.

Есть два решения этой проблемы. Рассмотрим каждое из них подробнее.

## База H2

Есть специальная БД для тестирования - [H2](http://www.h2database.com/html/tutorial.html). Она
эмулирует реляционные базы данных и поддерживает большинство фичей различных вендоров (PostgreSQL,
Oracle, MySQL). В то же самое время она хранит данные либо на локальном диске, либо в памяти. Ее
преимущество в том, что для запуска H2 не требуется использовать Docker или ставить ее отдельно как
приложение. Достаточно добавить зависимость, а Java запустит ее автоматически:

```xml

<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <version>2.2.220</version>
  <scope>test</scope>
</dependency>
```

Теперь напишем сам тест. Посмотрите на пример `OrganizationRepositoryTest` ниже:

```java
class OrganizationRepositoryTest {

  private static Jdbi jdbi;

  @BeforeAll
  static void beforeAll() {
    String h2Url = "jdbc:h2:~/test";
    Flyway flyway =
        Flyway.configure()
            .outOfOrder(true)
            .locations("classpath:db/migrations")
            .dataSource(h2Url, "sa", "")
            .load();
    flyway.migrate();
    jdbi = Jdbi.create(h2Url, "sa", "");
  }

  @BeforeEach
  void beforeEach() {
    jdbi.useTransaction(handle -> handle.createUpdate("DELETE FROM organization").execute());
  }

  @Test
  void shouldCreateNewOrganization() {
    OrganizationRepository repository = new OrganizationRepository(jdbi);

    long orgId = repository.createOrganization("new org");

    Organization organization = repository.findById(orgId);
    assertEquals("new org", organization.name());
    assertEquals(orgId, organization.id());
  }
}
```

Сначала в колбэке `beforeAll` (запускается один раз перед всеми тестами) мы настраиваем Flyway,
чтобы создать все нужные таблицы, и записываем настроенный инстанс `Jdbi` в статическую переменную.

Далее идет колбэк `beforeEach` (запускается перед каждым тестом). В нем мы удаляем содержимое
таблицы `organization`, чтобы тесты не влияли друг на друга и работали с "чистого листа".

> В нашем случае тест всего один. Но могут добавиться и новые. Так что лучше сразу учитывать детерминированность выполнения.

Теперь поговорим о тесте `shouldCreateNewOrganization`. Тут все тривиально:

1. Создаем экземпляр `OrganizationRepository` с настроенным `Jdbi`.
2. Обращаемся к репозиторию, чтобы создать организацию.
3. Находим организацию по полученному ID.
4. Проверяем, что ID и name равны ожидаемым значениям.

Запустите тест. Если вы всё сделали правильно, он будет зеленым.

У H2 довольно много плюсов:

1. Простая конфигурация. Нужно только добавить зависимость в `pom.xml`. Запуск будет произведен
   автоматически.
2. БД быстро запускается. Скорость выполнения тестов сопоставима с обычными unit-тестами.
3. Кроссплатформенность. Тесты будут работать гладко на любой платформе (в том числе, и на CI).
4. Не требуется установка дополнительного ПО.

Но несмотря на все преимущества, H2 для тестов в современной разработке стараются избегать. Причина
проста - эта база не поддерживает специфические фичи конкретного решения. Рассмотрим простой пример.
В PostgreSQL есть тип данных [jsonb](https://www.postgresql.org/docs/current/datatype-json.html),
который позволяет хранить в колонки JSON и даже писать SQL-запросы с выборкой оттуда конкретных
значений и фильтрацией по ним. Добавим новую Flyway миграцию с таблицей, где будет такой тип данных:

```sql
CREATE TABLE organization_settings
(
    id       BIGSERIAL PRIMARY KEY,
    settings JSONB NOT NULL
);
```

Запустите тесты повторно. К удивлению, они упадут. В логах же вы найдете следующее сообщение:

```
org.flywaydb.core.internal.command.DbMigrate$FlywayMigrateException: Migration V4__create_organization_settings.sql failed
-----------------------------------------------------
SQL State  : HY004
Error Code : 50004
Message    : Неизвестный тип данных: "JSONB"
Unknown data type: "JSONB"; SQL statement:
CREATE TABLE organization_settings
(
    id       BIGSERIAL PRIMARY KEY,
    settings JSONB NOT NULL
) [50004-220]
```

Причина проста: H2 не поддерживает тип данных JSONB. Так что если вы напишите программу так, что
этот тип вам понадобится, протестировать корректную работу с помощью H2 у вас не получится.

> Стоит сказать, что это распространяется не только на миграции, но и на обычные `SELECT/UPDATE/INSERT/DELETE` выражения.
> PostgreSQL поддерживает определенные конструкции, которые H2 выполнить не сможет. Значит, если они будут присутствовать в вашем коде,
> H2 воспримет их как невалидный SQL и вернет ошибку.

H2 может подойти для простых ситуаций, но в современном мире разработки этот подход нельзя назвать
удачным.

## Testcontainers

[Testcontainers](https://java.testcontainers.org/) - это библиотека для Java (есть порты и для
других языков), которая во время тестов позволяет запустить Docker-контейнеры и автоматически
удалить их по завершении сценария. В библиотеку уже встроены готовые обертки для популярных БД 
(PostgreSQL входит в их число). Проще говоря, во время теста мы будем поднимать не H2, которая
эмулирует базу данных, а настоящий экземпляр PostgreSQL в Docker-контейнере.

Прежде всего установите [Docker](https://www.docker.com/), если у вас его нет, на компьютер. Теперь
добавьте в проект зависимости:

```xml

<dependencies>
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.18.3</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.18.3</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.18.3</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

Первый артефакт - это core библиотеки Testcontainers. Второй предоставляет удобную интеграцию для
JUnit 5. Последний же - это контейнер для запуска PostgreSQL.

Напишем тест, который с помощью Testcontainers поднимет PostgreSQL в Docker и прогонит на нем тесты.
Посмотрите на пример ниже:

```java
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class OrganizationRepositoryTestcontainersTest {

  @Container
  public static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:13");

  private static Jdbi jdbi;

  @BeforeAll
  static void beforeAll() {
    String postgresJdbcUrl = POSTGRES.getJdbcUrl();
    Flyway flyway =
        Flyway.configure()
            .outOfOrder(true)
            .locations("classpath:db/migrations")
            .dataSource(postgresJdbcUrl, POSTGRES.getUsername(), POSTGRES.getPassword())
            .load();
    flyway.migrate();
    jdbi = Jdbi.create(postgresJdbcUrl, POSTGRES.getUsername(), POSTGRES.getPassword());
  }

  @BeforeEach
  void beforeEach() {
    jdbi.useTransaction(handle -> handle.createUpdate("DELETE FROM organization").execute());
  }

  @Test
  void shouldCreateNewOrganization() {
    OrganizationRepository repository = new OrganizationRepository(jdbi);

    long orgId = repository.createOrganization("new org");

    Organization organization = repository.findById(orgId);
    assertEquals("new org", organization.name());
    assertEquals(orgId, organization.id());
  }
}
```

Как вы можете заметить, тест очень похож на предыдущий, с H2, но различия присутствуют. Во-первых, это
аннотация `@Testcontainers` над классом. Она регистрирует
кастомный [JUnit Extension](https://www.baeldung.com/junit-5-extensions), который внутри вызывает
операции для запуска контейнеров.

Далее идет статическая переменная `POSTGRES`. Здесь мы создаем экземпляр контейнера и указываем
нужную версию. Если Docker Image отсутствует в локальном registry, Testcontainers автоматически
запросит его из Docker Hub при первом запуске. Аннотация же `@Container` нужна как раз в сочетании с
JUnit Extension, про который мы упомянули в предыдущем абзаце. Testcontainers по наличию
аннотации `@Container` понимает, что является контейнером, и требует запуска.

В колбэке `@BeforeAll` мы также настраиваем Flyway, но здесь мы не хардкодим JDBC URL, а
запрашиваем его напрямую у контейнера. Дело в том, что Testcontainers запускает контейнеры на
случайном доступном порту. Так что JDBC URL каждый раз будет отличаться.

Запустите тест. Если всё вы сделали правильно, он будет зеленым. Также обратите внимание на
миграцию с добавлением таблицы `organization_settings` из примера с H2 (если вы ее удалили, добавьте
заново). Вы заметили, что больше ошибок не возникает? Потому что теперь Flyway прогоняет скрипты не
на эмуляции БД H2, а на настоящем экземпляре PostgreSQL. Это дает нам возможность использовать любые
фичи той или иной БД в коде и всегда иметь возможность их протестировать.

Возможно, у вас возник вопрос: а как запускать эти тесты в CI-среде? Ведь в рамках тестов нам нужен
Docker. Допустимо ли такое? В случае GitHub Actions вам ничего дополнительно настраивать не нужно.
Работает "из коробки". Но если бы вы использовали какой-то другой провайдер для запуска билда вашего
проекта, то дополнительная конфигурация могла бы понадобиться. Как бы то ни было, разработчики
Testcontainers
предоставили [подробную инструкцию для такого случая](https://java.testcontainers.org/supported_docker_environment/continuous_integration/dind_patterns/).

> Хотим заметить, что главная фишка Testcontainers - запуск абсолютно любых контейнеров.
> Даже если в библиотеке нет готового решения, вы всегда [можете применить GenericContainer](https://java.testcontainers.org/features/creating_container/).

## Что лучше использовать?

Мы рекомендуем во всех ситуациях отдавать предпочтение Testcontainers, а H2 выбирать лишь в тех
случаях, когда на использование Testcontainers наложены какие-то технические ограничения. Но также
мы хотим заметить, что интеграционные тесты не должны превалировать над unit-ами. Применяйте
интеграционное тестирование в этих случаях:

1. Необходима прямая проверка взаимодействия со сторонней системой (базой данной, кэшем, очередью сообщений и
   так далее).
2. Необходим E2E-тест, который собирает всю матрешку объектов и проверяет корректность запроса от начала до
   конца.

При тестировании остальной функциональности лучше выбрать unit-тесты. Несмотря на свою
привлекательность, у интеграционных тестов есть ряд недостатков:

1. Они ощутимо медленнее, чем unit-тесты. Думаю, вы это заметили при первом запуске.
2. Интеграционные тесты сложнее писать и поддерживать.
3. Тесты, у которых есть один общий ресурс (база данных, в нашем случае), сложнее запускать
   параллельно, что также влияет на скорость не в лучшую сторону.
4. Если тесты делят один ресурc, то есть риск, что они могут неявно влиять друг на
   друга. [Такие тесты называют flaky](https://habr.com/ru/companies/jugru/articles/416757/). Это
   сценарии, которые иногда работают, а иногда - нет. Они вызывают больше всего недовольства, ведь
   сложно угадать, когда билд вдруг опять упадет на CI.

С другой же стороны, интеграционные тесты важны и полезны. Так что совсем от них отказываться не
стоит, иначе у вас будет часть кода, которая не протестирована. А поскольку она взаимодействует с
внешним ресурсом, отсутствие тестов здесь может быть наиболее критичным.