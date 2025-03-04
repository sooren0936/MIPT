# Язык запросов ORM

## Запросы через методы репозитория

Репозитории Spring Data предоставляют богатейший функционал для извлечения данных из БД, но иногда
его оказывается недостаточно для решения некоторых задач. В этом случае раньше нужно было сразу
переходить к использованию встроенного в Hibernate языка запросов HQL (он же JPQL). Однако в Spring
Data есть инструмент, который позволяет обойтись без непосредственного написания запросов для
довольно широкого круга задач. Наверняка вы помните метод `findByTitleLike ` в
репозитории `CourseRepository`, о котором мы обещали рассказать подробнее:

```java

@Repository
public interface CourseRepository extends JpaRepository<Course, Long> {

    List<Course> findByTitleLike(String title);
}

```

Как уже обсуждалось, для подобных методов Spring Data умеет самостоятельно создавать реализацию
SQL-запроса к БД. Давайте остановимся более подробно на правилах написания таких методов. Прежде
всего, их имена должны начинаться со слова `find`. Приведём некоторые ключевые слова, которые могут
быть использованы в названиях таких методов:

| Ключевое слово     | Пример                                                 | Вариант HQL-запроса                      |
|--------------------|--------------------------------------------------------|------------------------------------------|
| Is, Equals         | findByUsername, findByUsernameIs, findByUsernameEquals | ... where x.firstname = ?1               |
| LessThan           | findByAgeLessThan                                      | ... where x.age > ?1                     |
| LessThanEqual      | findByAgeLessThanEqual                                 | ... where x.age >= ?1                    |
| GreaterThan        | findByAgeGreaterThan                                   | ... where x.age < ?1                     |
| GreaterThanEqual   | findByAgeLessGreaterEqual                              | ... where x.age <= ?1                    |
| Between            | findByStartDateBetween                                 | ... where x.startDate between ?1 and ?2  |
| IsNull, Null       | findByAge(Is)Null                                      | ... where x.age null                     |
| IsNotNull, NotNull | findByAge(Is)NotNull                                   | ... where x.age not null                 |
| Like               | findByFirstnameLike                                    | ... where x.firstname like ?1            |
| NotLike            | findByFirstnameNotLike                                 | ... where x.firstname not like ?1        |
| IgnoreCase         | findByFirstnameIgnoreCase                              | ... where UPPER(x.firstname) = UPPER(?1) |

Также несколько условий можно комбинировать при помощи логических операций `And`, `Or` и `Not`,
например вот так: `findByLastnameOrFirstname(String lastname, String firstname);`. Предлагаю вам
поупражняться в составлении подобных методов, т.к. далее нам придется часто их использовать.

## Немного про HQL

Бывают ситуации, когда составленных подобным образом методов может быть недостаточно. Например, так
не получится решить проблему со списком пользователей в окне привязки пользователя к курсу. Сейчас
там всегда отображается полный список пользователей, и на самом деле можно дважды привязать один и
тот же курс к пользователю, к которому он уже привязан. Правильней было бы оставить в списке только
тех пользователей, к которым данный курс ещё не привязан. Попробуем написать запрос на HQL, при
помощи которого получим нужный список пользователей и разберемся в отличиях этого языка от SQL.

Прежде всего, в HQL мы оперируем не таблицами в БД, а классами сущностей, которые были объявлены в
нашем приложении. Давайте начнем с запроса, возвращающего `id` всех пользователей, к которым
привязан данный курс:

```sql
select u.id
from User u
         inner join u.courses c
where c.id = :courseId
```

В глаза должен бросаться оператор `inner join` без предложения `on` с указанием полей, по которым
происходит соединение. Так как в качестве параметра тут указано поле сущности `u.courses` (а о связи
таблицы `User` и `Course` Hibernate все известно), условие соединения можно опустить. Особенно это
удобно для отношения многие ко многим, так как мы можем опустить соединение с промежуточной
таблицей.

Теперь найдем пользователей, не входящих в список, который нам вернул написанный запрос. Проще всего
это сделать через подзапрос и оператор `in`:

```sql
select u
from User u
where u.id not in (select u.id
                   from User u
                            inner join u.courses c
                   where c.id = :courseId)
```

Ещё одна интересная особенность языка HQL заключается в том, что в нём иногда можно опускать
предложение `select`. Например, запрос `select u from User u` можно написать как `from User u`.
Аналогично можно поступить и с только что составленным запросом.

