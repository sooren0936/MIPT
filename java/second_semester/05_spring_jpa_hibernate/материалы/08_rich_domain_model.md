## Богатая доменная модель против анемичной

При использовании ORM есть два основных подхода к организации
бизнес-логики: [богатая модель](https://habr.com/ru/articles/87812/)
и [анемичная модель](https://martinfowler.com/bliki/AnemicDomainModel.html).
Чаще всего можно увидеть второй подход (именно он и рассмотрен в этом модуле).
Его суть в том, что entity выступают лишь хранилищем данных с набором getters/setters.
Вся бизнес-логика при этом расположена в сервисах.

Несмотря на то, что такой подход очень популярен, у него есть определенные проблемы:

1. Вы не можете проверить бизнес-логику с помощью unit-тестов. Приходится внедрять интеграционные.
   Потому что вся логика расположена в сервисах, а их методы очень плотно завязаны на репозиториях и
   транзакциях.
2. В entity могут
   быть [инварианты](https://ru.wikipedia.org/wiki/%D0%98%D0%BD%D0%B2%D0%B0%D1%80%D0%B8%D0%B0%D0%BD%D1%82).
   Это условия внутреннего состояния объекта, которые всегда должны соблюдаться. Например, у
   каждого `Course` всегда должен быть хотя бы один `Lesson`. Но поскольку entity предоставляет
   публичные setters для изменения своего состояния, в любой момент внешний код может вмешаться во
   внутреннее состояние и изменить его. Отслеживать такие вещи довольно сложно.
3. Бизнес-логика «размазывается» между разными слоями приложения. Что впоследствии может привести к
   «спаггети-коду».

Мы не будем агитировать использовать богатую модель над анемичной или наоборот. У богатой тоже есть
свои нюансы, связанные с performance.
Тем не менее, мы хотим дать вам несколько советов, как правильно использовать некоторые подходы
богатой модели, чтобы сделать код более поддерживаемым.

### Конструктор по умолчанию должен быть protected

Когда люди только начинают знакомиться с Hibernate, можно увидеть такой подход к работе с данными:

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

    public Course() {
        // no op
    }

    // getters, setters
}

@Service
public class CourseService {
    private final CourseRepository courseRepository;

    public Long createCourse(String author, String title) {
        Course course = new Course();
        course.setAuthor(author);
        course.setTitle(title);
        return courseRepository.save(course).getId();
    }
}
```

Проблема здесь в том, что мы должны явно вызвать `setAuthor` и `setTitle`. Иначе произойдет попытка
создания `Course` с неверными полями, что приведет к ошибке.

Хорошим вариантом является перенос части проверок на уровень компиляции. Если код не даст
возможность создать `Course` с неверными полями, риск ошибиться намного меньше.

Посмотрите на исправленный пример кода ниже:

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
        // no op
    }

    public Course(String author, String title) {
        this.author = author;
        this.title = title;
    }

    // getters, setters
}

@Service
public class CourseService {
    private final CourseRepository courseRepository;

    public Long createCourse(String author, String title) {
        Course course = new Course(author, title);
        return courseRepository.save(course).getId();
    }
}
```

Конструктор по умолчанию теперь `protected`: у `CourseService` не будет к нему доступа из другого
package. Другой же конструктор принимает все обязательные параметры.

> Также хорошей практикой является [fail-fast](https://habr.com/ru/articles/697084/) валидация.
> Можно проверять валидность переданных `author` и `title` прямо в конструкторе и при необходимости
> кидать исключение.

### Не добавляйте лишние setters

IDEA позволяет двумя кликами сгенерировать setters для всех полей класса. Также можно использовать
аннотацию `@Setter` из [Lombok](https://projectlombok.org/setup/maven) и добиться такого же
результата. Но здесь есть несколько проблем:

1. Некоторые поля вообще не предполагается мутировать из клиентского кода. Например, `id` или поля,
   которые заполняются лишь один раз при создании объекта.
2. Значения некоторых полей автоматически вычисляются на основании других.
3. Определенные поля можно менять только вместе.

Поэтому тут мы вам рекомендуем следующее:

1. Не добавляйте setters на все подряд. Действительно ли это конкретное поле можно менять?
2. Если поля меняются всегда вместе, объедините их в `@Embeddable`.

### Не используйте примитивы на все подряд, а внедряйте Value Object

Допустим, что вам нужно хранить количество денег в сущности. Какой тип здесь стоит
использовать? `int`? А может быть `long`?

> [Деньги никогда не хранят в числах с плавающей точкой](https://stackoverflow.com/questions/3730019/why-not-use-double-or-float-to-represent-currency).

Нюанс в том, что граница возможных значений для `int` и `long` намного больше, чем потенциальное
количество вариантов хранения денег. Вряд ли отрицательная сумма будет допустима (хотя все зависит
от конкретного случая).

В Domain Driven Design одним из главных строительных элементов
является [Value Object](https://habr.com/ru/articles/275599/). Это обертка вокруг одного или
нескольких значений (или других Value Objects), которая дополнительно вносит определенные правила
валидации.
В контексте Hibernate для Value Object можно использовать `@Embeddable`.

Посмотрите на пример класса `Money` ниже:

```java

@Embeddable
public class Money {
    private long value;

    @Enumerated(STRING)
    private Currency currency;

    protected Money() {
        // no op
    }

    public Money(long value, Currency currency) {
        this.value = validateValue(value); // проверяем, что value в диапазоне допустимых значений
        this.currency = currency;
    }

    // getters, equals, hashCode

    public enum Currency {
        USD, RUB
    }
}
```

Мы объединили два простых типа `long` и `currency` в один концепт – `Money`. Более того,
валидность `Money` проверяется при создании объекта. А значит, если entities будут использовать
именно Value Object `Money`, это исключит возможность сохранения невалидного состояния.

Value Object - удобный паттерн для моделирования ваших сущностей. Пользуйтесь им.