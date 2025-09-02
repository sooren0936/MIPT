# Проблемы ORM

Несмотря на то что концепция ORM очень удобна для разработки, она не лишена изъянов. Дело в том, что
Hibernate и подобные фреймворки предлагают нам абстракцию в виде Java-классов. Когда мы что-то
добавляем, изменяем или удаляем, ORM генерирует SQL, который и выполняет действие над СУБД. Вот
только есть один нюанс. В некоторых ситуациях выполняется совсем не тот набор запросов, который мы
ожидали. Иногда они могут быть крайне неэффективными.

## Проблема N + 1

N + 1 — самая распространенная проблема с производительностью в Hibernate. Если вы вобьете в
поиске `hibernate n+1 problem`, вы найдете сотни статей, которые описывают данный феномен.

В чем же дело? Давайте предположим, что мы занимаемся разработкой портала по публикации статей. В
системе будет две центральные сущности: `Post` и `Comment`.

```java

@Entity
public class Post {

  @Id
  @GeneratedValue(strategy = IDENTITY)
  private Long id;

  private String text;

  @OneToMany(fetch = LAZY, mappedBy = "post")
  private List<Comment> comments;
}

@Entity
public class Comment {

  @Id
  @GeneratedValue(strategy = IDENTITY)
  private Long id;

  private String text;

  @ManyToOne(fetch = LAZY)
  private Post post;
}
```

Обратите внимание, что и связь `Post.getComments()`, и `Comment.getPost()` объявлены как `lazy`. Это
необходимо, чтобы запрашивать дополнительные данные только по необходимости.

### Many-to-One

Допустим, мы проходимся в цикле по списку комментариев, и для каждого из них нам нужно получить
соответствующий пост.

```java
List<Comment> comments=commentRepository.findAll();
    for(Comment comment:comments){
    Post post=comment.getPost();
    // do something with post
    }
```

Сколько SQL-запросов будет выполнено, если всего 5 комментариев? Можно предположить, что один: тот,
который выбрал комментарии, так ведь? Что же, не совсем.

```sql
select comment0_.id as id1_2_, comment0_.post_id as post_id3_2_, comment0_.text as text2_2_
from comment comment0_
select post0_.id as id1_3_0_, post0_.text as text2_3_0_
from post post0_
where post0_.id = ?
select post0_.id as id1_3_0_, post0_.text as text2_3_0_
from post post0_
where post0_.id = ?
select post0_.id as id1_3_0_, post0_.text as text2_3_0_
from post post0_
where post0_.id = ?
select post0_.id as id1_3_0_, post0_.text as text2_3_0_
from post post0_
where post0_.id = ?
select post0_.id as id1_3_0_, post0_.text as text2_3_0_
from post post0_
where post0_.id = ?
```

Комментарии были и правда выбраны одним запросом. Однако по причине того, что связь между `Comment`
и `Post` является `lazy`, на момент вызова `getPost()` информация о посте еще не была запрошена.
Поэтому в данном случае поведение Hibernate вполне логично: на каждый дополнительный вызов делаем
новый запрос.

Как решить эту проблему? Можно попробовать заменить параметр на `@ManyToOne(fetch = EAGER)`. Правда,
в данном случае это не поможет, так как Hibernate просто сделает по одному дополнительному `select`
на каждый комментарий в момент их получения. То есть количество запросов не изменится, однако они
будут выполнены сразу, а не к моменту вызова `getPost()`.

Лучшим решением будет объявить кастомный метод в репозитории.

```java
public interface CommentRepository extends JpaRepository<Comment, Long> {

  @Query("from Comment c join fetch c.post")
  List<Comment> findAllWithPosts();
}
```

Теперь в логах будет всего один запрос.

```sql
select comment0_.id      as id1_2_0_,
       post1_.id         as id1_3_1_,
       comment0_.post_id as post_id3_2_0_,
       comment0_.text    as text2_2_0_,
       post1_.text       as text2_3_1_
from comment comment0_
         inner join post post1_ on comment0_.post_id = post1_.id
```

Все необходимые данные были выбраны одним запросом с помощью `INNER JOIN`. Благодаря такому подходу
мы можем оставить атрибут `Comment.getPost()` как `lazy` и не запрашивать лишние данные без
необходимости.

