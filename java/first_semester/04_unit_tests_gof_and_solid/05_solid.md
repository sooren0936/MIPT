# SOLID

**SOLID** - это аббревиатура, объединяющая 5 принципов. Вместе они образуют подход к разработке
поддерживаемого программного обеспечения.

> "Поддерживаемость" означает, что в код легко вносить изменения: добавлять новую функциональность, править баги.

Рассмотрим каждую из них подробнее.

## Single Responsibility Principle

Принцип единственной ответственности гласит, что каждый класс должен иметь только одну зону
ответственности.

Допустим, мы разрабатываем систему, которая позволяет бронировать отели. При этом у нас есть
следующие функции:

1. Получить информацию о комнате.
2. Забронировать комнату.
3. Отправить уведомление об успешном бронировании.

Попытка реализовать такую функциональность может выглядет так:

```java
public class HotelService {

  public Room findRoom(RoomId roomId) {
    // поиск комнаты по ID
  }

  public Order bookRoom(RoomId roomId) {
    // забронировать комнату
  }

  public List<Order> getOrders() {
    // получить список текущих заказов
  }

  public void notifyOrdering(OrderId orderId) {
    // отправить уведомление об оформлении заказа
  }
}
```

У класса `HotelService` есть три ответственности: поиск информации о комнате, бронирование и
уведомление. А это значит, что причин его менять в три раза больше. Например, из-за бага или
необходимости добавить новую функциональность. Это может привести к тому, что класс станет большим и
плохо поддерживаемым.

Мы можем исправить это, разделив его на три части. Посмотрите на пример кода ниже:

```java
public class RoomService {

  public Room findRoom(RoomId roomId) {
    // поиск комнаты по ID
  }
}

public class OrderService {

  public Order bookRoom(RoomId roomId) {
    // забронировать комнату
  }

  public List<Order> getOrders() {
    // получить список текущих заказов
  }
}

public class NotifyService {

  public void notifyOrdering(OrderId orderId) {
    // отправить уведомление об оформлении заказа
  }
}
```

Теперь каждый класс можно редактировать в отдельности, не затрагивая остальных.

## Open-Closed Principle

Принцип открытости-закрытости диктует, что в процессе добавления функциональности мы должны не
менять старый код, но лишь добавлять новый. Звучит сложно, но самом деле идея тривиальна. Давайте
рассмотрим ее на примере класса `NotificationService` из прошлого пункта. Допустим, что мы хотим
обрабатывать два варианта уведомлений: SMS и email-ы. Мы можем добавить `enum NotificationType` и
передавать его в качестве параметра метода `notifyOrdering`. Посмотрите на пример ниже:

```java
public class NotifyService {

  public void notifyOrdering(OrderId orderId, NotificationType type) {
    if (type.equals(NotificationType.SMS)) {
      // логика обработки SMS
    } else if (type.equals(NotificationType.EMAIL)) {
      // логика обработки EMAIL
    }
  }
}
```

Если появится новый тип уведомлений, нам придется добавлять `if` в метод `notifyOrdering`. А если их
станет достаточно много, метод будет постоянно расти в вертикальном направлении. Это не только
усложняет понимание кода, но и увеличивает риск допустить ошибку. Например, можно забыть проверить
какой-то `NotificationType` или проверить его там, где не нужно.

Чтобы отрефакторить это, давайте введем новый интерфейс `NotificationAction`. Посмотрите на пример
ниже:

```java
public interface NotificationAction {

  NotificationType type();

  void notify(OrderId orderId);
}
```

Каждая его реализация будет представлять собой конкретный тип уведомлений. Посмотрите на код ниже:

```java
public class SmsNotificationAction implements NotificationAction {

  @Override
  public NotificationType type() {
    return NotificationType.SMS;
  }

  @Override
  public void notify(OrderId orderId) {
    // логика уведомлений по sms
  }
}

public class EmailNotificationAction implements NotificationAction {

  @Override
  public NotificationType type() {
    return NotificationType.EMAIL;
  }

  @Override
  public void notify(OrderId orderId) {
    // логика уведомлений по email
  }
}
```

Но как нам применить эту новую абстракцию, если пользователь явно передает `NotificationType`,
который ему нужен? Очень просто: нужно внедрить список `NotificationAction` в `NotificationService`.
Посмотрите на пример ниже:

```java
public class NotificationService {

  private final List<NotificationAction> actions;

  public NotificationService(List<NotificationAction> actions) {
    this.actions = actions;
  }

  public void notifyOrdering(OrderId orderId, NotificationType type) {
    boolean foundAction = false;
    for (NotificationAction action : actions) {
      if (action.type().equals(type)) {
        foundAction = true;
        action.notify(orderId);
        break; // выходим из цикла, если нашли нужный action
      }
    }
    if (!foundAction) {
      // если не нашли подходящего action, кидаем исключение
      throw new AbsendNotificationException("Couldn't find NotificationAction for type=" + type);
    }
  }
}
```

Теперь `NotificationService` полностью закрыт от изменений. Даже если добавится новый способ
уведомлений, нам не придется менять ни `NotificationService`, ни уже существующие
реализации `NotificationAction`. Достаточно будет создать новый класс, который
наследует `NotificationAction`, а также добавить новое `enum` значение для `NotificationType`.

Посмотрите ниже на пример того, как можно создать экземпляр `NotificationService`:

```java
public class Main {

  public static void main(String[] args) {
    NotificationService service = new NotificationService(
        List.of(
            new SmsNotificationAction(),
            new EmailNotificationAction()
        )
    );
    // дальнейшие действия
  }
}
```

Как видите, `NotificationService` ничего не знает о конкретных реализациях для отправки уведомлений,
а оперирует лишь интерфейсом `NotificationAction`.

## Liskov Substitution Principle

Принцип подстановки Барбары Лисков гласит, что если мы зависим от интерфейса, то подстановка любой его реализации не должна ломать код.
Посмотрите еще раз на интерфейс `NotificationAction` ниже:

```java
public interface NotificationAction {

  NotificationType type();

  /**
   * Notifies user of order about its submission.
   *
   * @param orderId id of order
   * @throws NotificationActionException if couldn't send notification due to error
   */
  void notify(OrderId orderId);
}
```

Мы предполагаем, что метод `notify` может завершиться ошибкой, поэтому
в [javadoc](https://ru.wikipedia.org/wiki/Javadoc)
мы явно зафиксировали информацию о типе исключения - `NotificationActionException`.

> Javadoc - это стандарт документирования классов и методов в Java.
> Чтобы сгенерировать javadoc в Idea, нажмите правой кнопкой мыши на элемент и выберите пункт `Generate -> Javadoc`.

Отлично, теперь контракт известен. Давайте немного поправим `NotificationService`:

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

Мы обернули `action.notify` в `try-catch` и в случае ошибки логируем информацию в консоль, но не прерываем выполнения метода.
Вдруг в списке `actions` есть еще один экземпляр с нужным `type`? В этом случае пробрасывать исключение дальше неправильно,
поскольку операция теоретически еще может завершиться успешно.

> Логирование мы подробно рассмотрим далее в курсе. Пока можете считать, что это то же самое, что и вызов `System.out.println`.

Но давайте предположим, что `SmsNotificationAction` не стал соблюдать контракт и в случае ошибки кинул не `NotificationActionException`,
а обычный `RuntimeException`? Значит, исключение не будет отловлено и цикл прервется раньше, чем необходимо.
Проще говоря, бизнес-кейс нарушен.

Так вот, эта ситуация и является примером нарушения принципа подстановки Барбары Лисков. `SmsNotificationAction` кинул не то исключение, которое от него ожидается.
А значит, тот код, который использует интерфейс `NotificationService`, может поломаться, потому что он полагается лишь на контракт.

## Interface Segregation Principle

Принцип разделения интерфейсов является наименее важным, поэтому мы не будем на нем подробно останавливаться.
Суть в том, что не надо добавлять все методы в один
[god interface](https://ru.wikipedia.org/wiki/%D0%91%D0%BE%D0%B6%D0%B5%D1%81%D1%82%D0%B2%D0%B5%D0%BD%D0%BD%D1%8B%D0%B9_%D0%BE%D0%B1%D1%8A%D0%B5%D0%BA%D1%82),
а лучше создать несколько, где каждый объявляет связанный набор методов.

Если вам интересно узнать про этот принцип подробнее, можете ознакомиться с [этой статьей](https://makedev.org/principles/solid/isp.html).
