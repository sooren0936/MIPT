# HTTP-сервер на Java

Как мы уже выяснили, HTTP - самый популярный протокол для передачи данных по сети. Очень часто Java
используется для так называемой backend-части. То есть это приложение, которое принимает запросы по
HTTP, выполняет какие-то действия и возвращает результат. В этом модуле мы построим приложение по
управлению библиотекой. У него будут такие endpoint-ы:

1. Создание новой книги.
2. Редактирование существующей книги.
3. Получение конкретной книги по ID.
4. Получение списка всех книг.
5. Удаление книги по ID.

> Endpoint-ы (или ручки) - это точки входа в наш HTTP-сервер наш Java. Проще говоря, это методы, которые будут вызваны при поступлении
> того или иного HTTP-запроса от клиента.

> Пример конечного проекта приведен в [этом репозитории](https://github.com/SimonHarmonicMinor/java-spark-framework-http-server-example).
> Рекомендуем ознакомиться с ним уже после прочтения всех уроков модуля, чтобы вы смогли освежить знания, которые получите.

Простейший веб-сервер на Java можно создать, не используя вообще никаких библиотек или фреймворков.
Более того, мы уже это делали в модуле по многопоточности. Посмотрите на пример кода ниже:

```java
class WebServer {

  public static void main(String[] args) throws IOException {
    ExecutorService executorService = Executors.newFixedThreadPool(10);
    ServerSocket socket = new ServerSocket(80);
    while (true) {
      Socket connection = socket.accept();
      executorService.submit(() -> handleRequest(connection));
    }
  }
}
```

Хотя такой вариант и работает, это не то, что нам нужно. Во-первых, `ServerSocket` не заточен именно
на протокол `HTTP`. Он просто принимает любые данные в виде байтов, которые пришли по сети.
Но `HTTP` предполагает некую фиксированную структуру, которую нам бы пришлось парсить
самостоятельно. Поэтому мы пойдем другим путем.

Для наших целей мы будем использовать [Spark Framework](https://habr.com/ru/articles/339684/). Это
микрофреймворк для построения HTTP-сервисов на Java.

> Не путайте Spark Framework и [Apache Spark](https://spark.apache.org/).
> Второй - это система для построения задач по работе с большими данными (та самая Big Data).

Сначала добавьте в зависимости Spark Framework:

```xml

<dependency>
  <groupId>com.sparkjava</groupId>
  <artifactId>spark-core</artifactId>
  <version>2.9.4</version>
</dependency>
```

И давайте заодно настроим логирование. Мы подробно разберем его в конце модуля. Сейчас же нам это
нужно, чтобы видеть в консоли сообщения, которые будет писать Spark Framework. Сначала добавим еще
две зависимости:

```xml

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>2.0.5</version>
</dependency>
```

```xml

<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.4.8</version>
</dependency>
```

А теперь добавьте файл `logback.xml` в директорию `src/main/resources`. Содержимое представлено
ниже:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <layout class="ch.qos.logback.classic.PatternLayout">
      <Pattern>
        %d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n
      </Pattern>
    </layout>
  </appender>

  <root level="info">
    <appender-ref ref="CONSOLE"/>
  </root>

</configuration>
```

Теперь мы готовы. Чтобы добавить наш первый endpoint, достаточно всего лишь указать несколько
строчек:

```java
import spark.Request;
import spark.Response;
import spark.Spark;

public class Main {

  public static void main(String[] args) {
    Spark.get("/hello", (Request request, Response response) -> "Hello, World!");
  }
}
```

После запуска вы увидите в консоли различные логи, которые сообщают информацию о старте сервера.
Рекомендуем почитать их для лучшего понимания предмета. Spark по умолчанию стартует сервер на
порту `4567`. Так что запустите браузер и введите там `http://localhost:4567/hello`. Если вы увидели
надпись `Hello, World!`, то поздравляем вас с вашим первым HTTP endpoint-ом!

> Под "капотом" Spark Framework создает пул потоков, в котором запускаются задачи по обработке запроса (лямбда `(Request request, Response response) -> ...`).
> Значит, что знания из прошлого модуля по concurrency тут очень даже актуальны.
> Помните о нюансах многопоточности, когда будете писать свое приложение на Spark Framework.