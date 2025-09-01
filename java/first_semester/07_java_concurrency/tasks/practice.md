# Практика

Вам необходимо использовать полученные знания для реализации прототипа системы по обогащению
пользовательских сообщений.

## Концепция

Пользователь отправляет DTO в следующем формате:

```java
public class Message {

  private Map<String, String> content;
  private EnrichmentType enrichmentType;

  public enum EnrichmentType {
    MSISDN;
  }
}
```

DTO содержит сообщение `content` и тип обогащения `enrichmentType`.

`MSISDN` - это обогащение по номеру телефона. В качестве результата обогащения добавляются
ключи `firstName` и `lastName` в поле `content`.

## Пример

**Входное сообщение:**

```json
{
  "action": "button_click",
  "page": "book_card",
  "msisdn": "88005553535"
}
```

На Java это может выглядеть так:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, String> input = new HashMap<>();
    input.put("action", "button_click");
    input.put("page", "book_card");
    input.put("msisdn", "88005553535");
    // actions...
  }
}
```

**Обогащенное сообщение:**

```json
{
  "action": "button_click",
  "page": "book_card",
  "msisdn": "88005553535",
  "firstName": "Vasya",
  "lastName": "Ivanov"
}
```

Пример на Java:

```java
public class Enrichment {

  public Map<String, String> enrich(Map<String, String> input) {
    Map<String, String> result = new HashMap<>(input);
    result.put("firstName", "Vasya");
    result.put("lastName", "Ivanov");
    return result;
  }
}
```

## Условия обогащения по MSISDN

1. В `Message.content` должно быть поле `msisdn` со строковым значением. Остальные поля произвольны
   и могут отсутствовать.
2. При алгоритме `EnrichmentType.MSISDN` вычитывается значение `msisdn` и далее происходит
   обогащение. Результат обогащения - это всегда поля `firstName` и `lastName`.
3. Если поля, которые являются результатом обогащения, уже есть в `Message.content`, они
   перезаписываются.
4. Если не получилось обогатить сообщение или ключ `msisdn` отсутствовал, возвращаем сообщение в
   том же виде, как оно пришло.

## Точка входа

Точкой входа в приложение должен являться класс `EnrichmentService` с методом `enrich`.

```java
public class EnrichmentService {

  // возвращается обогащенный (или необогащенный content сообщения)
  public Message enrich(Message message) {...}
}
```

## Требования к реализации

1. Работоспособность проверяем с помощью Unit-тестов.
2. Система должна работать корректно в многопоточной среде. Предполагаем, что метод `enrich` может
   вызываться параллельно из разных потоков.
3. Информация о пользователях хранится в памяти (смотрите пример в приложении 1).
4. Должен быть написан хотя бы один End-to-End тест, который проверяет, что система работает
   корректно в многопоточном режиме. Как вариант, можно использовать `ExecutorService`
   и `CountDownLatch` или `Phaser` для запуска нескольких задач одновременно (смотрите пример в
   приложении 2).
5. Несмотря на то, что в системе всего один тип обогащения (`MSISDN`), код необходимо написать так,
   чтобы он был открыт для дальнейшего расширения (вспомните урок по GoF-паттернам и SOLID).
   Проще говоря, кроме `MSISDN` теоретически могут появиться и другие типы обогащения. Код должен предусмотреть такой исход.

### Приложение 1

Здесь удобно выделить абстракцию `UserRepository`, которая возвращает информацию о пользователе
по `msisdn`.

```java
public interface UserRepository {

  User findByMsisdn(String msisdn);

  void updateUserByMsisdn(String msisdn, User user);
}
```

У нас есть интерфейс, через который мы можем получить информацию о пользователе и обновить ее. Для
нашего примера будет достаточно добавить такую реализацию, которая сложит данные в `Map` или `List`.
Но помните о требовании про потокобезопасность. Возможно, вам стоит использовать здесь
concurrency-коллекцию.

### Приложение 2

End-to-End тест тестирует всю систему целиком, а не каждый компонент по отдельности. В данном
контексте это обычный JUnit-тест, который проверяет не каждый класс в изоляции, а всю «матрешку»
объектов. Посмотрите на пример ниже:

```java
class ApplicationTest {

  @Test
  void shouldReturnEnrichedMessage() {
    Message message = ...;
    Application app = new Application(
        new MessageParser(
            new MessageEnricher(...)
                      )
                  );

    Message enrichedMessage = app.enrich(message);

    // Проверить обогащенное сообщение на соответствие определенным условиям
  }
}
```

Что касается проверки корректности работы в многопоточной среде, то ее можно проверить примерно так:

```java
class ApplicationTest {

  @Test
  void shouldSucceedEnrichmentInConcurrentEnvironmentSuccessfully() {
    Application app = new Application(
        new MessageParser(
            new MessageEnricher(...)
                      )
                  );
    List<Message> enrichmentResults = new CopyOnWriteArrayList<>();
    ExecutorService executorService = Executors.newFixedThreadPool(5);
    CountDownLatch latch = new CountDownLatch(5);
    for (int i = 0; i < 5; i++) {
      executorService.submit(() -> {
        enrichmentResults.add(
            app.enrich(message) // message где-то создается
        );
        latch.countDown();     // уменьшаем значение latch на 1
      });
    }
    latch.await(); // ждем, пока latch не станет равным 0, то есть пока не закончат работу все джобы в цикле

    // проверяем валидность полученных сообщений в enrichmentResult
  }
}
```

> Названия классов и порядок «оборачивания» не являются указанием к действию. Ваша реализация может немного отличаться.


