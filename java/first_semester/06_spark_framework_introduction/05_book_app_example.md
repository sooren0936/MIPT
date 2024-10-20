# Пример конечного приложения на Spark Framework

Используя полученные знания, давайте построим приложение по управлению библиотекой на Spark
Framework. У него будут такие endpoint-ы:

1. Создание новой книги.
2. Редактирование существующей книги.
3. Получение конкретной книги по ID.
4. Получение списка всех книг.
5. Удаление книги по ID.

Прежде чем приступать к endpoint'ам, определим корень нашего приложения. Это сама сущность `Book`, а
также методы для ее изменения. Посмотрите на пример ниже:

```java
public class Book {

  private final long id;
  private final String name;
  private final String author;

  public Book(long id, String name, String author) {
    this.id = id;
    this.name = name;
    this.author = author;
  }

  public Book withName(String newName) {
    return new Book(this.id, newName, this.author);
  }

  public Book withAuthor(String newAuthor) {
    return new Book(this.id, this.author, newAuthor);
  }

  /* toString, getters... */

  @Override
  public boolean equals(Object o) {
    if (this == o) {
      return true;
    }
    if (o == null || getClass() != o.getClass()) {
      return false;
    }
    Book book = (Book) o;
    return id == book.id;
  }

  @Override
  public int hashCode() {
    return Objects.hash(id);
  }
}
```

Как видите, мы объявили класс иммутабельным, чтобы не иметь проблем в многопоточной среде.

> Если вы сомневаетесь в значение термина "иммутабельность", вернитесь ко второму модулю,
> где мы разбирали основы Java.

Методы `withName` и `withAuthor` создают новый экземпляр `Book`, но с измененными полями. А вот
на `equals/hashCode` стоит обратить особое внимание. Заметили, что здесь мы проверяем только ID? Это
не ошибка. Когда речь идет об `entity` (то есть о бизнес-объекте, состояние которого может меняться
во времени), сравнивать их между собой нужно по ID. В данный момент эта деталь не очень важна, но
она станет крайне значительной в следующем семестре, когда мы начнем рассматривать Hibernate.

