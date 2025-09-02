# Отношения между сущностями

В этом разделе мы поговорим о возможных отношениях между сущностями в реляционных базах данных и
Hibernate в частности и добавим уроки в каждый из курсов.

## Виды отношений в реляционных базах данных

Любая реальная бизнес-модель предполагает наличие большого числа сущностей и многочисленных связей
между ними. Можно выделить три основных вида таких связей:

* one to one — один к одному
* one to many – один ко многим
* many to one - многие к одному
* many to many - многие ко многим

Связь один к одному используется довольно редко, а вот примеров связей one to one или many to one в
нашем проекте будет достаточно.

> Как мы увидим далее, many to one является зеркальным отражением для one to many.

Начнем с того, что каждый курс должен состоять из одного
или нескольких уроков – это характерный пример связи один ко многим (один курс — один или больше
уроков). При этом каждый урок относится лишь к одному курсу.

## Отношение многие к одному и один к многим. Добавляем уроки

Начнем с уроков. Класс `Lesson` будет выглядеть так:

```java

@Entity
@Table(name = "lessons")
public class Lesson {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    private String title;

    @NotNull
    private String text;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id")
    private Course course;

    protected Lesson() {
    }

    public Lesson(String title, String text, Course course) {
        this.title = title;
        this.text = text;
        this.course = course;
    }

    // getters, setters
}
```

Поле `course` — это ссылка на экземпляр сущности `Course`, в который входит данный урок. Об этом
Hibernate сообщает аннотация `@ManyToOne`. Атрибут `fetch = FetchType.LAZY` указывает на то, что
информация о `course` будет получена отдельным запросом в БД только в том случае, если мы если мы
вызовем `lesson.getCourse()`.
Таким образом, мы можем не выбирать лишние данные без необходимости.

> Важный момент: lazy-связь может быть запрошена только в рамках той же транзакции (
> аннотация `@Transactional`).
> Если транзакция уже закрыта, а мы попытаемся вытащить данные lazy-связи, то
> получим [LazyInitializationException](https://thorben-janssen.com/lazyinitializationexception/).

Если вы попробуете запустить сейчас приложение, то получите ошибку. Действительно, у нас нет
таблицы, куда можно было бы замаппить сущность `Lesson`. Исправим это. Добавьте такую миграцию:

```sql
CREATE TABLE lessons
(
    id        BIGSERIAL PRIMARY KEY,
    title     TEXT NOT NULL,
    text      TEXT NOT NULL,
    course_id BIGINT REFERENCES courses (id)
);
```

Как видите, связь `ManyToOne` равнозначна тому, что вы замаппили ID. Но Hibernate при этом позволяет
ссылаться сразу на сущность, а не на ее ID.

Чтобы стало понятнее, посмотрите на пример ниже без связи `ManyToOne`. Он тоже будет корректно
работать.

```java

@Entity
@Table(name = "lessons")
public class Lesson {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    private String title;

    @NotNull
    private String text;

    @Column(name = "course_id")
    private Long courseId;

    protected Lesson() {
    }

    public Lesson(String title, String text, Long courseId) {
        this.title = title;
        this.text = text;
        this.courseId = courseId;
    }

    // getters, setters
}
```

Вообще говоря, мы уже создали корректную связь между сущностями `Lesson` и `Course`, но это так
называемая односторонняя (unidirectional) связь, ведь мы указали информацию о ней только в одной из
связанных сущностей.

Hibernate позволяет создавать двусторонние связи. То есть если `Lesson` ссылается на `Course`,
то `Course` содержит много `Lesson`.

Исправим сущность `Course`. Посмотрите на пример ниже:

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

    @OneToMany(mappedBy = "course", orphanRemoval = true)
    private List<Lesson> lessons = new ArrayList<>();

    protected Course() {
    }

    public Course(String author, String title) {
        this.author = author;
        this.title = title;
    }

    // getters
}
```

Важным является атрибут `mappedBy`. Именно он делает связь двусторонней. В качестве значения этого
атрибута в сущности `Lesson`
мы указываем имя поля `course`, которое ссылается на сущность `Course`.

> Все `OneToMany` связи являются lazy по умолчанию.

Особенность двусторонних отношений заключается в том, что стороны в них неравноправны.
Выделяют сторону владельца отношений (relationship owner) и противоположную сторону (inverse side).
Атрибут `mappedBy` всегда указывается на стороне-владельце. В нашем случае владельцем отношения
является сущность `Course`.

### Каскадные операции для OneToMany

В связях `OneToMany` очень распространены каскадные операции. Суть в том, что если мы меняем
содержимое коллекции `lessons`, это также отражается и в таблице `lessons` в БД.

Рассмотрим `orhanRemoval`. Если этот параметр выставлен в `true`, то при удалении элемента из
коллекции он также будет удален из таблицы. Посмотрите на пример кода ниже:

```java