Теперь добавим его как метод в репозитории. Это делается при помощи аннотации `@Query`:

```java

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("from User u " +
        "where u.id not in ( " +
        "select u.id " +
        "from User u " +
        "left join u.courses c " +
        "where c.id = :courseId)")
    List<User> findUsersNotAssignedToCourse(long courseId);
}
```

По умолчанию параметр в запросе (`:courseId`) будет называться так же, как и переменная в методе. Но
это поведение можно изменить с помощью аннотации `@Param`:

```java

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("from User u " +
        "where u.id not in ( " +
        "select u.id " +
        "from User u " +
        "left join u.courses c " +
        "where c.id = :course_id)")
    List<User> findUsersNotAssignedToCourse(@Param("course_id") long courseId);
}
```

## Проекции

HQL предоставляет удобное решение для создания DTO-классов непосредственно при выполнении запросов.
Этот механизм в Hibernate называют проекциями. Например, преобразование списка `Lesson` в
список `LessonDto` можно сделать вот так:

```java

@Repository
public interface LessonRepository extends JpaRepository<Lesson, Long> {

    @Query("select new com.example.dto.LessonDto(l.id, l.title, l.text, l.course.id) " +
        "from Lesson l")
    List<LessonDto> findAllWithProjection();
}
```

Обратите внимание, что здесь необходимо указывать полное имя класса со всеми пакетами. Попробуйте
заменить преобразование через Stream API на использование проекции.

Проекции удобно использовать не только для создания DTO, но и в любой другой ситуации, когда
структура результатов запроса не совпадает ни с одной из сущностей, описанных в приложении.

## Оптимизация метода courseForm при помощи проекций

Ранее мы обращали внимание, что в методе `courseForm` мы делаем неоптимальный запрос в БД. Мы
извлекаем тексты всех уроков несмотря на то, что для отображения списка уроков нам достаточно только
их заголовков. При использовании PostgreSQL нам пришлось добавить аннотацию `@Transactional`. Эту
проблему можно решить при помощи проекции:

```java

@Repository
public interface LessonRepository extends JpaRepository<Lesson, Long> {

    @Query("select new com.example.dto.LessonDto(l.id, l.title, l.course.id) " +
        "from Lesson l where l.course.id = :id")
    List<LessonDto> findAllForLessonIdWithoutText(long id);
}
```

# Расширение JPA-репозиториев

Автоматическая генерация имплементации JPA-репозиториев удобна тем, что избавляет от необходимости
писать простые запросы. Однако в некоторых ситуациях нам требуется более широкая функциональность.

Предположим, что мы разрабатываем сайт с рецензиями на кинофильмы, который дает пользователям
возможность ставить оценки. Допустим, нам необходимо написать запрос, который вернет список жанров по
ID. Причем для каждого жанра нам нужно получить топ N соответствующих фильмов по убыванию оценки.
Также стоит отметить, что для уменьшения нагрузки нам не требуется вся информация о фильме.
Достаточно только ID, названия и рейтинга.

Написать такой запрос с помощью ранее описанного подхода с аннотациями будет довольно проблематично.
Во-первых, сам запрос непростой. А во-вторых, здесь присутствует сложный маппинг. Именно для таких
целей Spring и предоставляет возможность добавления кастомных репозиториев.

> В рамках этого примера мы предполагаем, что у фильма может быть только один жанр.
> Конечно, в реальности это не так, и запрос будет сложнее. Однако базовый принцип реализации от
> этого не поменяется.

## DTO

Сначала необходимо описать конечные сущности, которые будут результатами выборки. В нашем случае их
две: жанр и фильм.

```java
// геттеры, конструкторы, equals и hashCode опущены для краткости

public class GenreDto {

    private Long id;
    private String name;
    private List<MovieDto> movies;
}

public class MovieDto {

    private Long id;
    private String name;
}
```

## Кастомный репозиторий

Теперь создадим кастомный репозиторий.

```java
public interface CustomGenreRepository {

    List<GenreDTO> findGenresWithMovies(Collection<Long> genreIds, int topMoviesCount);
}
```

`Collection<Long> genreIds` - это список ID-шников жанров, а `topMoviesCount` - максимальное
количество "топовых" фильмов для каждого жанра.

