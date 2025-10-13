# Dependency Inversion Principle и тестирование

Принцип инверсии зависимости является одним из самых важных в SOLID. Более того, он напрямую связан с тестируемостью кода.
Суть в следующем: классы должны зависеть от абстракций, но не это конкретных реализаций.
Давайте посмотрим еще раз на объявление класса `NotificationService`:

```java
public class NotificationService {

  private static final Logger LOG = LoggerFactory.getLogger(NotificationService.class);

  private final List<NotificationAction> actions;

  public NotificationService(List<NotificationAction> actions) {
    this.actions = actions;
  }

  public void notifyOrdering(OrderId orderId, NotificationType type) {
    boolean foundAction = false;
    for (NotificationAction action : actions) {
      if (action.type().equals(type)) {
        foundAction = true;
        try {
          action.notify(orderId);
          break; // выходим из цикла, если нашли нужный action
        } catch (NotificationActionException e) {
          LOG.error("Couldn't execute action '{}'. Trying next one", action, e);
        }
      }
    }
    if (!foundAction) {
      // если не нашли подходящего action, кидаем исключение
      throw new AbsendNotificationException("Couldn't find NotificationAction for type=" + type);
    }
  }
}
```

Ранее мы обговорили, что у `NotificationAction` есть две реализации: SMS и email.
Здесь у начинающих разработчиков может возникнуть соблазн: давайте сразу присвоим нужные реализации внутри `NotificationService`
и не будем передавать их в конструкторе. Посмотрите на пример кода ниже:

```java
public class NotificationService {

  private static final Logger LOG = LoggerFactory.getLogger(NotificationService.class);

  private final List<NotificationAction> actions = List.of(
      new SmsNotificationAction(),
      new EmailNotificationAction()
  );

  public void notifyOrdering(OrderId orderId, NotificationType type) {
    boolean foundAction = false;
    for (NotificationAction action : actions) {
      if (action.type().equals(type)) {
        foundAction = true;
        try {
          action.notify(orderId);
          break; // выходим из цикла, если нашли нужный action
        } catch (NotificationActionException e) {
          LOG.error("Couldn't execute action '{}'. Trying next one", action, e);
        }
      }
    }
    if (!foundAction) {
      // если не нашли подходящего action, кидаем исключение
      throw new AbsendNotificationException("Couldn't find NotificationAction for type=" + type);
    }
  }
}
```

Кажется, что код стал проще. Ведь теперь нет необходимости передавать список `NotificationAction`
при создании экземпляров `NotificationService`. Достаточно просто написать `new NotificationService()`, и на этом все!
В чем же здесь подвох?

Во-первых, здесь нарушается Open-Closed Principle: если добавится новый вариант отправки уведомлений,
нам придется явно править код в `NotificationService`. Тут вы можете возразить, что нам в любом случае
придется каким-то образом менять код. Есть ли разница, указываем ли мы новую реализацию при создании `NotificationService`
через `new` или же фиксируем ее прямо внутри класса? На самом деле, есть значительная разница.
И заключается она в тестируемости кода.

Давайте предположим, что мы хотим протестировать `NotificationService` с учетом того, что
мы зафиксировали реализации `NotificationAction` прямо внутри класса `NotificationService`.
Посмотрите на пример теста ниже.

```java
class NotificationServiceTest {

  @Test
  void shouldThrowExceptionIfActionsListIsEmpty() {
    NotificationService service = new NotificationService();
    assertThrows(
        AbsendNotificationException.class,
        () -> service.notifyOrdering(new OrderId(1), NotificationType.SMS)
    );
  }
}
```

Здесь мы хотим проверить, что метод `notifyOrdering` бросит `AbsendNotificationException`,
если мы не смогли найти `NotificationAction` с типом `SMS`. По названию же теста понятно,
что мы хотим объявить список `actions` пустым. Но вот загвоздка.
Конструктор `NotificationService` не принимает никаких параметров, а внутри класса "зашита"
функциональность, которая всегда будет создавать список из двух `NotificationAction`.
Следовательно, написать unit-тест на эту часть кода не представляется возможным.

Более того, проблемы могут возникнуть и с инициализацией `SmsNotificationAction` и `EmailNotificationAction`.
Они взаимодействуют с внешним миром: пытаются отправить SMS или письмо по электронной почте.
Значит, при создании эти объекты, скорее всего, пытаются подключиться по сети к каким-то шлюзам,
через которые SMS и email-ы можно отправлять. Проще говоря, в тесте, когда вы создаете объект `NotificationService`,
произойдет также попытка соединения с удаленными ресурсами, которые вы не настраивали. Тут тест либо упадет, либо будет выдавать неожиданный результат.

