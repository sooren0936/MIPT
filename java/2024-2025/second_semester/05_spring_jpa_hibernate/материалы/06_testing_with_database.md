# Интеграционное тестирование с реальной БД

Написание unit-тестов является важной частью процесса разработки ПО. Однако, к сожалению, с их
помощью нельзя покрыть все тест-кейсы и быть уверенным в корректной работоспособности программы.

Например, предположим, что у нас есть метод репозитория, который возвращает уникальные названия
существующих курсов.

```java
public interface CourseRepository extends JpaRepository<Course, Long> {

    @Query("select distinct c.title from Course c;")
    Set<String> findCoursesTitles();
}
```

Как нам протестировать эту функциональность? Конечно, можно с помощью моков предположить, как
поведет себя программа. Однако это наивный подход. Мы хотим знать точно, работает ли наш запрос или
нет. А для этого нужно отправить его в базу и посмотреть результат.

Использование Embedded Database - самый простой способ запуска базы в тестовом окружении. Для этого
можно
использовать [H2 Database](https://www.h2database.com/html/main.html). Мы рассматривали вариант с
применением этой БД в прошлом семестре и сошлись на том, что в современных проектах лучше
использовать [Testcontainers](https://testcontainers.com/). Так что не будем останавливаться на
варианте с H2 и сразу двинемся далее.

## Testcontainers

Как можно догадаться, для ее работы требуется Docker, установленный на локальной машине.

> Если нет возможности установить Docker, Testcontainers позволяет соединяться с Docker daemon на
> удаленном хосте.
> Подробнее можно почитать по [этой ссылке](https://www.testcontainers.org/features/configuration/).

Если мы хотим поднять в Docker PostgreSQL, нужно добавить соответствующие зависимости.

```xml

<dependencies>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>1.19.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

Теперь напишем сам тест:

```java

@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class CourseRepositoryTest {
  ...
}
```

`@DataJpaTest` поднимает ту часть Spring-контекста, которая связана со слоем доступа к
данным (`DataSource`, repositories, `EntityManager` и так далее).
Также все тесты, объявленные с помощью этой аннотации, по умолчанию выполняются транзакционно.
Причем по завершении транзакция откатывается автоматически.
Это кажется удобным, однако может привести к неожиданным багам и странным эффектам.
Их подробное обсуждение выходит за рамки этого курса,
поэтому мы просто отключаем транзакционное выполнение тестов с помощью следующей
аннотации - `@Transactional(propagation = Propagation.NOT_SUPPORTED)`.

> Если вам интересно узнать об это подробнее, рекомендуем ознакомиться со следующими статьями:
>
> 1. [Spring Data — Never Rollback Readonly Transactions](https://dev.to/kirekov/spring-never-rollback-readonly-transactions-28kb)
> 2. [Spring Data — Transactional Caveats](https://dev.to/kirekov/spring-data-transactional-caveats-19di)

`@AutoConfigureTestDatabase(replace = Replace.NONE)` нужна, чтобы сказать Spring не использовать
тестовую БД.
Дело в том, что `@DataJpaTest` по умолчанию настраивает тестовую базу данных, даже если мы в
конфигурации указали использовать Testcontainers.
Чтобы устранить это поведение, мы явно даем указание не заменять базу тестовой.

> В качестве тестовой БД Spring использует H2, если она есть в зависимостях. Если ее нет, запуск просто
> приведет к ошибке.

Теперь давайте настроим запуск PostgreSQL через Testcontainers:

```java

@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class CourseRepositoryTest {

    @Container
    public static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:13");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }

    // tests
}
```

Назначение аннотации `@Testcontainers` и `@Container` мы уже рассматривали в прошлом
семестре. `@DynamicPropertySource` нужна для того, чтобы перед стартом Spring контекста выставить
нужные конфиги. В данном случае это url, user и password БД, которая была запущена.

Начиная со Spring Boot 3 этот код можно упростить. Для этого добавьте зависимость:

```xml

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

А теперь поправим сам тест:

```java

@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class CourseRepositoryTest {

    @Container
    @ServiceConnection
    public static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:13");

    // tests
}
```

Аннотация `@ServiceConnection` - это указания для Spring, что после старта контейнера нужно
автоматически переопределить нужные конфиги.
На основе того, какой именно это контейнер, Spring сам понимает, какие конфиги нужно выставить.

Теперь достаточно проинжектировать в тест `CourseRepository` через `@Autowired`, чтобы проверить
работоспособность решения.

Если вы будете добавлять переменную `POSTGRES` в каждый тестовый класс, то в каждом из них будет
запускаться новый контейнер.
Чтобы переиспользовать один контейнер и ускорить тесты, можно выделить для этого базовый класс.
Правда тогда нам придется отказаться от автоконфигураций Spring-а и написать немного кода вручную.
Посмотрите на пример кода ниже:

```java

@ContextConfiguration(initializers = DatabaseSuite.Initializer.class)
public class DatabaseSuite {
    private static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:13");

    static class Initializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {

        @Override
        public void initialize(ConfigurableApplicationContext context) {
            Startables.deepStart(POSTGRES).join();

            TestPropertyValues.of(
                "spring.datasource.url=" + POSTGRES.getJdbcUrl(),
                "spring.datasource.username=" + POSTGRES.getUsername(),
                "spring.datasource.password=" + POSTGRES.getPassword()
            ).applyTo(context);
        }
    }
}

@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
@AutoConfigureTestDatabase(replace = Replace.NONE)
class CourseRepositoryTest extends DatabaseSuite {
    // tests
}
```

Здесь метод `DatabaseSuite.Initializer.initialize` вызывается Spring-ом перед стартом контекста. Тут
мы самостоятельно запускаем контейнер и переопределяем конфиги.

## Тестирование сервисов

Предположим, что мы хотим протестировать уровень сервисов, а не репозиториев. Посмотрите на пример
ниже:

```java

@Service
public class CourseService {

    private final CourseRepository courseRepository;

    @Transactional
    public void updateCourseName(Long courseId, String newName) {
        Course course = courseRepository.findById(courseId).orElseThrow();
        course.setName(newName);
        courseRepository.save(course);
    }
}
```

Если мы установим те же самые настройки для тестов, что мы делали ранее, и попробуем заинжектить `CourseService`, то получим ошибку.
Дело в том, что `CourseService` – это наш кастомный Spring bean. Аннотация же `@DataJpaTest` сканирует только то, что относится к БД.
К счастью, это легко исправить:

```java

@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Import({CourseService.class})
class CourseServiceTest extends DatabaseSuite {
    @Autowired
    private CourseService courseService;
    
    // tests
}
```