### One-to-Many

Обратная ситуация: мы хотим пройтись по списку постов и для каждого получить комментарии.

```java
List<Post> posts=postRepository.findAll();
    for(Post post:posts){
    List<Comment> comments=post.getComments();
    // do something with comments
    }
```

Как и в предыдущем случае, количество запросов также не будет равняться единице.

```sql
select post0_.id as id1_3_, post0_.text as text2_3_
from post post0_;

select comments0_.post_id as post_id3_2_0_,
       comments0_.id      as id1_2_0_,
       comments0_.id      as id1_2_1_,
       comments0_.post_id as post_id3_2_1_,
       comments0_.text    as text2_2_1_
from comment comments0_
where comments0_.post_id = ?;

select comments0_.post_id as post_id3_2_0_,
       comments0_.id      as id1_2_0_,
       comments0_.id      as id1_2_1_,
       comments0_.post_id as post_id3_2_1_,
       comments0_.text    as text2_2_1_
from comment comments0_
where comments0_.post_id = ?;

select comments0_.post_id as post_id3_2_0_,
       comments0_.id      as id1_2_0_,
       comments0_.id      as id1_2_1_,
       comments0_.post_id as post_id3_2_1_,
       comments0_.text    as text2_2_1_
from comment comments0_
where comments0_.post_id = ?;

select comments0_.post_id as post_id3_2_0_,
       comments0_.id      as id1_2_0_,
       comments0_.id      as id1_2_1_,
       comments0_.post_id as post_id3_2_1_,
       comments0_.text    as text2_2_1_
from comment comments0_
where comments0_.post_id = ?;

select comments0_.post_id as post_id3_2_0_,
       comments0_.id      as id1_2_0_,
       comments0_.id      as id1_2_1_,
       comments0_.post_id as post_id3_2_1_,
       comments0_.text    as text2_2_1_
from comment comments0_
where comments0_.post_id = ?;
```

Теперь при вызове `getComments()` для каждого поста мы запрашиваем список его комментариев.

Решений проблемы несколько.

#### Subselect

Можно добавить аннотацию `@Fetch(SUBSELECT)`.

```java
@OneToMany(fetch = LAZY, mappedBy = "post")
@Fetch(SUBSELECT)
private List<Comment> comments;
```

Здесь инициализация комментариев останется ленивой, но логика построения запроса изменится.
Hibernate запомнит SQL, который был выполнен при получении постов, и в момент первого
вызова `getComments()` использует его как подзапрос, чтобы получить все комментарии для всех нужных
постов. Тогда данные в предыдущем примере будут выбраны с помощью всего двух запросов.

```sql
select post0_.id as id1_3_, post0_.text as text2_3
from post post0_;

select comments0_.post_id as post_id3_2_1_,
       comments0_.id      as id1_2_1_,
       comments0_.id      as id1_2_0_,
       comments0_.post_id as post_id3_2_0_,
       comments0_.text    as text2_2_0_
from comment comments0_
where comments0_.post_id in (select post0_.id from post post0_)
```

Сначала запрашиваем информацию о постах, а потом — о комментариях.

У этого подхода есть один недостаток. Если запрос на выборку постов был комплексный (с
фильтрами, `JOIN`, `UNION` и так далее), Hibernate будет вынужден повторить его в подзапросе.
Фактически это значит, что вы выполняете сложный запрос два раза. В некоторых ситуациях это может
сказаться на производительности.

#### Join

Есть вариант поступить таким же образом, как и с выборкой комментариев. Объявим кастомный запрос
с `JOIN`.

```java
public interface PostRepository extends JpaRepository<Post, Long> {

  @Query("from Post p left join fetch p.comments")
  List<Post> findAllWithComments();
}
```

> `LEFT JOIN` нужен, чтобы выбрать еще и те посты, у которых нет комментариев.

Теперь все данные будут получены одним запросом.

```sql
select post0_.id          as id1_3_0_,
       comments1_.id      as id1_2_1_,
       post0_.text        as text2_3_0_,
       comments1_.post_id as post_id3_2_1_,
       comments1_.text    as text2_2_1_,
       comments1_.post_id as post_id3_2_0__,
       comments1_.id      as id1_2_0__
from post post0_
         left outer join comment comments1_ on post0_.id = comments1_.post_id
```

