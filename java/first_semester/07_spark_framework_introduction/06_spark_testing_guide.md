# Тестирование приложения на Spark

Мы с вами разобрались, как проверять работоспособность endpoint-ов с помощью Postman или другого HTTP-клиента.
Но такой вариант не автоматизирует проверки, как в случае с unit-тестами, что мы разбирали в прошлом модуле.
К счастью, в Java есть способ протестировать и более сложные вещи. Посмотрите на пример кода ниже с тестированием `BookController`:

```java
class BookControllerTest {

  private Service service;

  @BeforeEach
  void befofeEach() {
    service = Service.ignite();
  }

  @AfterEach
  void afterEach() {
    service.stop();
    service.awaitStop();
  }

  @Test
  void should201IfBookIsSuccessfullyCreated() throws Exception {
    BookService bookService = Mockito.mock(BookService.class);
    ObjectMapper objectMapper = new ObjectMapper();
    Application application = new Application(
        List.of(
            new BookController(
                service,
                bookService,
                objectMapper
            )
        )
    );
    Mockito.when(bookService.create("book", "jack"))
        .thenReturn(45L);
    application.start();
    service.awaitInitialization();

    HttpResponse<String> response = HttpClient.newHttpClient()
        .send(
            HttpRequest.newBuilder()
                .POST(
                    BodyPublishers.ofString(
                        """
                            { "name": "book", "author": "jack" }"""
                    )
                )
                .uri(URI.create("http://localhost:%d/api/books".formatted(service.port())))
                .build(),
            BodyHandlers.ofString(UTF_8)
        );

    assertEquals(201, response.statusCode());
    BookCreateResponse bookCreateResponse =
        objectMapper.readValue(response.body(), BookCreateResponse.class);
    assertEquals(45L, bookCreateResponse.id());
  }
}
```

В колбэке `BeforeEach` (запускается перед каждым тестом) мы создаем новый экземпляр `spark.Service`,
который отвечает за обработку запросов. В `AfterEach` же мы выключаем его, чтобы тесты не зависели друг от друга.

Далее в тесте мы проверяем, что endpoint возвращает `201` в случае успешного создания книги.
Как можете заметить, инициализация очень похоже на то, что написано в `main`, только вместо реального
экземпляра `BookService` мы передаем мок.

Потом стартуем приложение вызовом `application.start` и ждем, пока `Service` начнет принимать запросы: `service.awaitInitialization()`;

Теперь можно осуществить HTTP-запрос. Для этого мы используем `HttpClient`, который встроен в Java начиная с 11-ой версии.
Проверяем статус-код ответа и смотрим, что в теле получили тот id-шник, который настраивали в моке.

Ничего сложного, как можете заметить. В практическом задании вам нужно будет написать нечто похожее, только
вместо мока `BookService` нужно будет воссоздать всю матрешку объектов, как в `main` (такие тесты называют _функциональными_).