@Entity
@Table(name = "courses")
public class Course {

    // fields, methods...

    public void deleteLesson(Long lessonId) {
        lessons.removeIf(lesson -> Objects.equals(lesson.getId(), lessonId));
    }
}

public class CourseService {
    private final CourseRepository courseRepository;

    @Transactional
    public void deleteLesson(Long lessonId) {
        Course course = courseRepository.findByLessonId(lessonId);
        course.deleteLesson(lessonId);
        courseRepository.save(course);
    }
}
```

В данном случае по окончании транзакции будет сгенерирован SQL:

```sql
DELETE
FROM lessons
WHERE id = ?
```

Помимо каскадного удаления еще есть создание новых сущностей. Для этого в `@OneToMany` связь
добавьте параметр `cascade` со значением `PERSIST`. Тогда мы сможем добавлять уроки следующим
образом:

```java

@Entity
@Table(name = "courses")
public class Course {

    // fields, methods...

    @OneToMany(mappedBy = "course", orphanRemoval = true, cascade = {PERSIST})
    private List<Lesson> lessons = new ArrayList<>();


    public void createLesson(String text, String title) {
        lessons.add(new Lesson(text, title, this));
    }
}

public class CourseService {
    private final CourseRepository courseRepository;

    @Transactional
    public void createLesson(Long courseId, String text, String title) {
        Course course = courseRepository.findById(courseId);
        course.createLesson(text, title);
        courseRepository.save(course);
    }
}
```

Есть еще cascade, которая заслуживает внимания. А именно `CascadeType.REMOVE`. В случае
удаления `Course` Hibernate сначала удалит все `OneToMany` связи, а потом сгенерирует `DELETE`
для `Course`.
Таким образом можно избежать проблемы нарушения constraint, когда мы удаляем курс, а на него еще
ссылаются уроки.

Но мы не рекомендуем применять этот
cascade. [Он показывает плохую производительность](https://thorben-janssen.com/avoid-cascadetype-delete-many-assocations/).
Если вы хотите каскадом удалить курс и уроки, то лучше
объявить [foreign key в БД каскадным на удаление](https://www.commandprompt.com/education/postgresql-delete-cascade-with-examples/).
Тогда если в БД попадает `DELETE FROM course WHERE id = ?`, то все уроки, которые относятся к этому
курсу, также будут удалены.

## Отношение многие ко многим. Добавляем пользователей

Связь `ManyToMany` отличается от того, что мы рассмотрели ранее. Она выполняется через промежуточную
таблицу.

Допустим, у нас есть сущность `User`, который может быть записан на один или несколько `Course`. При
этом в одном `Course` может участвовать несколько `User`. Классический пример `ManyToMany` связи.

Сначала добавим миграцию, чтобы хранить эту информацию в БД:

```sql
CREATE TABLE users
(
    id       BIGSERIAL PRIMARY KEY,
    username TEXT NOT NULL
);

CREATE TABLE course_user
(
    course_id BIGINT REFERENCES courses (id) NOT NULL,
    user_id   BIGINT REFERENCES users (id)   NOT NULL,
    PRIMARY KEY (course_id, lesson_id)
);
```

Теперь давайте добавим класс-сущность `User` для пользователя и свяжем её с курсами через отношение
многие ко многим. Важный момент здесь заключается в том, что в качестве контейнера для связанных
сущностей мы будем использовать не `List`, как в отношении один ко многим, а `Set`. Множество вместо
списка здесь мы используем, так как нельзя привязать один и тот же курс к
пользователю дважды, а пользователь не может повторно записаться на курс, на который он уже записан.
То есть набор курсов у пользователя и набор пользователей у курса должны состоять из уникальных
значений.

```java
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;

    @ManyToMany(fetch = LAZY, cascade = PERSIST)
    @JoinTable(
        name = "course_user",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();

    protected User() {
    }

    public User(String username) {
        this.username = username;
    }
}
```

Здесь мы указываем тип отношения при помощи аннотации `@ManyToMany` над полем `courses`.
Дополнительная аннотация `JoinTable` указывает на промежуточную таблицу, на которую связь будет
маппится.
При этом наличие `JoinTable` означает, что `User` - это owning side. То есть привязки `User-Course`
мы должны менять именно со стороны `User`, а не `Course`. Проще говоря, правильным будем такой
вариант: `user.assignCourse(course)`.

> Со стороны `JoinTable` связи являются автоматически `orphanRemoval = true`. То есть если
> удалить `Course` из коллекции `courses`,
> то будет удалена именно строка из `course_user`, но не `courses`.
>
> Иначе говоря, при удалении элемента из `courses` из БД удалится именно связь, но не сам курс.

Добавить же маппинг со стороны сущности `Course` будет намного проще. Посмотрите на пример кода
ниже:

```java

