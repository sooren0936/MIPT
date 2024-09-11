# Конфигурирование Java-приложений

Посмотрите еще раз на этот код ниже:

```java
public class Main {

  public static void main(String[] args) {
    Flyway flyway =
        Flyway.configure()
            .outOfOrder(true)
            .locations("classpath:db/migrations")
            .dataSource("jdbc:postgresql://localhost:5432/postgres", "user", "password")
            .load();
    flyway.migrate();
  }
}
```

Как думаете, в чем его проблема? Дадим вам подсказку. Что если путь подключения к БД отличается? Что
если вам нужно деплоить сервис на два разных стенда, где пользователь и пароль у БД разные? В этой
ситуации вам придется менять код и компилировать его под разные среды. Как можете догадаться, это не
лучший подход. Намного удобнее написать код один раз, а затем "подкладывать" нужные параметры
подключения. Для этого нам понадобится библиотека конфигурации. Мы
воспользуемся [Lightbend Config](https://github.com/lightbend/config).

> Как и в предыдущих примерах, мы не ставим ограничение использовать именно эту библиотеку, но примеры в курсе будут с ее использованием.

Сначала добавим зависимость:

```xml

<dependency>
  <groupId>com.typesafe</groupId>
  <artifactId>config</artifactId>
  <version>1.4.2</version>
</dependency>
```

Теперь создайте файл `application.properties` в директории `src/main/resources`:

```properties
app.database.url=jdbc:postgresql://localhost:5432/postgres
app.database.user=user
app.database.password=password
```

Запишем сюда параметры БД, на которую накатывали прошлые миграции (`organization`, `department`,
`employee`).

Теперь немного поменяем код в `main`, чтобы вычитывать параметры из `application.properties`:

```java
import com.typesafe.config.Config;
import com.typesafe.config.ConfigFactory;
import org.flywaydb.core.Flyway;

public class Main {

  public static void main(String[] args) {
    Config config = ConfigFactory.load();

    Flyway flyway =
        Flyway.configure()
            .outOfOrder(true)
            .locations("classpath:db/migrations")
            .dataSource(config.getString("app.database.url"), config.getString("app.database.user"),
                config.getString("app.database.password"))
            .load();
    flyway.migrate();
  }
}
```

Как видите, мы подставляем название конфига, вместо того
чтобы [хардкодить](https://clov.net/hardcode) конкретные значения. Запустите программу. Если она
выполнилась без ошибок, значит, подключение к БД отработало верно.

Теперь протестируем некорректное поведение. Поменяйте значение `app.database.password` на неверное и
запустите `main` снова. Вы увидите примерно такое сообщение об ошибке:

```
FATAL: password authentication failed for user "user"
```

Отлично! Значения из конфига подтягиваются корректно. Поменяем обратно
значение `app.database.password` на корректное.

Вопрос: а как нам переопределять значения при запуске? Ведь именно в этом и был смысл внедрения
библиотеки конфигурации. Это можно сделать в момент запуска результирующего `jar` файла. Сначала
запустите команду `package` в Maven. Теперь откройте директорию `target` и найдите там `jar` файл с
суффиксом `shade`. Запустите его следующей командой в консоли:

```shell
java -jar java-spark-framework-http-server-example-1.0-SNAPSHOT-shaded.jar
```

Команда отработает без ошибок, потому что значения конфига вычитаются из `application.properties`,
который запаковывается вместе с кодом. Теперь попробуем поменять значения конфига так,
чтобы `app.database.user` указывал на неверного пользователя:

```shell
java -Dapp.database.user=wrong_user -jar java-spark-framework-http-server-example-1.0-SNAPSHOT-shaded.jar
```

В консоли вы увидите примерно следующее сообщение об ошибке:

```
Message    : FATAL: password authentication failed for user "wrong_user"
```

> Если вы используете Windows, то указание конфигов через точку (`-Dapp.database.user`) может привести к ошибке парсинга.
> В таком случае, рекомендуем вам поставить [Bash for Windows](https://itsfoss.com/install-bash-on-windows/).

Следовательно, переопределение конфигов работает корректно. То есть в `application.properties` мы
сможем указать значения по умолчанию, а потом с помощью JVM-параметров переопределить при деплое на
ту или иную среду.

> Библиотека позволяет не только переопределять конкретные параметры, но и передавать целиком [новый файл с конфигурацией](https://github.com/lightbend/config#standard-behavior).


