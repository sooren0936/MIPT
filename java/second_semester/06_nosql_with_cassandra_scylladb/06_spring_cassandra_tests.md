### Написание тестов для Spring Data Cassandra

#### Введение

Рассмотрим, как писать тесты для приложений, использующих Spring Data Cassandra. Мы изучим различные подходы к тестированию, включая модульные тесты, интеграционные тесты и использование тестовых контейнеров.

#### Настройка проекта

Для начала работы с тестированием Spring Data Cassandra необходимо добавить соответствующие зависимости в ваш проект. Если вы используете Maven, добавьте следующие зависимости в ваш `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-cassandra</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>cassandra</artifactId>
        <version>1.16.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

#### Модульные тесты

Модульные тесты предназначены для тестирования отдельных компонентов приложения без зависимости от внешних систем, таких как базы данных. Для тестирования репозиториев и сервисов можно использовать моки.

##### Пример модульного теста с использованием Mockito

Рассмотрим пример модульного теста для сервиса, который использует `CassandraTemplate`.

```java
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.springframework.data.cassandra.core.CassandraTemplate;

public class UserServiceUnitTest {

    @Mock
    private CassandraTemplate cassandraTemplate;

    @InjectMocks
    private UserService userService;

    @BeforeEach
    public void setUp() {
        MockitoAnnotations.openMocks(this);
    }

    @Test
    public void testSaveUser() {
        User user = new User();
        user.setId("1");
        user.setName("John Doe");
        user.setEmail("john.doe@example.com");

        userService.saveUser(user);

        verify(cassandraTemplate, times(1)).insert(user);
    }

    @Test
    public void testGetUserById() {
        User user = new User();
        user.setId("1");
        user.setName("John Doe");
        user.setEmail("john.doe@example.com");

        when(cassandraTemplate.selectOneById("1", User.class)).thenReturn(user);

        User result = userService.getUserById("1");

        assertNotNull(result);
        assertEquals("John Doe", result.getName());
        assertEquals("john.doe@example.com", result.getEmail());
    }
}
```

В этом примере мы используем Mockito для создания моков `CassandraTemplate` и тестирования методов `saveUser` и `getUserById`.

#### Интеграционные тесты

Для тестирования Spring Data Cassandra можно использовать встроенную базу данных или тестовые контейнеры.

##### Пример интеграционного теста с использованием встроенной базы данных

Spring Boot предоставляет возможность использовать встроенную базу данных для тестирования. Рассмотрим пример интеграционного теста для репозитория.

```java
import static org.junit.jupiter.api.Assertions.*;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

@SpringBootTest
@TestPropertySource(properties = {
    "spring.data.cassandra.contact-points=127.0.0.1",
    "spring.data.cassandra.port=9042",
    "spring.data.cassandra.keyspace-name=test_keyspace",
    "spring.data.cassandra.local-datacenter=datacenter1"
})
public class UserRepositoryIntegrationTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    public void testSaveAndFindUser() {
        User user = new User();
        user.setId("1");
        user.setName("John Doe");
        user.setEmail("john.doe@example.com");

        userRepository.save(user);

        User foundUser = userRepository.findById("1").orElse(null);

        assertNotNull(foundUser);
        assertEquals("John Doe", foundUser.getName());
        assertEquals("john.doe@example.com", foundUser.getEmail());
    }
}
```

В этом примере мы используем аннотацию `@SpringBootTest` для запуска теста в контексте Spring Boot и аннотацию `@TestPropertySource` для настройки параметров подключения к Cassandra.

##### Пример интеграционного теста с использованием тестовых контейнеров

Тестовые контейнеры позволяют запускать реальные экземпляры баз данных в Docker-контейнерах, что делает тестирование более приближенным к реальным условиям.

Рассмотрим пример интеграционного теста с использованием тестовых контейнеров.

```java
import static org.junit.jupiter.api.Assertions.*;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.testcontainers.containers.CassandraContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
@SpringBootTest
public class UserRepositoryContainerTest {

    @Container
    private static final CassandraContainer<?> cassandraContainer = new CassandraContainer<>("cassandra:3.11.2")
        .withExposedPorts(9042);

    @Autowired
    private UserRepository userRepository;

    @Test
    public void testSaveAndFindUser() {
        User user = new User();
        user.setId("1");
        user.setName("John Doe");
        user.setEmail("john.doe@example.com");

        userRepository.save(user);

        User foundUser = userRepository.findById("1").orElse(null);

        assertNotNull(foundUser);
        assertEquals("John Doe", foundUser.getName());
        assertEquals("john.doe@example.com", foundUser.getEmail());
    }
}
```

В этом примере мы используем аннотацию `@Testcontainers` для запуска контейнера с Cassandra и аннотацию `@Container` для создания экземпляра контейнера.
