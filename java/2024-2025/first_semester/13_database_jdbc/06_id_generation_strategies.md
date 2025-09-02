# Стратегии генерации ID

## Сложности с ID при внедрении реализации для БД

Давайте посмотрим еще раз на код из предыдущего урока, где мы создавали организации:

```java
public class Organization {

  private final long id;
  private final String name;

  /* constructor, equals, hashCode */
}

public class OrganizationRepository {

  private final Jdbi jdbi;

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

  /* другие методы + конструктор... */
}
```

Вы обратили внимание, что при создании организации мы не передаем в качестве параметра
объект `Organization`, а вместо этого - лишь поля отдельными параметрами? Мы сделали это для
упрощения, чтобы сосредоточиться на концепции интеграционных тестов.

Тем не менее, давайте еще раз взглянем на интерфейс `BookRepository` из модуля, в котором мы знакомились
со Spark Framework:

```java
public interface BookRepository {

  long generateId();

  List<Book> findAll();

  /**
   * @throws BookNotFoundException
   */
  Book findById(long bookId);

  /**
   * @throws BookIdDuplicatedException
   */
  void create(Book book);

  /**
   * @throws BookNotFoundException
   */
  void update(Book book);

  /**
   * @throws BookNotFoundException
   */
  void delete(long bookId);
}
```

Посмотрите на метод `create`. Он возвращает `void`, но принимает на вход объект `Book`, в котором
уже подставлен нужный ID-шник. Правильный ли это подход при реализации доступа к БД? Есть ли другие
варианты? Ответ на оба вопроса - да. Но мы с вами рассмотрим разные варианты решения этой проблемы.

## Метод generateId

Самый простой вариант - сохранить тот контракт, который уже есть. То есть метод `generateId` вернет
новый уникальный ID-шник, а мы его запишем в экземпляр `Book` и передадим в метод `create`. Иначе
говоря, `create` по-прежнему будет ожидать, что ему на вход передадут экземпляр с заполненным ID.

При реализации этого метода со стороны БД есть два подхода: client-side ID и database-side ID.

### Client-side ID

Этот подход подразумевает, что приложение само ответственно за генерацию ID-шника. База данных никак
в этом не участвует, а лишь ожидает сразу получить его на вход при вставке новой записи. В Java для
этого есть удобный тип данных - `UUID`. Посмотрите на пример `generateId` ниже:

```java
public class BookRepositoryImpl implements BookRepository {

  @Override
  public UUID generateId() {
    return UUID.randomUUID();
  }
}
```

