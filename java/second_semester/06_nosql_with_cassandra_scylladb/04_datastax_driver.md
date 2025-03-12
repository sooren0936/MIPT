### Лекция: Datastax Java Driver для Apache Cassandra

#### Введение

Сегодня мы поговорим о Datastax Java Driver для Apache Cassandra. Этот драйвер является одним из наиболее популярных и мощных инструментов для взаимодействия с Cassandra на языке Java. Мы рассмотрим архитектуру драйвера, его основные компоненты, а также примеры кода, которые помогут вам лучше понять, как использовать этот инструмент в своих проектах.

#### Архитектура Datastax Java Driver

Datastax Java Driver представляет собой высокоуровневую библиотеку, которая абстрагирует сложности взаимодействия с Cassandra. Он предоставляет удобный API для выполнения запросов, управления соединениями и обработки результатов.

Основные компоненты драйвера:

1. **Cluster**: Этот объект представляет собой точку входа для взаимодействия с кластером Cassandra. Он управляет соединениями с узлами кластера и предоставляет методы для выполнения запросов.

2. **Session**: Сессия используется для выполнения запросов к Cassandra. Она создается из объекта Cluster и предоставляет методы для выполнения CQL-запросов.

3. **Statement**: Этот объект представляет собой CQL-запрос, который может быть выполнен с помощью Session. Существуют различные типы Statement, такие как SimpleStatement, PreparedStatement и BoundStatement.

4. **ResultSet**: Результат выполнения запроса возвращается в виде объекта ResultSet. Этот объект предоставляет методы для итерации по результатам и извлечения данных.

5. **Row**: Каждая строка результата представлена объектом Row. Этот объект предоставляет методы для извлечения значений из строки.

#### Настройка и подключение

Для начала работы с Datastax Java Driver необходимо добавить зависимость в ваш проект. Если вы используете Maven, добавьте следующий фрагмент в ваш `pom.xml`:

```xml
<dependency>
    <groupId>com.datastax.oss</groupId>
    <artifactId>java-driver-core</artifactId>
    <version>4.13.0</version>
</dependency>
```

После добавления зависимости можно приступить к настройке подключения к кластеру Cassandra. Вот пример кода для создания объекта Cluster и Session:

```java
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.CqlSessionBuilder;

public class CassandraConnector {

    private CqlSession session;

    public void connect(String node, Integer port, String dataCenter) {
        CqlSessionBuilder builder = CqlSession.builder();
        builder.addContactPoint(new InetSocketAddress(node, port));
        builder.withLocalDatacenter(dataCenter);
        session = builder.build();
    }

    public CqlSession getSession() {
        return this.session;
    }

    public void close() {
        session.close();
    }
}
```

В этом примере мы создаем объект `CqlSession`, который используется для выполнения запросов к Cassandra. Метод `connect` принимает адрес узла, порт и название дата-центра, к которому мы подключаемся.

#### Выполнение запросов

Теперь, когда у нас есть подключение к кластеру, мы можем выполнять запросы. Рассмотрим несколько примеров.

##### Простой запрос (SimpleStatement)

```java
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.cql.SimpleStatement;
import com.datastax.oss.driver.api.core.cql.ResultSet;
import com.datastax.oss.driver.api.core.cql.Row;

public class SimpleStatementExample {

    public static void main(String[] args) {
        CassandraConnector client = new CassandraConnector();
        client.connect("127.0.0.1", 9042, "datacenter1");

        CqlSession session = client.getSession();

        SimpleStatement statement = SimpleStatement.newInstance("SELECT * FROM my_keyspace.my_table");
        ResultSet resultSet = session.execute(statement);

        for (Row row : resultSet) {
            System.out.println(row.getString("column_name"));
        }

        client.close();
    }
}
```

В этом примере мы создаем объект `SimpleStatement`, который содержит CQL-запрос для выборки всех строк из таблицы `my_table`. Затем мы выполняем этот запрос и итерируем по результатам.

##### Подготовленный запрос (PreparedStatement)

Подготовленные запросы полезны, когда вы хотите выполнить один и тот же запрос несколько раз с разными параметрами. Они также помогают предотвратить SQL-инъекции.