Плюсы здесь аналогичны примеру с One-to-Many. Мы имеем возможность выбирать посты вместе с
комментариями только тогда, когда нам это нужно.

### Batch Size

Батчинг является компромиссом между `SUBSELECT` и `JOIN`. Вместо того чтобы выбирать дочерние записи
на каждый вызов `getComments()`, мы запрашиваем комментарии для нескольких постов сразу.

Предположим, что размер батча равен 3.

```java
@OneToMany(fetch = LAZY, mappedBy = "post")
@BatchSize(size = 3)
private List<Comment> comments;
```

Тогда, если у нас 5 постов, нам понадобится 2 дополнительных запроса, чтобы получить все комментарии
для них.

```sql
select post0_.id as id1_3_, post0_.text as text2_3_
from post post0_;

select comments0_.post_id as post_id3_2_1_,
       comments0_.id      as id1_2_1_,
       comments0_.id      as id1_2_0_,
       comments0_.post_id as post_id3_2_0_,
       comments0_.text    as text2_2_0_
from comment comments0_
where comments0_.post_id in (?, ?, ?);

select comments0_.post_id as post_id3_2_1_,
       comments0_.id      as id1_2_1_,
       comments0_.id      as id1_2_0_,
       comments0_.post_id as post_id3_2_0_,
       comments0_.text    as text2_2_0_
from comment comments0_
where comments0_.post_id in (?, ?);
```

В первом SQL мы запрашиваем комментарии для 3-х постов, а во втором — для оставшихся 2-х.
Оптимальный размер батча определяется в зависимости от особенностей продукта.

> Важно заметить, что `@BatchSize` определяет максимальное количество **родительских** сущностей,
> для которых будут запросы дочерние.
> При этом аннотация никак не влияет на количество запрашиваемых комментариев.

### Batch Size + Many-to-One

Батчинг также успешно работает и в связках "много-к-одному".

Давайте вернемся к примеру получения поста для каждого комментария.

```java
List<Comment> comments=commentRepository.findAll();
    for(Comment comment:comments){
    Post post=comment.getPost();
    // do something with post
    }
```

Что если мы хотим запрашивать посты батчами по 3 штуки? Во-первых, нужно поставить аннотацию над
сущностью `Post`.

```java

@Entity
@BatchSize(size = 3)
public class Post {
  ...
}
```

> `@BatchSize` над полем работает только для связей `@OneToMany` и `@ManyToMany`.
> Отсюда следует, что нельзя настроить разные батчи для разных связей `@ManyToOne` к одной и той же сущности.

Пусть у нас всего 8 комментариев. Тогда посты будут запрошены тремя дополнительными запросами.

```sql
select comment0_.id as id1_2_, comment0_.post_id as post_id3_2_, comment0_.text as text2_2_
from comment comment0_;

select post0_.id as id1_3_0_, post0_.text as text2_3_0_
from post post0_
where post0_.id in (?, ?, ?);

select post0_.id as id1_3_0_, post0_.text as text2_3_0_
from post post0_
where post0_.id in (?, ?, ?);

select post0_.id as id1_3_0_, post0_.text as text2_3_0_
from post post0_
where post0_.id in (?, ?);
```

Как видите, поведение аналогично случаю с `@OneToMany`.

# Lombok Builder при использовании Hibernate

В современной разработке принцип иммутабельности является довольно популярным. Действительно,
объявляя классы неизменяемыми, мы получаем ряд преимуществ:

1. Состояние объекта всегда статично. Следовательно, предсказуемо.
2. Легко гарантировать соблюдение всех инвариантов: достаточно провести валидацию входных параметров
   единожды в конструкторе.
3. С неизменяемыми объектами легче работать в многопоточной среде.