> Если вам интересно узнать подробнее о понятиях "entity" и "value object",
> рекомендуем ознакомиться с [этой статьей на русском языке](https://habr.com/ru/articles/275599/).
> Также советуем почитать [вот этот материал с объяснением термина "чистая архитектура"](https://habr.com/ru/articles/269589/).

Стоит отметить, что использовать `long` в качестве ID напрямую - не самый лучший вариант. Ведь так
тип не очень информативен: `long` - всего лишь число. К тому же, у ID-шников могут быть
ограничения (например, только неотрицательные числа). Лучше было бы объявить отдельный
класс `BookId` и использовать его. Посмотрите на пример ниже:

```java
public class BookId {

  private final long value;

  /* констструктор, equals, hashCode... */
}

public class Book {

  private final BookId id;
  private final String name;
  private final String author;

  /* конструктор, методы... */
}
```

Как бы то ни было, для простоты мы будем и дальше применять обычный `long`. Но держите в голове, что
есть вариант лучше ;)

---

Теперь нам нужно как-то управлять книгами: сохранять новые, находить существующие, удалять и так
далее. Давайте объявим интерфейс, который будет предоставлять необходимые операции. Посмотрите на
пример кода ниже:

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

Первый метод генерирует ID, поскольку он должен быть заполнен для каждой новой сущности.

> Реализация на текущий момент нас не волнует. Об этом мы поговорим далее.

Метод `findAll` возвращает список всех книг, которые сейчас существуют. Есть еще `findById`, который
возращает конкретную книгу по ID. Но он может бросить исключение, потому что такая книга по
заданному ID может отсутствовать.

Далее идет метод `create`. Он добавляет новую книгу. В то же время он может бросить исключение,
если такой ID уже занят.

Метод `update`, в отличие от `create`, находит уже существующую книгу по ID и обновляет ее контент.
Он тоже может бросить исключение, но оно связано с тем, что книга с заданным ID отсутствует.

Последним идет метод `delete`. Он просто удаляет сущность по указанному ID.

Для нашего приложения нам будет достаточно хранить информацию о книгах в памяти процесса. Так что
давайте добавим соответствующую реализацию.

```java
public class InMemoryBookRepository implements BookRepository {

  private final AtomicLong nextId = new AtomicLong(0);
  private final Map<Long, Book> booksMap = new ConcurrentHashMap<>();

  @Override
  public long generateId() {
    return nextId.incrementAndGet();
  }

  @Override
  public List<Book> findAll() {
    return new ArrayList<>(booksMap.values());
  }

  @Override
  public Book findById(long bookId) {
    Book book = booksMap.get(bookId);
    if (book == null) {
      throw new BookNotFoundException("Cannot find book by id=" + bookId);
    }
    return book;
  }

  @Override
  public synchronized void create(Book book) {
    if (booksMap.get(book.getId()) != null) {
      throw new BookIdDuplicatedException("Book with the given id already exists: " + book.getId());
    }
    booksMap.put(book.getId(), book);
  }

  @Override
  public synchronized void update(Book book) {
    if (booksMap.get(book.getId()) == null) {
      throw new BookNotFoundException("Cannot find book by id=" + book.getId());
    }
    booksMap.put(book.getId(), book);
  }

  @Override
  public void delete(long bookId) {
    if (booksMap.remove(bookId) == null) {
      throw new BookNotFoundException("Cannot find book by id=" + bookId);
    }
  }
}
```

Текущий максимальный ID храним в `AtomicLong`. Каждый раз при вызове `generateId` увеличиваем его на 1. 
Сами же книги храним в `Map` в отношении `bookId -> Book`.

Настала очередь use cases, или же доменных сервисов. Это классы, которые инкапсулируют в себе
репозитории и выполняют те или иные бизнес-операции. Посмотрите на пример кода ниже на `BookService`.

```java
public class BookService {

  private final BookRepository bookRepository;

  public BookService(BookRepository bookRepository) {
    this.bookRepository = bookRepository;
  }

  public List<Book> findAll() {
    return bookRepository.findAll();
  }

  public Book findById(long bookId) {
    try {
      return bookRepository.findById(bookId);
    } catch (BookNotFoundException e) {
      throw new BookFindException("Cannot find book by id=" + bookId, e);
    }
  }

  public long create(String name, String author) {
    long bookId = bookRepository.generateId();
    Book book = new Book(bookId, name, author);
    try {
      bookRepository.create(book);
    } catch (BookIdDuplicatedException e) {
      throw new BookCreateException("Cannot create book", e);
    }
    return bookId;
  }

  public void update(long bookId, String name, String author) {
    Book book;
    try {
      book = bookRepository.findById(bookId);
    } catch (BookNotFoundException e) {
      throw new BookUpdateException("Cannot find book with id=" + bookId, e);
    }

    try {
      bookRepository.update(
          book.withName(name)
              .withAuthor(author)
      );
    } catch (BookNotFoundException e) {
      throw new BookUpdateException("Cannot update book with id=" + bookId, e);
    }
  }

  public void delete(long bookId) {
    try {
      bookRepository.delete(bookId);
    } catch (BookNotFoundException e) {
      throw new BookDeleteException("Cannot delete book with id=" + bookId, e);
    }
  }
}
```

Класс `BookService` инкапсулирует `BookRepository` и предоставляет более удобное API для верхних
уровней нашего приложения. Обратите внимание на метод `create`: он возвращает ID книги, которую мы
создали. Поскольку мы генерируем их внутри системы, клиент должен знать, какой ID мы присвоили новой
книге, чтобы он имел возможность ее запросить или удалить в дальнейшем.

Также посмотрите, что мы ловим исключения от репозиториев и оборачиваем их в более высокоуровневые
бизнесовые ошибки. Пользователю сервиса не обязательно знать детали работы репозитория. Ему
достаточно понимать, что та или иная операция завершилась неудачно. Тем не менее, мы все равно
указываем исключение в качестве `cause` (последний параметр конструктора).

> Передача `cause` исключения особенно важна при логировании. Мы поговорим об этом в конце модуля.

> Строго говоря, возвращать `Book` напрямую в качестве результата `find` операции - не лучшая идея.
> Сущности должны использоваться только в тех контекстах, когда мы манипулируем данными (`create/delete/update`).
> При использовании `select` лучше создавать отдельные структуры данных, которые сохранят в себе результат того или иного запроса.
> В рамках этого курса мы будем использовать сущности и для просмотра, и для редактирования, чтобы снизить сложность.
> Но если вам интересна альтернатива, рекомендуем ознакомиться с [этой статьей по CQRS](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs).

С сервисом уже можно добавлять HTTP endpoint-ы. Посмотрите на пример класса `BookController`:

```java
import spark.Service;

public interface Controller {

  void initializeEndpoints();
}

public class BookController implements Controller {

  private static final Logger LOG = LoggerFactory.getLogger(BookController.class);

  private final Service service;
  private final BookService bookService;
  private final ObjectMapper objectMapper;

  public BookController(Service service, BookService bookService, ObjectMapper objectMapper) {
    this.service = service;
    this.bookService = bookService;
    this.objectMapper = objectMapper;
  }

  @Override
  public void initializeEndpoints() {
    createBook();
  }

  private void createBook() {
    service.post(
        "/api/books",
        (Request request, Response response) -> {
          response.type("application/json");
          String body = request.body();
          BookCreateRequest bookCreateRequest = objectMapper.readValue(body,
              BookCreateRequest.class);
          try {
            long bookId = bookService.create(bookCreateRequest.name(), bookCreateRequest.author());
            response.status(201);
            return objectMapper.writeValueAsString(new BookCreateResponse(bookId));
          } catch (BookCreateException e) {
            LOG.warn("Cannot create book", e);
            response.status(400);
            return objectMapper.writeValueAsString(new ErrorResponse(e.getMessage()));
          }
        }
    );
  }
}
```

> Чтобы не показывать вам все решение сразу, здесь мы объявляем только `POST` endpoint для создания книг.
> Остальные добавляются по аналогии.

Класс `BookController` инкапсулирует `BookRepository`, которому делегирует вызовы. По поводу
класса `Service`: именно он стартует `endpoint-ы`. Когда вы вызывали статические методы
`Spark.get/post/delete...`, то он создавался внутри. Здесь же, чтобы не нарушать принцип инверсии
зависимости, мы передаем его в конструкторе.

Теперь объявим класс `Application`, который инкапсулирует список `Controller`, и у каждого
вызывает `initializeEndpoints`:

```java
public class Application {

  private final List<Controller> controllers;

  public Application(List<Controller> controllers) {
    this.controllers = controllers;
  }

  public void start() {
    for (Controller controller : controllers) {
      controller.initializeEndpoints();
    }
  }
}
```

Последний этап - запустить приложение. Для этого составим матрешку объектов в `main`:

```java
import spark.Service;

public class Main {

  public static void main(String[] args) {
    Service service = Service.ignite();
    ObjectMapper objectMapper = new ObjectMapper();
    Application application = new Application(
        List.of(
            new BookController(
                service,
                new BookService(
                    new InMemoryBookRepository()
                ),
                objectMapper
            )
        )
    );
    application.start();
  }
}
```

Если вы все сделали правильно, у вас будет доступен endpoint `POST /api/books` для создания книг.

> Напоминаем, что полный код проекта можно посмотреть в [этом репозитории](https://github.com/SimonHarmonicMinor/java-spark-framework-http-server-example). 