Далее напишем имплементацию.

```java

@Repository
@Transactional(readOnly = true)
public class CustomGenreRepositoryImpl implements CustomGenreRepository {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<GenreDto> findGenresWithMovies(Collection<Long> genreIds, int topMoviesCount) {
        List<Tuple> tuples = em.createNativeQuery(
                """select * from (
                 select g.id as genre_id, 
                 g.name as genre_name, "
                 m.id as movie_id, "
                 m.name as movie_name,
                 m.rating as movie_rating,
                 row_number() over (
                     partition by genre_id  
                     order by movie_rating desc
                 ) as rn
                 from genre g
                 where id in (:genreIds) 
                 left join movie m on m.genre_id = g.id) sub 
               where sub.rn <= :topMoviesCount""",
                Tuple.class
            ).setParameter("genreIds", genreIds)
                                 .setParameter("topMoviesCount", topMoviesCount)
                                 .getResultList();
        Map<Long, GenreDto> genres = new HashMap<>();
        for (Tuple tuple : tuples) {
            Long genreId = tuple.get("genre_id", Long.class);
            GenreDto genreDto = genres.computeIfAbsent(
                genreId,
                k -> new GenreDto(genreId, tuple.get("genre_name", String.class), new ArrayList<>())
            );
            Long movieId = tuple.get("movie_id", Long.class);
            if (movieId != null) {
                genreDto.getMovies().add(new MovieDto(movieId, tuple.get("movie_name", String.class),
                    tuple.get("movie_rating", Double.class)));
            }
        }
        return new ArrayList<>(genres.values());
    }
}
```

Давайте разберем этот класс детально.

Во-первых, аннотация `@Repository`. В случае с JpaRepository добавлять ее не обязательно, так как
Spring и так создаст реализацию и добавит ее в контекст. Здесь же этого автоматически не произойдет.

Во-вторых, `@Transactional(readOnly = true)`. В кастомных репозиториях это является хорошей
практикой. Если же где-то будет производиться операция `insert` или `update`, можно сделать
перегрузку, указав над соответствующим методом `@Transactional`.

`@PersistenceContext` обладает похожим поведением на `@Autowired`, но есть определенные отличия.
Дело в том, что один инстанс `EntityManager` должен быть привязан к одной транзакции.

Аннотация `@PersistenceContext` позволяет упростить этот процесс. Она внедряет прокси над
реальным `EntityManager`, который инициализируется только при первом обращении. Более того, прокси
также проверяет текущую активную транзакцию и при необходимости создает новый инстанс внутри себя.
Правда есть ограничение. Внедрение зависимости с помощью `@PersitenceContext` не работает через
конструктор.

Здесь мы
используем [оконную функцию `row_number`](https://www.postgresqltutorial.com/postgresql-row_number/),
чтобы пронумеровать жанры по убыванию их рейтинга. Затем с помощью `where` выбираем не
более `topMoviesCount` фильмов. Полученные строки преобразуются в `javax.persistence.Tuple`. Это
специальный интерфейс, который упрощает доступ к результату динамического запроса. Потом, наконец,
преобразуем результат в `GenreDto`.

Проходясь по циклу, мы проверяем, был ли уже данный фильм. Они могут повторяться из-за
блока `left join`. Если `movie_id` не `null`, то мы добавляем информацию о фильме в список.

> `movie_id` может равняться `null` в том случае, если у жанра нет ни одного фильма.
> Так как набор колонок фиксированный, единственный выходом для БД будет записать в отсутствующие
> колонки `null`.

В конце полученные значения преобразуем в список и возвращаем клиенту.

## Интеграция с Jpa-репозиторием

Как теперь использовать данный метод в приложении? Можно заинжектить бин `CustomGenreRepository` и
вызывать нужный метод. Однако это неудобно, так как, скорее всего, у нас будет еще
и `GenreRepository`, который наследуется от `JpaRepository`. Чтобы не было необходимости инжектить
два бина вместо одного, Spring позволяет делегировать вызовы наших кастомных методов через прокси,
который сгенерируется Spring.

```java

@Repository
public interface GenreRepository extends JpaRepository<Genre, Long>, CustomGenreRepository {

}
```

Теперь мы можем вызывать наши кастомные методы через `GenreRepository`. Таким способом можно
расширить JpaRepository несколькими пользовательскими слоями.