Следуя этому принципу, некоторые разработчики применяют его и к сущностям Hibernate. Особенно к
этому подталкивает [Lombok](https://projectlombok.org/). Давайте рассмотрим проблемы данного подхода
на примере сущности `Person`.

```java

@Entity
@Table(name = "person")
@Getter
@Builder(toBuilder = true)
@AllArgsConstructor
@NoArgsConstructor
public class Person {

  @Id
  @GeneratedValue(strategy = IDENTITY)
  private Long id;

  @Column(name = "first_name")
  private String firstName;

  @Column(name = "last_name")
  private String lastName;
}
```

Аннотация `@Builder` сгенерирует код для построения объекта по шаблону
паттерна [«Строитель»](https://metanit.com/sharp/patterns/2.5.php). А свойство `toBuilder = true`
позволяет создавать новый инстанс билдера на основании существующего объекта `Person`.

> Обратите внимание на наличие аннотации `@NoArgsConstructor`.
> Контракт JPA требует присутствия конструктора без аргументов.
> Поэтому его нужно указывать явно.

Таким образом, инстанс `Person` остается неизменным, в то время как его обновление осуществляется
путем создания временной мутабельной конструкции — строителя.

```java
public class PersonService {

  public void createPerson() {
    Person simon = Person.builder()
        .firstName("Simon")
        .lastName("Rails")
        .build();
    Person jack = simon.toBuilder()
        .firstName("Jack")
        .build();
  }
}
```

На первый взгляд, все выглядит неплохо. Но дьявол кроется в деталях.

## Проблемы с Domain Events

Предположим, что мы хотим отслеживать события изменения имени или фамилия человека. Spring
предоставляет механизм event listeners «из коробки»:

```java

@Service
public class Listener {

  @EventListener
  public void onFirstNameChanged(FirstNameChangedEvent e) {
    // ...
  }

  @EventListener
  public void onLastNameChanged(LastNameChangedEvent e) {
    // ...
  }
}
```

Теперь нужно как-то зарегистрировать событие. Самый простой вариант — расположить логику в сервисном
слое.

```java

@Service
@RequiredArgsConstructor
public class PersonService {

  private final PersonRepository personRepository;
  private final ApplicationEventListener applicationEventListener;

  @Transactional
  public void updateFirstName(Long id, String newFirstName) {
    Person person = personRepository.findById(id).orElseThrow();
    Person newPerson = person.toBuilder().firstName(newFirstName).build();
    personRepository.save(newPerson);

    applicationEventListener.publishEvent(new FirstNameChangedEvent(id));
  }
}
```

Но что если имя человека может меняться в разных местах программы в зависимости от бизнес-операции?
В таком случае, логичнее инкапсулировать эту логику непосредственно внутри класса `Person`. К
счастью, Spring Data в комбинации с Hibernate дает такую возможность.

```java

@Entity
@Table(name = "person")
@NoArgsConstructor
@Getter
public class Person extends AbstractAggregateRoot {

  @Id
  @GeneratedValue(strategy = IDENTITY)
  private Long id;

  @Column(name = "first_name")
  private String firstName;

  @Column(name = "last_name")
  private String lastName;

  public void updateFirstName(String newFirstName) {
    this.firstName = newFirstName;
    registerEvent(new FirstNameChangedEvent(this.id));
  }
}
```

Класс `AbstractAggregateRoot` содержит метод `registerEvent`, который в рамках данного инстанса
собирает события в коллекцию. Они будут опубликованы после вызова метода `save` или `saveAndFlush` в
соответствующем репозитории.

Теперь логика редактирования имени человека будет выглядеть так:

```java

@Service
@RequiredArgsConstructor
public class PersonService {

  private final PersonRepository personRepository;

  @Transactional
  public void updateFirstName(Long id, String newFirstName) {
    Person person = personRepository.findById(id).orElseThrow();
    person.updateFirstName(newFirstName);
    personRepository.save(newPerson);
  }
}
```

Здесь мы отказываемся от Lombok Builder и вместо этого объявляем кастомный метод `updateFirstName`,
который инкапсулирует как редактирование имени, так и регистрацию доменного события внутри
класса `Person`. Теперь мы можем добавлять новые события и менять существующие, не затрагивая код,
реализованный в сервисном слое.

Хотя Lombok Builder — это полезный инструмент, сокращающий boilerplate, но также он подталкивает к
неправильной архитектуре в контексте применения Spring Data JPA.

> Spring Domain Events — большая тема, которая выходит за рамки этого курса.
> Если вам интересно, как можно применять доменные события в Spring Data JPA,
> рекомендуем ознакомиться с [этой статьей](https://dev.to/kirekov/spring-data-power-of-domain-events-2okm).