@Entity
@Table(name = "courses")
public class Course {

    // fields, methods...

    @ManyToMany(mappedBy = "courses")
    private Set<User> users = new HashSet<>();
}
```

В `mappedBy` указываем поле, в котором хранится обратная связь.

## Многие ко многим через OneToMany

В некоторых ситуациях связь через промежуточную таблицу нельзя отобразить через `ManyToMany`.
Предположим, что при записи пользователя на курс еще указывается дата, до которой доступ к курсу
будет активен.
Тогда структура БД изменится:

```sql
CREATE TABLE users
(
    id       BIGSERIAL PRIMARY KEY,
    username TEXT NOT NULL
);

CREATE TABLE course_user
(
    valid_until TIMESTAMP WITH TIME ZONE       NOT NULL,
    course_id   BIGINT REFERENCES courses (id) NOT NULL,
    user_id     BIGINT REFERENCES users (id)   NOT NULL,
    PRIMARY KEY (course_id, lesson_id)
);
```

Как видите, у нас появилось еще одно обязательное поле – `valid_until`. Теперь связка
через `ManyToOne` не сработает, потому что Hibernate не знает, как именно это поле заполнять.

Чтобы это исправить, нужно добавить отдельную сущность `CourseUser`, которая будет маппиться на
таблицу `course_user`, а затем использовать ее в связях в `User` и `Course`.

Посмотрите на пример ниже:

```java

@Entity
@Table(name = "course_user")
public class CourseUser {
    @EmbeddedId
    private Id id;
    @Column(name = "valid_until")
    private OffsetDateTime validUntil;

    protected CourseUser() {
    }

    public CourseUser(Long courseId, Long userId, OffsetDateTime validUntil) {
        this.id = new Id(courseId, userId);
        this.validUntil = validUntil;
    }

    @Embeddable
    static class Id {
        @Column(name = "course_id")
        private Long courseId;
        @Column(name = "user_id")
        private Long userId;

        protected Id() {
        }

        public Id(Long courseId, Long userId) {
            this.courseId = courseId;
            this.userId = userId;
        }
    }
}

@Entity
@Table(name = "courses")
public class Course {
    // fields, methods...

    @OneToMany
    @JoinColumn(name = "course_id", updatable = false, cascade = {PERSIST})
    private List<CourseUser> courseUsers = new ArrayList<>();

    public void assignUser(User user, OffsetDateTime validUntil) {
        this.courseUsers.add(new CourseUser(this.id, user.getId(), validUntil));
    }
}

@Entity
@Table(name = "users")
public class User {
    // fields, methods...

    @OneToMany
    @JoinColumn(name = "user_id", updatable = false, cascade = {PERSIST})
    private List<CourseUser> courseUsers = new ArrayList<>();

    public void assignCourse(Course course, OffsetDateTime validUntil) {
        this.courseUsers.add(new CourseUser(course.getId(), this.id, validUntil));
    }
}
```

> Подробнее Embeddable мы разберем далее в модуле. Здесь он используется по той причине, что в
> таблице `course_user` несколько колонок являются primary key.

Поскольку связь односторонняя (есть `@OneToMany`, но с обратной стороны нет `ManyToOne`), нам нужно
через `@JoinColumn` явно указать колонку, по которой список будет цепляться.

## Equals и hashCode

Когда речь заходит об `equals/hashCode`, начинающие пользователи Hibernate совершают типичную
ошибку. Они добавляют в проверки поля, которые могут меняться. Поскольку сущность в Hibernate
мутабельная по определению, изменение в поле приведет к нарушению контракта equals/hashCode, что
также приведет к неожиданным багам в runtime.

Чтобы такого не возникало, equals/hashCode нужно строить по ID. Но здесь тоже есть нюанс. По
умолчанию ID null, а потом Hibernate его заполняет. То есть опять же значение нестабильное.

В качестве альтернативы можно использовать другую колонку, которая гарантированно уникальна и
заполняется при создании объекта. Например, ИНН или СНИЛС.
Но такие колонки есть не всегда, тогда для ID можно использовать такой подход:

```java
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    /* other fields */

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof User user)) {
            return false;
        }
        return id != null && id.equals(user.id);
    }

    @Override
    public int hashCode() {
        return User.class.hashCode();
    }
}
```

Логика здесь такая:

1. Функция `hashCode` всегда возвращает одинаковое значение. В данном случае, хэш от класса.
2. Если объекты равны по ссылкам в `equals`, возвращаем `true`.
3. Если переданный объект не является экземпляром или наследником `User`, возвращаем `false`.
4. Если `id` в текущем объекте равен `null`, возвращаем `false`.
5. Иначе сравниваем `id` текущего и переданного объектов на равенство.

> Подробнее о логике данного алгоритма
>
можете [почитать здесь](https://vladmihalcea.com/how-to-implement-equals-and-hashcode-using-the-jpa-entity-identifier/).

Смысл четвертого пункта в том, что если у текущего объекта `id` еще не заполнен, мы не можем
сказать, равен ли он переданной сущности или нет.
Так что в этом случае мы всегда по умолчанию считаем объекты неравными.

Если же `id` заполнены, то их равенство означает, что две entity равны.

> На самом деле, вам необязательно писать это все самостоятельно.
> Для IDEA есть замечательный плагин [JPA Buddy](https://jpa-buddy.com/).
> Он не только может сгенерировать корректный `equals/hashCode` для вашей сущности, но также
> подсказывает вам, как правильно настроить маппинг и где у вас ошибки.

## Embeddable

Иногда удобно объединить несколько полей сущности в один Java-объект. С этим поможет
аннотация `Embeddable`.
Допустим, что таблица `users` у нас представлена в таком виде:

```sql
CREATE TABLE users
(
    id          BIGSERIAL PRIMARY KEY,
    username    TEXT NOT NULL,
    first_name  TEXT NOT NULL,
    last_name   TEXT NOT NULL,
    middle_name TEXT NOT NULL
);
```

```java
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @Column(name = "first_name")
    private String firstName;

    @NotNull
    @Column(name = "last_name")
    private String lastName;

    @NotNull
    @Column(name = "middle_name")
    private String middleName;

    public void updateUserInfo(String firstName, String lastName, String middleName) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.middleName = middleName;
    }

    // constructors, equals, hashCode, getters, setters
}
```

Как мы видим, информация о пользователе всегда меняется совместно. Поэтому было бы логичнее
объединить `firstName`, `lastName` и `middleName` в один объект. Посмотрите на пример кода ниже:

```java

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Embedded
    @NotNull
    private UserInfo userInfo;

    public void updateUserInfo(UserInfo userInfo) {
        this.firstName = userInfo;
    }

    // constructors, equals, hashCode, getters, setters
}

