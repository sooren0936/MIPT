# Подключаем Spring Data JPA и Hibernate

Пришло время добавить поддержку работы с БД в наше приложение.

Добавим в `pom.xml` нашего проекта следующую зависимость:

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

Этот стартер подключает сконфигурированную и почти готовую к работе комбинацию Spring Data JPA +
Hibernate.

Теперь нам нужно настроить подключения к серверу БД. Ниже приводим инструкцию для наиболее часто
используемых серверов с открытым исходным кодом MySQL и PostgreSQL. Вы можете выбрать любой из них.

## Подключение к серверу PostgreSQL

Начнем с установки локального сервера реляционной БД PostgreSQL, который можно
скачать [здесь](https://www.postgresql.org/download/).

Для подключения нам понадобится JDBC-драйвер для PostgreSQL. Он добавляется так:

```xml

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

В прошлом семестре мы с вами рассмотрели Flyway как инструмент миграции для реляционных БД.
Давайте сразу добавим его в зависимости:

```xml

<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

Также перед запуском требуется добавить ряд настроек в файл `application.properties`:

```
spring.datasource.url=jdbc:postgresql://localhost:5432/courses_app
spring.datasource.username=postgres
spring.datasource.password=password

spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.hibernate.ddl-auto=validate
```

Здесь мы указываем:

* строку подключения JDBC к базе данных с именем `courses_app`.
* имя пользователя и пароль для подключения к БД
* имя класса JDBC-драйвера
* настройка `spring.jpa.show-sql` отвечает за вывод в консоль всех запросов, которые Hibernate
  отправляет в БД, а `spring.jpa.properties.hibernate.format_sql` красиво форматирует эти запросы
* настройка `spring.jpa.hibernate.ddl-auto=validate` означает, что при запуске Hibernate проверит,
  что таблицы, которые присутствуют в БД, соответствуют описанным сущностям. В противном случае
  приложение завершится ошибкой при старте.

У настройки `spring.jpa.hibernate.ddl-auto` есть ещё несколько возможных значений:

* `update` — Hibernate проверит, есть ли в БД нужные таблицы; если таблиц нет, то они будут созданы.
* `create-drop` — Похоже на `create` с тем отличием, что Hibernate удалит схему БД после завершения
  всех операций (при закрытии контекста Spring). Это значение по умолчанию при использовании
  встроенной БД.
* `none` — При запуске приложения не выполнять никаких действий. Это значение по умолчанию при
  использовании не встроенной БД.

> Может сложиться впечатление, что нам не нужен Flyway, раз Hibernate умеет сам создавать таблицы с
> помощью
> `create`, `create-drop` или `update`. Но эти опции крайне не рекомендуется использовать в
> production, потому что их поведение может быть неожиданным. Например, при использовании `update`
> Hibernate может удалить колонку, если вы просто уберете поле из сущности.
>
> Поэтому мы сразу настроим Flyway, чтобы избежать подобных проблем.

По умолчанию скрипты Flyway будут вычитываться из директории `src/main/resources/db/migration`.
Никаких дополнительных настроек не требуется. Flyway запуститься автоматически при старте
приложения.

Перед запуском приложения нужно будет создать БД с именем `courses_app`. Для этого подключитесь к
серверу Postgres из командной строки, набрав:

```bash
psql -U postgres
```

После подключения введите команду создания БД:

```sql
create database courses_app;
```

Теперь БД готова к использованию.

> Вместо командной строки вы можете использовать GUI клиент для подключения к PostgreSQL.
> Например, [PGAdmin](https://www.pgadmin.org/) или [DBeaver](https://dbeaver.io/).

Также стоит отметить, что при запуске приложения на реальном сервере, параметры подключения к базе
данных могут отличаться от заданных в `application.properties`. Поэтому Spring предоставляет
возможность переопределять значения параметров при запуске.

```shell
java -jar my-app.jar --spring.datasource.url=jdbc:postgresql://localhost:1234/my_database --spring.datasource.username=some_user --spring.datasource.password=some_password
```

`my-app.jar` - это скомпилированное Spring-приложение. В данном случае мы переопределяем
параметры `spring.datasource.url`,
`spring.datasource.username` и `spring.datasource.password`.

Можно запомнить значения параметров в переменных окружения. Тогда строка запуска будет выглядеть
так:

```shell
java -jar my-app.jar --spring.datasource.url=$MY_JDBC_URL --spring.datasource.username=$MY_USER --spring.datasource.password=$MY_PASSWORD
```

## Доработка кода приложения

Теперь пришло время превратить класс `Course` из обычного класса в управляемую Spring Data и
Hibernate сущность. Для этого нам нужно выполнить ряд требований:

* над классом сущности должна быть аннотация `@Entity`
* в классе сущности должно быть поле с уникальным идентификатором, над таким полем должна быть
  аннотация `@Id`
* у класса обязательно должен быть конструктор по умолчанию
* класс сущности не может быть inner-классом
* класс сущности не может быть `final` или `abstract` классом

Помимо обязательных требований есть ряд дополнительных, которые просто делают работу с БД удобней:

* к аннотации `@Entity` рекомендуется добавить аннотацию `@Table`, при помощи которой можно указать
  имя таблицы для сущности
* над полем уникального идентификатора вместе с аннотацией `@Id` следует указать
  аннотацию `@GeneratedValue` – она позволяет настроить автоматическую генерацию идентификаторов для
  создаваемых сущностей
* над всеми полями, которые должны сохраняться в БД, рекомендуется указать аннотацию `@Column`

С учетом этих требований наш класс примет вид:

```java

@Entity
@Table(name = "courses")
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull(message = "Course author have to be filled")
    private String author;

    @NotNull(message = "Course title have to be filled")
    private String title;

    protected Course() {
    }

    public Course(String author, String title) {
        this.author = author;
        this.title = title;
    }

    // getters
}
```

> Конструктор без аргументов мы объявляем как `protected`, чтобы бизнес-код, который находится в
> другом package,
> мог создать инстанс `Course` только валидным через конструктор со всеми аргументами.

Аннотация `@Id` указывает, что данное поле является первичном ключом. Это некоторое поле, которое
должно быть уникально для каждой строки в базе данных. Наличие этого поля обязательно для сущности
Hibernate.

`@GeneratedValue` в свою очередь говорит о том, что значение для ID должно быть сгенерировано.

Поэтому у нас и нет метода `setId` в классе `Course`. Hibernate предоставляет различные стратегии
генерации (можно даже писать собственные). В данном случае мы используем `IDENTITY`. Это значит,
что при операции `insert` Hibernate просто не будет передавать никакого значения для поля ID,
рассчитывая, что база данных подставит его сама. В случае MySQL, для этого тип колонки должен
быть [AUTO_INCREMENT](https://www.w3schools.com/sql/sql_autoincrement.asp). В случае же PostgreSQL,
вариантов несколько, но самый простой -
использовать [SERIAL или BIGSERIAL](https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-serial/).

Ниже представлен пример Flyway-миграции для создания таблицы для сущности `Course`:

```sql
CREATE TABLE courses
(
    id     BIGSERIAL PRIMARY KEY,
    author TEXT NOT NULL,
    title  TEXT NOT NULL
)
```

> По умолчанию Hibernate считает, что названия колонок равны названиям полей в сущности.
> Наименование колонки можно указать явно с помощью аннотации `@Column`.

Теперь доработаем класс репозитория. Тут Spring Data очень сильно упрощает нам жизнь. Начнем с того,
что удалим класс `InMemoryCourseRepository`, а интерфейс `CourseRepository` изменим следующим
образом:

```java

@Repository
public interface CourseRepository extends JpaRepository<Course, Long> {

    List<Course> findByTitleLike(String title);
}
```

Этого достаточно, чтобы на основе объявленного интерфейса Spring Data создала полноценный бин
репозитория. Отметим также, что для метода `findByTitleLike` будет автоматически создана реализация,
которая будет выполнять SQL запрос вида:

```sql
select *
from title t
where t.title like :title
```

Здесь `:title` - значение параметра метода. Подробнее о том, как использовать методы с
автогенерящейся реализацией запроса, будет дальше.

Пришло время запустить приложение. Перед этим следует убедиться, что запущен сервер БД. Проверьте,
всё ли работает.

## Изменение properties в тестах

Порой в тестах нужно использовать другие значения настроек, описываемых в
файле `aplication.properties`. Для этих целей можно создать файл `aplication.properties` в
каталоге `src/test/resources` и указать там все необходимые значения.

Например, чтобы при выполнении unit-тестов Liquibase не выполнял скрипты миграции, мы можем создать
файл `src/test/resources/aplication.properties`, скопировав в него всё
из `src/main/resources/aplication.properties`
и добавить настройку, отключающую Liquibase:

```shell
spring.flyway.enabled=false
```

Альтернативный вариант - создать файл `src/test/resources/aplication-test.properties` и указать в
нём только те настройки, которые отличаются от `src/main/resources/aplication.properties`. В нашем
случае созданный файл будет содержать только одну строку:

```shell
spring.flyway.enabled=false
```

Поскольку в названии этого файла присутствует суффикс `-test`, Spring Boot применит его только при
активном профиле `test`. Для запуска unit-тестов в профиле `test` используется
аннотация `@ActiveProfiles`:

```java

@SpringBootTest
@ActiveProfiles("test")
class ApplicationTest {

    @Test
    void contextLoads() {

    }
}
```

Также можно переопределять properties в самой аннотации `@SpringBootTest`:

```java

@SpringBootTest(properties = {"spring.flyway.enabled=false"})
class ApplicationTest {

    /* some tests */
}
```

## Open session in view

Перед тем как продолжать работу над вашим приложением,
откройте `src/main/resources/application.properties` и поставьте
параметр `spring.jpa.open-in-view=false`.

Дело в том, что при использовании Spring Data JPA фреймворк по умолчанию открывает соединение с БД
на каждый HTTP-запрос, даже если явно к БД вы не обращаетесь. С одной стороны, это упрощает работу.
С другой же, это приводит к неэффективному использованию ресурсов БД, а также к неожиданнам багам в
коде, если этот параметр выключить на поздней стадии.

Чтобы не иметь с этим проблем в будущем, рекомендуем выключить его сразу. Подробнее о том, что
плохого в open session in view, можете [почитать здесь](https://habr.com/ru/articles/440734/).

## Валидация с помощью аннотаций

Вы, наверное, обратили внимание, что мы использовали аннотацию `@NotNull`. Посмотрите еще раз:

```java

@Entity
@Table(name = "courses")
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull(message = "Course author have to be filled")
    private String author;

    @NotNull(message = "Course title have to be filled")
    private String title;

    // getters, setters, constructors
}
```

Очень важно понимать, что эти аннотации **не относятся** к проверкам в БД. Они выполняют здесь ту же
роль, что и в примерах с request/response из предыдущего модуля. То есть валидируют сами поля
объекта без привязки к тому, что они потом будут преобразованы в SQL-выражения.

Если конкретно, валидатор запускается в двух случаях: при сохранении и чтении данных. В первом
случае, значения будут проверены на валидность аннотациям **перед** генерацией `INSERT`
или `UPDATE`. В случае же чтения данных Hibernate запустит валидатор после того, как создаст
экземпляр сущности на основе полученных из БД данных.

Ставить их или нет – выбор ваш. Если у вас есть проверка на `not null` на уровне БД, вы все равно не
сможете сохранить невалидные данные. Тем не менее аннотации срабатывают даже до фактического
обращения к БД, что может немного ускорить работу.