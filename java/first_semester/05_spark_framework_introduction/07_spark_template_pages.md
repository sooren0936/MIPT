# Шаблонизаторы на Spark

Возвращать JSON'ы в качестве ответа - это классический [RESTful подход](https://aws.amazon.com/ru/what-is/restful-api/).
Обычно клиентами таких API являются SPA (Single Page Application) приложения для браузеров. Их пишут на 
таких фреймворках, как [React](https://react.dev/), [Angular](https://angular.io/), [Vue JS](https://vuejs.org/) и других.
Мы не будем затрагивать в курсе эти технологии, потому что они объемные и требуют погружения (да и к тому же не относятся к курсу Java-разработки).

Но как бы то ни было, вам нужно понимать, как отрисовать простые веб-страницы с данными от сервера (тем более, что это потребуется в финальной работе).
Поэтому мы пойдем другим путем: будем возвращать HTML-страницы с нужными данными в качестве ответа при вызове API.

В целом, ничто не помешает нам в тех же самых коллбэках в качестве ответа возвращать строку, которая и содержит HTML-разметку.
Но такой подход многословен и не очень удобен. Поэтому мы воспользуемся шаблонизаторами. Это библиотеки, которые позволяют
динамически формировать веб-страницы в зависимости от ответа сервера.

Spark поддерживает большой список шаблонизаторов. Для примера мы возьмем [Freemarker](https://habr.com/ru/articles/420549/)
Сначала добавьте его в зависимости:

```xml
<dependency>
  <groupId>com.sparkjava</groupId>
  <artifactId>spark-template-freemarker</artifactId>
  <version>2.7.1</version>
</dependency>
```

Теперь объявим HTML-шаблон, куда будем вставлять данные. Мы остановимся на странице с отображением списка книг:

```html
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>Главная страница</title>
  <link rel="stylesheet"
        href="https://cdn.jsdelivr.net/gh/yegor256/tacit@gh-pages/tacit-css-1.6.0.min.css"/>
</head>

<body>

<h1>Список книг</h1>
<table>
  <tr>
    <th>Название</th>
    <th>Автор</th>
  </tr>
    <#list books as book>
      <tr>
        <td>${book.name}</td>
        <td>${book.author}</td>
      </tr>
    </#list>
</table>

</body>

</html>
```

> Библиотека [tacit-css](https://yegor256.github.io/tacit/) - это набор стилей, которые применяются к элементам автоматически.
> Нет необходимости прописывать CSS: ваш сайт сразу будет выглядеть хорошо.

Файл добавляем в путь `src/main/resources/index.ftl`.

Конструкция `<#list books as book>` нужна для итерирования по списку элементов. У каждого же элемента списка мы ожидаем наличие
свойств `name` и `author`, чтобы вывести их в виде таблицы.

Теперь объявим в коде фабрику для настройки `FreeMarkerEngine`:

```java
public class TemplateFactory {

  public static FreeMarkerEngine freeMarkerEngine() {
    Configuration freeMarkerConfiguration = new Configuration(Configuration.VERSION_2_3_0);
    FreeMarkerEngine freeMarkerEngine = new FreeMarkerEngine(freeMarkerConfiguration);
    freeMarkerConfiguration.setTemplateLoader(new ClassTemplateLoader(Main.class, "/"));
    return freeMarkerEngine;
  }
}
```

Объявим еще один контроллер `BookFreemarkerController`. Посмотрите на блок кода ниже:

```java
public class BookFreemarkerController implements Controller {

  private final Service service;
  private final BookService bookService;
  private final FreeMarkerEngine freeMarkerEngine;

  public BookFreemarkerController(
      Service service,
      BookService bookService,
      FreeMarkerEngine freeMarkerEngine
  ) {
    this.service = service;
    this.bookService = bookService;
    this.freeMarkerEngine = freeMarkerEngine;
  }

  @Override
  public void initializeEndpoints() {
    getAllBooks();
  }

  private void getAllBooks() {
    service.get(
        "/",
        (Request request, Response response) -> {
          response.type("text/html; charset=utf-8");
          List<Book> books = bookService.findAll();
          List<Map<String, String>> bookMapList =
              books.stream()
                  .map(book -> Map.of("name", book.getName(), "author", book.getAuthor()))
                  .toList();

          Map<String, Object> model = new HashMap<>();
          model.put("books", bookMapList);
          return freeMarkerEngine.render(new ModelAndView(model, "index.ftl"));
        }
    );
  }
}
```

Мы выполняем следующие действия:

1. Запрашиваем список книг у `BookService`.
2. Преобразовываем их в `List<Map<String, String>>`, где в каждой `Map` хранятся пары с ключами `name` и `author` (те самые, что мы указали в шаблоне).
3. В `HashMap` добавляем свойство `books`, где в качестве значения указываем список, полученный на предыдущем шаге.
   Название `books` фигурирует в шаблоне: по этому списку мы итерируемся.
4. Вызываем функцию `render` у шаблонизатора, где в качестве параметра передаем нужное имя шаблона и его параметры. Полученный HTML возвращаем клиенту.

> Обратите внимание, что у пути нет преффикса `/api`. Обычно их добавляют только на те endpoint-ы, которые возвращают JSON.

Осталось лишь немного поправить функцию `main`, чтобы новый `BookFreemarkerController` также был частью приложения:

```java
public class Main {

  public static void main(String[] args) {
    Service service = Service.ignite();
    ObjectMapper objectMapper = new ObjectMapper();
    final var bookService = new BookService(
        new InMemoryBookRepository()
    );
    Application application = new Application(
        List.of(
            new BookController(
                service,
                bookService,
                objectMapper
            ),
            new BookFreemarkerController(
                service,
                bookService,
                TemplateFactory.freeMarkerEngine()
            )
        )
    );
    application.start();
  }
}
```

Запустите приложение, вызовете несколько раз `POST /api/books`, а потом откройте в браузере `http://localhost:4567`.
Если вы увидели таблицу с созданными книгами, то все сделали верно. Поздравляем!

## Тестирование шаблонизаторов

Шаблонизатор можно тестировать так же, как мы это описывали в предыдущем уроке. Оговорка лишь в том,
что вызываемые endpoint-ы возвращают HTML-строку, а не JSON. Чтобы проверить наличие нужного контента внутри,
можно вызвать `assertThat(html).contains(...)`.