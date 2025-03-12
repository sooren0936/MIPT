### Интеграция Spring Data Cassandra с Apache Cassandra

#### Введение

Сегодня мы рассмотрим, как интегрировать Apache Cassandra с приложениями на Spring Framework с использованием Spring Data Cassandra. Spring Data Cassandra — это модуль Spring Data, который упрощает работу с Cassandra, предоставляя высокоуровневый API для выполнения операций с базой данных. Мы рассмотрим основные концепции, настройку, примеры кода и лучшие практики.

#### Основные концепции Spring Data Cassandra

Spring Data Cassandra предоставляет следующие ключевые возможности:

1. **CassandraTemplate**: Абстракция для выполнения CQL-запросов, аналогичная `JdbcTemplate` в Spring JDBC.
2. **Repository Support**: Поддержка репозиториев, позволяющая создавать интерфейсы для доступа к данным с минимальным количеством кода.
3. **Entity Mapping**: Аннотации для маппинга Java-классов на таблицы Cassandra.
4. **Query Methods**: Возможность создания запросов на основе имен методов в репозиториях.

#### Настройка проекта

Для начала работы с Spring Data Cassandra необходимо добавить зависимости в ваш проект. Если вы используете Maven, добавьте следующие зависимости в ваш `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-cassandra</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

#### Конфигурация подключения к Cassandra

Для настройки подключения к Cassandra необходимо указать параметры подключения в файле `application.properties` или `application.yml`.

Пример конфигурации в `application.properties`:

```properties
spring.data.cassandra.contact-points=127.0.0.1
spring.data.cassandra.port=9042
spring.data.cassandra.keyspace-name=my_keyspace
spring.data.cassandra.local-datacenter=datacenter1
```

Пример конфигурации в `application.yml`:

```yaml
spring:
  data:
    cassandra:
      contact-points: 127.0.0.1
      port: 9042
      keyspace-name: my_keyspace
      local-datacenter: datacenter1
```

#### Создание Entity

Для маппинга Java-классов на таблицы Cassandra используются аннотации. Рассмотрим пример создания Entity для таблицы `users`.

```java
import org.springframework.data.cassandra.core.mapping.PrimaryKey;
import org.springframework.data.cassandra.core.mapping.Table;

@Table("users")
public class User {

    @PrimaryKey
    private String id;
    private String name;
    private String email;

    // Геттеры и сеттеры
    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}
```

В этом примере мы используем аннотацию `@Table` для указания имени таблицы и `@PrimaryKey` для указания первичного ключа.

#### Использование CassandraTemplate

`CassandraTemplate` — это основной класс для выполнения CQL-запросов в Spring Data Cassandra. Рассмотрим пример использования `CassandraTemplate` для вставки и выборки данных.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.cassandra.core.CassandraTemplate;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    private final CassandraTemplate cassandraTemplate;

    @Autowired
    public UserService(CassandraTemplate cassandraTemplate) {
        this.cassandraTemplate = cassandraTemplate;
    }

    public void saveUser(User user) {
        cassandraTemplate.insert(user);
    }

    public User getUserById(String id) {
        return cassandraTemplate.selectOneById(id, User.class);
    }
}
```

В этом примере мы используем `CassandraTemplate` для вставки и выборки данных из таблицы `users`.

#### Создание репозиториев

Spring Data Cassandra поддерживает создание репозиториев, что позволяет упростить доступ к данным. Рассмотрим пример создания репозитория для сущности `User`.

```java
import org.springframework.data.cassandra.repository.CassandraRepository;

public interface UserRepository extends CassandraRepository<User, String> {
    User findByEmail(String email);
}
```

В этом примере мы создаем интерфейс `UserRepository`, который расширяет `CassandraRepository`. Spring Data автоматически реализует методы для доступа к данным, такие как `findByEmail`.

#### Использование Query Methods

Spring Data Cassandra позволяет создавать запросы на основе имен методов в репозиториях. Рассмотрим пример использования Query Methods.

```java
import org.springframework.data.cassandra.repository.Query;
import org.springframework.data.cassandra.repository.CassandraRepository;

public interface UserRepository extends CassandraRepository<User, String> {
    @Query("SELECT * FROM users WHERE name = ?0 ALLOW FILTERING")
    List<User> findByName(String name);
}
```

В этом примере мы используем аннотацию `@Query` для создания пользовательского запроса.