> Чтобы проверить работоспособность `SmsNotificationAction` и `EmailNotificationAction`,
> можно применить интеграционные тесты. Мы рассмотрим их далее в курсе, когда перейдем к базам данным.

Еще одно следствие нарушения Dependency Inversion Principle - тесты могут упасть из-за багов в других частях кода.
Предположим, что в `SmsNotificationAction` есть баг, из-за которого отправка уведомлений не работает.
Так как `NotificationServiceTest` напрямую зависит от `SmsNotificationAction` (неявно экземпляр этого класса инстанцируется в момент создания `NotificationService`),
то и баги в нем будут проникать в `NotificationServiceTest`. Иначе говоря, ошибка в одной части кода приведет к тому, что каскадом у вас будут падать и другие тесты.
Хотя это неправильно: каждый unit-тест должен проверять лишь изолированную часть кода и не зависеть от изменений, которые происходят в других.

## Пример тестирования с соблюдением Dependency Inversion Principle

Что же делать в этой ситуации? Во-первых, давайте устраним нарушение принципа инверсии зависимости
и вернем внедрение `List<NotificationAction>` через конструктор. Посмотрите на пример кода ниже:

```java
public class NotificationService {

  private static final Logger LOG = LoggerFactory.getLogger(NotificationService.class);

  private final List<NotificationAction> actions;

  public NotificationService(List<NotificationAction> actions) {
    this.actions = actions;
  }

  public void notifyOrdering(OrderId orderId, NotificationType type) {
    boolean foundAction = false;
    for (NotificationAction action : actions) {
      if (action.type().equals(type)) {
        foundAction = true;
        try {
          action.notify(orderId);
          break; // выходим из цикла, если нашли нужный action
        } catch (NotificationActionException e) {
          LOG.error("Couldn't execute action '{}'. Trying next one", action, e);
        }
      }
    }
    if (!foundAction) {
      // если не нашли подходящего action, кидаем исключение
      throw new AbsendNotificationException("Couldn't find NotificationAction for type=" + type);
    }
  }
}
```

Теперь попробуем снова написать тест, проверяющий выбрасывание `AbsendNotificationException`:

```java
class NotificationServiceTest {

  @Test
  void shouldThrowExceptionIfActionsListIsEmpty() {
    NotificationService service = new NotificationService(Collections.emptyList());
    assertThrows(
        AbsendNotificationException.class,
        () -> service.notifyOrdering(new OrderId(1), NotificationType.SMS)
    );
  }
}
```