@Embeddable
public class UserInfo {
    @NotNull
    @Column(name = "first_name")
    private String firstName;

    @NotNull
    @Column(name = "last_name")
    private String lastName;

    @NotNull
    @Column(name = "middle_name")
    private String middleName;

    protected UserInfo() {
        // ...
    }

    public UserInfo(String firstName, String lastName, String middleName) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.middleName = middleName;
    }

    @Override
    public int hashCode() {
        return Objects.hash(firstName, lastName, middleName);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof UserInfo userInfo)) {
            return false;
        }
        return Objects.equals(firstName, o.firstName)
            && Objects.equals(lastName, o.lastName)
            && Objects.equals(middleName, o.middleName);
    }
}
```

> По аналогии с `@Entity`, для `@Embeddable` нужен `protected` конструктор без аргументов.

Хотя в `User` и присутствует объект `UserInfo` в качестве поля, Hibernate будет считать совокупность
трех полей в БД благодаря сочетанию аннотаций `@Embeddable` и `@Embedded`.

У Embeddable есть особенности:

1. В отличие от `@Entity`, их стоит проектировать как иммутабельные классы.
2. Поскольку в `@Embeddable` как правило нет id, в `equals` и `hashCode` нужно добавлять все поля.

Еще одна killer feature `@Embeddable` в том, что их можно переиспользовать. Допустим, у нас есть
сущность `Client`, у которой также присутствуют поля `firstName`, `lastName` и `middleName`. Мы
можем внедрить туда `@Embeddable`, чтобы не дублировать код:

```java

@Entity
@Table(name = "client")
public class Client {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Embedded
    @NotNull
    private UserInfo clientInfo;

    // constructors, equals, hashCode, getters, setters
}
```

А теперь предположим, что колонки в `Client` называются немного иначе:

```sql
CREATE TABLE users
(
    id         BIGSERIAL PRIMARY KEY,
    username   TEXT NOT NULL,
    name       TEXT NOT NULL,
    surname    TEXT NOT NULL,
    patronymic TEXT NOT NULL
);
```

Поскольку мы уже «зашили» названия колонок внутри `@Embeddable`, такой маппинг приведет к ошибке в
сущности `Client`.
Но Hibernate и это предусмотрел. В том месте, где вы используете `@Embeddable`, можно перегружать
маппинг колонок:

```java

@Entity
@Table(name = "client")
public class Client {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "firstName", column = @Column(name = "name")),
        @AttributeOverride(name = "lastName", column = @Column(name = "surname")),
        @AttributeOverride(name = "middleName", column = @Column(name = "patronymic"))
    })
    @NotNull
    private UserInfo clientInfo;

    // constructors, equals, hashCode, getters, setters
}
```

Важно, что в `name` мы указываем название **свойства** (то есть название поля в
классе `@Embeddable`), а в `@Column` – название **колонки** в БД. 