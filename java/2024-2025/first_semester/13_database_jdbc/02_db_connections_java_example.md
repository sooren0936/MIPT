# Подключение к БД в Java

Давайте попробуем реализовать подключение к БД в Java с помощью JDBC. Интерфейсы уже идут в
комплекте со стандартной библиотекой, но нам все еще нужна реализация. Добавим в зависимость драйвер
для PostgreSQL:

```xml

<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <version>42.6.0</version>
</dependency>
```

Теперь создайте новую БД на PostgreSQL (удобно
для [этого воспользоваться Docker-ом](https://hub.docker.com/_/postgres)). Добавьте новую
таблицу `users`:

```sql
CREATE TABLE users
(
    id        BIGSERIAL PRIMARY KEY,
    firstName VARCHAR(200) NOT NULL,
    lastName  VARCHAR(200) NOT NULL
)
```

Посмотрите на код ниже, который при запуске добавит новую строку в `users`.

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class Main {

  public static void main(String[] args) throws SQLException, ClassNotFoundException {
    Class.forName("org.postgresql.Driver");
    Connection connection =
        DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres", "user",
            "password");
    PreparedStatement preparedStatement = connection.prepareStatement(
        "INSERT INTO users (firstName, lastName) VALUES (?, ?)"
    );
    preparedStatement.setString(1, "Иван");
    preparedStatement.setString(2, "Иванов");
    preparedStatement.executeUpdate();
  }
}
```

> Строчка `Class.forName` нужна, чтобы драйвер подгрузился, и вызов `DriverManager.getConnection` смог его найти.
> Мы не будем больше применять это в курсе, так что здесь не останавливаемся. Но если вам интересно,
> можете почитать про [механизм reflection в Java](https://www.baeldung.com/java-reflection).

Метод `DriverManager.getConnection` возвращает соединение с БД. Передаем:

1. Строку подключения.
2. Пользователя.
3. Пароль.

Строка подключения `JDBC` составлена из частей:

1. `jdbc` - тип подключения.
2. `postgresql` - тип базы данных.
3. `localhost` - хост, к которому выполняем подключение.
4. `5432` - порт.
5. `postgres` - название схемы (или БД в случае PostgreSQL) в рамках СУБД, к которой выполняем
   подключение.

> Если у вас при создании контейнера были заданы иные хост/порт/название схемы, нужно вписать соответствующие.

Далее через `connection` мы
создаем [PreparedStatement](https://www.baeldung.com/java-statement-preparedstatement). Это объект,
который инкапсулирует параметризованное SQL-выражения.

> `Statement` же, как можно догадаться, является SQL-выражением без параметров.

Здесь мы хотим выполнить один `INSERT`. С помощью `preparedStatement.setString` устанавливаем
значения для параметров (указаны как `?`) и выполняем запрос с помощью `executeUpdate`.

> По умолчанию у `Connection`, который возвращает `DriverManager.getConnection`, выставлен параметр [autocommit](https://www.baeldung.com/java-jdbc-auto-commit).
> Это значит, что в момент выполнения запроса (`executeUpdate` в нашем случае), он сразу же закоммитится.
> В целом, такая практика считается не очень удачной, потому что тогда невозможно откатить изменения, которые мы сделали в рамках транзакции, вызвав `rollback`.
> Далее мы рассмотрим другие варианты.

Запустите Java-приложение. Если все отработало без ошибок, то в БД вы увидите новую строчку в
таблице `users`.

## Высокоуровневые библиотеки

Несмотря на то, что интерфейсы `JDBC` позволяют выполнять любые операции с БД, напрямую это API в
коде обычно не используют. Оно довольно низкоуровневое, и действительно сложные вещи на нем
плохо воспринимаются. Поэтому в Java есть ряд библиотек, которые внутри используют `JDBC`, но
предоставляют более понятный и высокоуровневый API, чтобы код работы с БД выглядел проще. Одна из
таких библиотек - [JDBI](https://jdbi.org/).

Сначала добавим зависимость:

```xml

<dependency>
  <groupId>org.jdbi</groupId>
  <artifactId>jdbi3-core</artifactId>
  <version>3.40.0</version>
</dependency>
```

Теперь немного перепишем код обращения к БД.

```java
public class Main {

  public static void main(String[] args) {
    Jdbi jdbi = Jdbi.create("jdbc:postgresql://localhost:5432/postgres", "user", "password");
    jdbi.useTransaction((Handle handle) -> {
      handle
          .createUpdate("INSERT INTO users (firstName, lastName) VALUES (:firstName, :lastName)")
          .bind("firstName", "Иван")
          .bind("lastName", "Иванов")
          .execute();
    });
  }
}
```

Как видите, он стал немного проще. В `useTransaction` мы передаем лямбду, которая будет выполняться
в рамках одной транзакции и успешно закоммитится по ее завершению. Также для удобства мы применяем
именованные параметры (`:firstName`, `:lastName`) вместо позиционных (`?`).

Запустите код - он даст тот же результат.

Хорошо, напишем что-нибудь посложнее. Например, давайте проверим, что JDBI корректно обрабатывает
rollback-и. Посмотрите на пример кода ниже:

```java
public class Main {

  public static void main(String[] args) {
    Jdbi jdbi = Jdbi.create("jdbc:postgresql://localhost:5432/postgres", "user", "password");
    jdbi.useTransaction((Handle handle) -> {
      handle
          .createUpdate("INSERT INTO users (firstName, lastName) VALUES (:firstName, :lastName)")
          .bind("firstName", "Иван")
          .bind("lastName", "Иванов")
          .execute();

      handle.createUpdate("INSERT INTO users (firstName) VALUES (:firstName)")
          .bind("firstName", "Джон")
          .execute();
    });
  }
}
```

Во втором `INSERT` есть ошибка: мы намеренно упустили одну колонку. Поскольку и `firstName`,
и `lastName` отмечены как `not null`, пропуск колонки в `INSERT` равносильно тому, что мы записали
бы `null` напрямую. Значит, этот statement должен завершиться ошибкой. А следовательно, и все
выражения, которые были запущены в данной лямбде, откатятся. Если вы запустите этот код, то увидите
примерно такое сообщение в консоли:

```
Exception in thread "main" org.jdbi.v3.core.statement.UnableToExecuteStatementException: org.postgresql.util.PSQLException: ERROR: null value in column "lastname" of relation "users" violates not-null constraint
  Подробности: Failing row contains (6, Джон, null). [statement:"INSERT INTO users (firstName) VALUES (:firstName)", arguments:{positional:{}, named:{firstName:Джон}, finder:[]}]
```

Посмотрите содержимое таблицы `users` в БД: новая запись `Иван-Иванов` не добавилась. А значит
коммиты и роллбэки корректно отрабатываются в JDBI.

> В мире Java есть еще одна очень популярная SQL-библиотека - [JOOQ](https://www.jooq.org/).
> Ее особенность в том, что JOOQ может сканировать структуру БД и генерировать классы для типизированных SQL.
> Проще говоря, вы не пишите `INSERT/DELETE/UPDATE/SELECT` строками, а вызываете функции `jooq.select(...).from(...)...`.
> Дальнейшие примеры в курсе будут на JDBI. Но вам не запрещается использовать JOOQ ни в домашних заданиях, ни в финальной работе.
> Так что, если чувствуете здесь дополнительный интерес, можете попробовать JOOQ.