> Примеры всех тестов можете посмотреть в [репозитории](https://github.com/SimonHarmonicMinor/java-unit-tests-example).

![test empty list](img/test-empty-list.png)

Как видите, тест выполняется успешно. Так как при создании объекта `NotificationService`
мы сами решаем, с какими реализациями `NotificationAction` это нужно делать, то мы можем передать пустой список,
чтобы сделать эмуляцию нужного поведения.

Теперь давайте напишем такой тест, который проверит, что вызов `notify` на нужный `action` был
передан успешно. Посмотрите на пример кода ниже:

```java
class NotificationServiceTest {
  @Test
  void shouldSucceedNotifying() {
    final var actionResult = new AtomicBoolean(false);
    final var notificationAction = new NotificationAction() {
      @Override
      public NotificationType type() {
        return NotificationType.EMAIL;
      }

      @Override
      public void notify(OrderId orderId) {
        actionResult.set(true);
      }
    };

    NotificationService service = new NotificationService(List.of(notificationAction));

    assertDoesNotThrow(
        () -> service.notifyOrdering(new OrderId(1), NotificationType.EMAIL)
    );
    assertTrue(actionResult.get());
  }
}
```

Во-первых, мы создаем собственную реализацию `NotificationAction`, которая существует только в рамках теста.
Не стоит завязываться на те реализации, которые уже есть в коде (`SmsNotificationAction` или `EmailNotificationAction`),
потому что они могут меняться независимо от `NotificationService`. А как мы уже выяснили, мы не хотим,
чтобы `NotificationServiceTest` зависел от изменений в `NotificationAction`.

> Такие реализации, которые мы создаем специально для тестов, называются стабами.

Далее идет `actionResult`. Это просто контейнер, который хранит результат успешного выполнения операции `notify`
и стаба. Если бы `NotificationService.notifyOrdering` возвращал какое-то значение,
нам было бы достаточно проверять его в тестах. Но раз этого нет, то обходимся другими вариантами.

> Далее мы рассмотрим более элегантную альтернативу.

После этого создаем экземпляр `NotificationService` и в конструкторе передаем настроенный стаб
`NotificationAction`. Теперь вызываем `notifyOrdering` и проверяем, что он не бросил никаких исключений.
Ну и наконец смотрим, что в `actionResult` записано `true`.

![test success](img/test-success.png)

Тест работает без ошибок. Но все равно он выглядит немного вычурным. Этот workaround с `actionResult`
воспринимается как-то неправильно. К счастью, есть альтернатива - мокирование.

Моки и стабы - это почти одно и то же. Разница в том, что моки создаются с помощью библиотек динамически,
а стабы мы пишем самостоятельно. Возьмем библиотеку [Mockito](https://site.mockito.org/). Сначала добавьте зависимость на нее в `pom.xml`:

```xml
<dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <version>5.4.0</version>
      <scope>test</scope>
</dependency>
```

Теперь давайте напишем еще один тест `shouldSucceedNotifyingWithMocks`. Посмотрите на пример кода ниже:

```java
class NotificationServiceTest {
  
  @Test
  void shouldSucceedNotifyingWithMocks() {
    final var notificationAction = Mockito.mock(NotificationAction.class);
    Mockito.when(notificationAction.type())
        .thenReturn(NotificationType.EMAIL);

    NotificationService service = new NotificationService(List.of(notificationAction));

    assertDoesNotThrow(
        () -> service.notifyOrdering(new OrderId(1), NotificationType.EMAIL)
    );
    Mockito.verify(notificationAction, Mockito.times(1)).notify(Mockito.any());
  }
}
```

По смыслу этот тест равнозначен предыдущему. Давайте разберемся в нюансах.
Строчка `Mockito.mock(NotificationAction.class)` возвращает мок интерфейса `NotificationAction`.
Мок - это обычная реализация, в которой все методы возвращают дефолтное значение (например, `null`).
При этом тело метода отсутствует (то есть в них нет никакой логики).

> Особенность Mockito еще и в том, что можно создавать моки не только от интерфейсов,
> но даже от классов: библиотека во время выполнения создает наследника.

После этого мы настраиваем вызов `notificationAction.type()` с помощью `Mockito.when`.
Здесь мы сообщаем, что если кто-то вызовет на этом моке метод `type()`, в ответ он получит
`NotificationType.EMAIL`.

Теперь мы передаем настроенный экземпляр `NotificationAction` в конструкторе, как и ранее,
и вызываем `NotificationService.notifyOrdering`. Далее интерес представляет
`Mockito.verify`. В нем мы можем проверить, что определенный метод у мока вызывался нужное количество раз.
Параметр `Mockito.any()` означает, что нас не волнует, какой именно экземпляр `OrderId` был передан
в `NotificationAction.notify`.

В целом, Mockito - не сложная библиотека. Если вы хотите больше узнать о ее возможностях,
можете ознакомиться с [этой ссылкой](https://www.baeldung.com/mockito-series).

## Лучше моки или стабы?

Когда разработчики впервые узнают о Mockito, у них может возникнуть желание использовать эту библиотеку всегда и везде.
Но мы рекомендуем вам отдавать предпочтение стабам и использовать моки только в тех ситуациях, когда стабов недостаточно.

> Классический пример ограничения стабов мы рассмотрели на методе `void`.
> Так как он не возвращает никаких значений, проверить его работу с помощью только лишь стабов - сложная задача.
> В этом случае использование моков оправданно.

Проблема моков в том, что тесты становятся неустойчивыми к рефакторингу. Помните, в первом модуле мы обсуждали инкапсуляцию?
Моки нарушают ее в том числе. Потому что тест начинает слишком много заботиться о том, как работает объект внутри и какие методы он вызывает.
А тесты должны проверять поведение, а не внутреннюю логику вызовов. Обычно для этого достаточно
добавить ассерт на возвращаемое значение.

Наш совет такой: старайтесь всегда использовать стабы, а моки применяйте только тогда,
когда стабы не решают проблему.