```java
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.cql.PreparedStatement;
import com.datastax.oss.driver.api.core.cql.BoundStatement;
import com.datastax.oss.driver.api.core.cql.ResultSet;
import com.datastax.oss.driver.api.core.cql.Row;

public class PreparedStatementExample {

    public static void main(String[] args) {
        CassandraConnector client = new CassandraConnector();
        client.connect("127.0.0.1", 9042, "datacenter1");

        CqlSession session = client.getSession();

        PreparedStatement preparedStatement = session.prepare(
            "INSERT INTO my_keyspace.my_table (id, name) VALUES (?, ?)"
        );

        BoundStatement boundStatement = preparedStatement.bind(
            java.util.UUID.randomUUID(), "John Doe"
        );

        session.execute(boundStatement);

        client.close();
    }
}
```

В этом примере мы создаем подготовленный запрос для вставки данных в таблицу `my_table`. Затем мы связываем параметры с запросом и выполняем его.

#### Управление соединениями

Datastax Java Driver автоматически управляет соединениями с узлами кластера. Однако вы можете настроить параметры соединения, такие как количество потоков, таймауты и политики повторных попыток.

Пример настройки соединения:

```java
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.CqlSessionBuilder;
import com.datastax.oss.driver.api.core.config.DriverConfigLoader;
import com.datastax.oss.driver.api.core.context.DriverContext;
import com.datastax.oss.driver.api.core.session.SessionBuilder;

public class AdvancedCassandraConnector {

    private CqlSession session;

    public void connect(String node, Integer port, String dataCenter) {
        DriverConfigLoader configLoader = DriverConfigLoader.fromClasspath("application.conf");
        CqlSessionBuilder builder = CqlSession.builder();
        builder.withConfigLoader(configLoader);
        builder.addContactPoint(new InetSocketAddress(node, port));
        builder.withLocalDatacenter(dataCenter);
        session = builder.build();
    }

    public CqlSession getSession() {
        return this.session;
    }

    public void close() {
        session.close();
    }
}
```

В этом примере мы используем конфигурационный файл `application.conf` для настройки параметров драйвера. Этот файл может содержать настройки для пула соединений, таймаутов и других параметров.

#### Обработка ошибок и повторные попытки

Datastax Java Driver предоставляет механизмы для обработки ошибок и автоматических повторных попыток. Вы можете настроить политики повторных попыток для различных типов ошибок.

Пример настройки политики повторных попыток:

```java
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.CqlSessionBuilder;
import com.datastax.oss.driver.api.core.config.DriverConfigLoader;
import com.datastax.oss.driver.api.core.retry.RetryPolicy;
import com.datastax.oss.driver.internal.core.retry.DefaultRetryPolicy;

public class RetryPolicyExample {

    public static void main(String[] args) {
        DriverConfigLoader configLoader = DriverConfigLoader.fromClasspath("application.conf");
        CqlSessionBuilder builder = CqlSession.builder();
        builder.withConfigLoader(configLoader);
        builder.withRetryPolicy(new DefaultRetryPolicy());

        CqlSession session = builder.build();

        // Ваш код для выполнения запросов

        session.close();
    }
}
```

В этом примере мы используем стандартную политику повторных попыток `DefaultRetryPolicy`. Вы также можете создать свою собственную политику, реализовав интерфейс `RetryPolicy`.

#### Асинхронные запросы

Datastax Java Driver поддерживает асинхронные запросы, что позволяет выполнять запросы без блокировки основного потока. Это особенно полезно для высоконагруженных приложений.

Пример асинхронного запроса:

```java
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.cql.SimpleStatement;
import com.datastax.oss.driver.api.core.cql.AsyncResultSet;
import java.util.concurrent.CompletionStage;

public class AsyncExample {

    public static void main(String[] args) {
        CassandraConnector client = new CassandraConnector();
        client.connect("127.0.0.1", 9042, "datacenter1");

        CqlSession session = client.getSession();

        SimpleStatement statement = SimpleStatement.newInstance("SELECT * FROM my_keyspace.my_table");
        CompletionStage<AsyncResultSet> future = session.executeAsync(statement);

        future.whenComplete((resultSet, throwable) -> {
            if (throwable != null) {
                System.err.println("Ошибка при выполнении запроса: " + throwable.getMessage());
            } else {
                resultSet.currentPage().forEach(row -> {
                    System.out.println(row.getString("column_name"));
                });
            }
        });

        client.close();
    }
}
```

В этом примере мы используем метод `executeAsync` для выполнения запроса асинхронно. Результат возвращается в виде `CompletionStage`, который позволяет обрабатывать результат в отдельном потоке.

#### **Лучшие практики**
1. **Использование PreparedStatement для многократных запросов**: Это повышает производительность и безопасность.
2. **Минимизация использования SimpleStatement**: SimpleStatement подходит только для однократных запросов.