[UUID](https://ru.wikipedia.org/wiki/UUID) - это 128-битное число. Метод же `UUID.randomUUID()`
возвращает случайное значение. Поскольку 128-битное обладает огромным диапазоном, то вероятность
сгенерировать дубликаты ничтожна. А значит, UUID очень удобен для распределенной генерации ID. Вы
можете запустить ваше приложение в нескольких инстансах и на каждом независимо генерировать ID при
вставке новой записи. С огромной вероятностью никаких дублей у вас не возникнет.

> В данном примере метод `BookRepository.generateId` возвращает `UUID` вместо `long`.
> Значит вам придется в вашей доменной модели (в сущностях) также поменять тип ID-шника.

Ниже пример, как в PostgreSQL объявить ID-колонку для хранения `UUID`:

```sql
CREATE TABLE organization
(
    id   UUID PRIMARY KEY,
    name TEXT NOT NULL
);
```

Плюсы такого подхода заключается в удобстве для разработчика. Минусы:

1. Функция `UUID.randomUUID()`
   [может быть синхронизированной](https://github.com/openjdk/jdk8u-ri/blob/b08cddec8625424b1292051088513a60606ef1e9/jdk/src/share/classes/java/security/SecureRandom.java#L467)
   в зависимости от JDK и версии, которую вы используете. Значит, если несколько потоков в вашем
   приложении пытаются сгенерировать `UUID`, то по факту они это будут делать не параллельно, а
   последовательно.
2. ID-шники обычно попадают в URL (`/api/books/{bookId}`), который виден в браузере. Длинные ID
   могут отрицательно сказаться на восприятии продукта, особенно если кто-то будет копировать ссылки и
   пересылать.
3. [Есть много статей о том](https://brandur.org/nanoglyphs/026-ids), что использование `UUID` в
   качестве `PRIMARY KEY` в БД ведет к худшим показателям производительности, чем инкрементирующие
   значения.

В любом случае, в рамках учебного проекта использование UUID в качестве ID абсолютно нормально и не
возбраняется.

## Database-side ID

Здесь мы перекладываем ответственность по генерации ID на сторону БД. Предположим, что наша
таблица `organization` объявлена так:

```sql
CREATE TABLE organization
(
    id   BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL
);
```

Тип `BIGSERIAL` - это всего лишь alias, который создает `sequence` для данной колонки. Вы можете не
передавать значения в `INSERT`, и тогда БД вызовет `sequence` сама и подставит его. А можете передать
явно - результат будет один и тот же.

> Название `sequence` строится как `{table_name}_{column_name}_seq`. В данном случае `sequence` будет называться `organization_id_seq`.

Посмотрите на пример `BookRepositoryImpl.generateId`, где ID генерируется обращением к `sequence`.

```java
public class BookRepositoryImpl implements BookRepository {

  private final Jdbi jdbi;

  @Override
  public long generateId() {
    long value = (long)
        handle.createQuery("SELECT nextval('organization_id_seq') AS value")
            .mapToMap()
            .first()
            .get("value");
    return value;
  }
}
```

БД гарантирует, что каждое обращение к `SELECT nextval('seq_name')` вернет уникальное возрастающее
значение для указанной `sequence`. Так что результат тоже можно использовать в качестве ID.

## Устранение метода generateId

Какой бы алгоритм генерации ID вы ни выбрали, всегда есть возможность совсем не использовать
метод `generateId`. Как этого добиться? Для этого потребуется внести изменения в доменную сущность.
Посмотрите на пример `Book` ниже:

```java
public class Book {

  @Nullable
  private final Long id;
  private final String name;
  private final String author;

  public Book(String name, String author) {
    this.name = name;
    this.author = author;
  }

  public Book withId(long id) {
    return new Book(this.id, name, author);
  }

  @Override
  public boolean equals(Object o) {
    if (this == o) {
      return true;
    }
    if (o == null || getClass() != o.getClass()) {
      return false;
    }
    Book book = (Book) o;
    return id != null && id.equals(book.id);
  }

  @Override
  public int hashCode() {
    return Objects.hash(Book.class);
  }
}
```

> Аннотация `@Nullable` служит просто указателем, что поле может быть равно `null`. Она не вносит никаких изменений в работу кода.

Если мы создаем новый экземпляр `Book`, то ID равен `null`. Когда мы же мы передаем его в
метод `BookRepository.create(...)`, то внутри видим, что ID не заполнен, и сами генерируем его на
месте (одним из алгоритмов выше). Проще говоря, мы полностью убираем из бизнес-кода факт того, что
где-то нужен отдельный вызов `generateId`. Вместо этого мы полагаемся на репозиторий: он сам
подставит ID. Если же мы передадим `Book` с незаполненным ID в метод `update`, то логично, что
придется выбросить исключение, ведь такое поведение не ожидаемо.

Особое внимание обратите на методы `equals/hashCode`. Смысл в том, что если у двух экземпляров `Book`
не заполнены ID, мы всегда считаем их не равными друг другу. Почему же так? Отсутствие ID означает,
что сущность еще не имеет идентичности. Следовательно, нельзя сказать, равна она чему-то или
нет. Поэтому безопаснее всего считать ее не равной никакой другой.

Что касается метода `hashCode`, то здесь тоже все просто. Его значение должно быть
детерминированным: возвращать одно и то же значение при повторных вызовах. Иначе, если складывать
объекты в `Set` или `HashMap`, возможно неожиданное поведение. Поэтому мы возвращаем hash от того
значения, которое гарантированно стабильно. То есть от класса самого объекта.

> В следующем семестре мы будем разбирать [Hibernate](https://hibernate.org/) - самую популярную [ORM](https://ru.wikipedia.org/wiki/ORM) в Java.
> Там подход с _temporary null id_ очень популярен.

Также стоит разъяснить дополнительно метод `create`. Если ID заполняется на внутри метода, а он
возращает `void`, как мы узнаем новый ID и сообщим его клиенту? Все просто: достаточно поменять
сигнатуру метода:

```java
public interface BookRepository {

  Book create(Book book);

  /* остальные методы... */
}
```

Тогда в реализации в качестве ответа нужно будет вернуть `book.withId(generatedId)`.

## Ремарка по UUID

Возможно, вы уже догадались, что при использовании UUID необязательно выделять отдельный
метод `generateId`. Можно создавать его прямо внутри сущности `Book`. Посмотрите на пример кода
ниже:

```java
public class Book {

  private final UUID id;
  private final String name;
  private final String author;

  public Book(String name, String author) {
    this.id = UUID.randomUUID();
    this.name = name;
    this.author = author;
  }

  /* */
}
```

Мы также не передаем ID в конструкторе, но он не заполняется как `null`, а генерируется сразу же на
месте. В большинстве ситуаций такой подход допустим (к тому же, `equals/hashCode` можно будет
написать без лишних "приседаний"). Главный нюанс в том, что метод генерации ID зашит напрямую в
сущность: нарушается принцип Dependency Inversion. Например, если мы захотим поменять алгоритм
генерации UUID так, чтобы каждое следующее значение генерировалось по возрастанию (для этого
есть [библиотеки](https://github.com/f4b6a3/tsid-creator/)), то нам придется менять код в каждой
сущности. В случае же использования метода репозитория `generateId`, достаточно лишь поправить
реализацию и не трогать доменные сущности.

## Выводы по генерации ID

Как для выполнения финального задания, так и для практики этого модуля, вы можете выбрать любой
алгоритм генерации ID из представленных. Но мы хотим, чтобы вы осознавали достоинства и недостатки
каждого решения.