# GoF паттерны

> Мы немного отвлечемся от тестов и рассмотрим паттерны проектирования программного обеспечения.
> В конце модуля мы снова вернемся к тестам, и вы поймете, что тестируемость продукта напрямую связана
> с тем, как вы организуете код.

Большинство программных продуктов в современном мире разрабатываются с помощью Agile-подобных
методологий (Scrum, Kanban и так далее). Во-первых, это дает заказчикам волю выстраивать требования
к конечной системе постепенно. А во-вторых, позволяет наблюдать за тем, как развивается продукт в
реальном времени. Текущее состояние отслеживается на *демо-показах*, которые могут проходить,
например, раз в две недели.

Правда, с другой стороны, это создает дополнительные сложности для разработчиков. Из-за
того что требования к системе могут поменяться в любой момент, код продукта должен быть выстроен в
соответствии с этими ожиданиями. На протяжении многих лет разработчики описывали лучшие практики
структуризации программного кода для наилучшей поддержки в будущем. В 1994 году произошло
знаменательное событие: выход в свет книги "Design Patterns: Elements of Reusable Object-Oriented
Software". Данный труд собрал воедино опыт сотен разработчиков в виде конкретных рецептов.

> Design Patterns были написаны четырьмя разработчиками, которых впоследствии назвали "Бандой четырех" (Gang of Four).
> Отсюда и название — GoF паттерны.

В книге описаны 23 различных паттерна (шаблона проектирования). Мы рассмотрим лишь некоторые из
них. Тем не менее крайне рекомендуем прочитать книгу целиком и попробовать реализовать предложенные
паттерны самостоятельно.

> Несмотря на то что книга вышла более 25 лет назад, многие предлагаемые решения до сих пор остаются актуальными.
> Правда, стоит заметить, что прогресс тоже не стоит на месте.
> Ряд паттернов теперь реализуются стандартными возможностями языков программирования.
> Например, Stream API, который мы будем рассматривать далее, целиком построен на одном из GoF паттернов.
> Попробуйте догадаться, на каком именно.

## Декоратор

Пожалуй, один из самых популярных шаблонов проектирования в ООП языках и Java в частности.
Рассмотрим следующий пример. Допустим, у нас есть сервис, который обновляет информацию о пользователе в
БД.

```java
public interface UserService {
  void updateUser(Long id, String firstName, String lastName);
}

public class UserServiceImpl implements UserService {
  private final UserRepository userRepository;

  @Override
  public void updateUser(Long id, String firstName, String lastName) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new NoSuchElementException("No User with ID=" + id));
    user.update(firstName, lastName);
  }
}
```

> Метод `UserRepository.findById` возвращает `Optional`. Мы рассматривали его в предыдущем модуле.
> Если нужно освежить воспоминания, то вернитесь к нему еще раз в раздел "Не возвращайте null".

Далее нам поступает новое требование. Если попытка обновления завершилась с ошибкой, необходимо
фиксировать информацию в системе аудита. Мы можем просто добавить соответствующую логику в метод.

```java
public class UserServiceImpl implements UserService {
  private final UserRepository userRepository;
  private final AuditService auditService;

  @Override
  public void updateUser(Long id, String firstName, String lastName) {
    try {
      User user = userRepository.findById(id)
          .orElseThrow(() -> new NoSuchElementException("No User with ID=" + id));
      user.update(firstName, lastName);
    } catch (RuntimeException e) {
      auditService.auditFailedUserCreation(id, e);
      throw e;
    }
  }
}
```

Проходит время, и к нам поступает новое требование. Если создание пользователя было успешным, нужно
отправлять e-mail администратору.

```java
public class UserServiceImpl implements UserService {
  private final UserRepository userRepository;
  private final AuditService auditService;
  private final EmailService emailService;

  @Override
  public void updateUser(Long id, String firstName, String lastName) {
    try {
      User user = userRepository.findById(id)
          .orElseThrow(() -> new NoSuchElementException("No User with ID=" + id));
      user.update(firstName, lastName);
      emailService.notifyAdmin(new UserUpdatedEvent(id));
    }
    catch (RuntimeException e) {
      auditService.auditFailedUserCreation(id, e);
      throw e;
    }
  }
}
```

У этого класса есть ряд проблем.
1. Большое количество зависимостей, которое затрудняет понимание того, зачем сервис нужен.
2. Класс трудно тестировать.
3. При добавлении новой функциональности есть риск того, что старые тесты упадут.

Декоратор предлагает иной подход.
Каждый раз, когда мы хотим добавить новую функциональность, мы создаем новую имплементацию `UserService`,
которая инкапсулирует в себе логику и проксирует вызовы далее по необходимости.

Схематично это можно изобразить следующим образом.

![decorator](img/decorator.drawio.png)

Каждый класс отвечает за одну функциональность. При необходимости декоратор может делегировать работу далее по цепочке.

Если переписать код, он будет выглядеть следующим образом.

```java
public class UserServiceImpl implements UserService {
  private final UserRepository userRepository;

  @Override
  public void updateUser(Long id, String firstName, String lastName) {
      User user = userRepository.findById(id)
          .orElseThrow(() -> new NoSuchElementException("No User with ID=" + id));
      user.update(firstName, lastName);
  }
}

public class AuditUserService implements UserService {
  private final UserService origin;
  private final AuditService auditService;

  @Override
  public void updateUser(Long id, String firstName, String lastName) {
    try {
      origin.updateUser(id, firstName, lastName);
    }
    catch (RuntimeException e) {
      auditService.auditFailedUserCreation(id, e);
      throw e;
    }
  }
}

public class EmailUserService implements UserService {
  private final UserService origin;
  private final EmailService emailService;

  @Override
  public void updateUser(Long id, String firstName, String lastName) {
    origin.updateUser(id, firstName, lastName);
    emailService.notifyAdmin(new UserUpdatedEvent(id));
  }
}
```

Теперь каждый класс выполняет одну заданную функцию. Их можно протестировать как изолированно друг от друга, так и вместе.
Более того, декораторы зависят от интерфейса `UserService`, а не от имплементации `UserServiceImpl`.
Это дает возможность менять порядок вызовов и добавлять новые декораторы при необходимости.

## Фабричный метод

Допустим, мы разрабатываем ПО для автоматизации банковского обслуживания.
Банк предоставляет три типа кредитных карт: `Standard`, `Gold` и `Platinum`.
У каждого объекта карты есть два метода: `getCreditLimit` (остаток лимитного кредита) и `getAnnualCharge` (ежегодный платеж за использование).

```java
public interface Card {
  BigDecimal getCreditLimit();
  BigDecimal getAnnualCharge();
}

public interface StandardCard implements Card {
  ...
}

public interface GoldCard implements Card {
  ...
}

public interface PlatinumCard implements Card {
  ...
}
```

Банк хочет, чтобы пользователи могли заказывать карты онлайн.
Что ж, нам нужно как-то создавать объекты `Card`.
Для этого можно воспользоваться простейшей статической фабрикой.

```java
public class CardFactory {
  public static Card newCard(ClientAttributes attributes) {
    // some calculations
    // ...
    // return new Card instance
  }
}
```

Со временем появились новые требования.
Например, у клиента не может быть более пяти стандартных карт.
Чтобы перейти со `Standard` на `Gold`, нужно либо обладать определенным уровнем годового дохода, либо заплатить входную комиссию.
Все клиенты, которые приобретают платиновую карту, переходят в категорию VIP.

Подобных условий может быть большое разнообразие. Также они могут меняться со временем.
Если мы попытаемся учесть все данные требования в рамках метода `newCard`, то столкнемся с рядом проблем:

1. При появлении нового требования со стороны бизнеса (например, добавление нового типа карты), придется менять логику `CardFactory.newCard`
   (это нарушает [Open-Closed principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle#:~:text=In%20object%2Doriented%20programming%2C%20the,without%20modifying%20its%20source%20code.)).
2. Метод содержит в себе слишком много бизнес-логики, из-за чего его трудно будет тестировать.
3. Если в методе есть баг, это может затронуть процесс получения любых карт.

Паттерн [фабричный метод](https://ru.wikipedia.org/wiki/%D0%A4%D0%B0%D0%B1%D1%80%D0%B8%D1%87%D0%BD%D1%8B%D0%B9_%D0%BC%D0%B5%D1%82%D0%BE%D0%B4_(%D1%88%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD_%D0%BF%D1%80%D0%BE%D0%B5%D0%BA%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F))
помогает решить эту проблему.
Во-первых, сделаем `CardFactory` интерфейсом.

```java
public interface CardFactory {
  Card newCard(ClientAttributes attributes);
}
```

Во-вторых, для каждого типа карт создадим свою имплементацию фабрики.

```java
public class StandardCardFactory implements CardFactory {
  public Card newCard(ClientAttributes attributes) {
    ...
    return new StandardCard();
  }
}

public class GoldCardFactory implements CardFactory {
  public Card newCard(ClientAttributes attributes) {
    ...
    return new GoldCard();
  }
}

public class PlatinumCardFactory implements CardFactory {
  public Card newCard(ClientAttributes attributes) {
    ...
    return new PlatinumCard();
  }
}
```

Теперь разные фабрики отвечают за создание разных объектов типа `Card`.
Это дает нам много преимуществ.

1. Каждая фабрика может быть протестирована изолированно от других.
2. При появлении нового типа карты старый код менять не потребуется. Нужно будет лишь создать новую фабрику.
3. Если понадобится изменить логику получения определенного типа карты, достаточно будет затронуть лишь одну фабрику.


## Цепочка обязанностей

Поведенческий шаблон проектирования, который позволяет передавать запросы по цепочке от одного обработчика к другому.
Каждый обработчик решает, может ли он обработать запрос, а также стоит ли передавать запрос дальше по цепочке.

Предположим, что мы автоматизируем работу call-центра. На каждый входящий звонок необходимо ответить.
Можно реализовать это следующим образом:
```java
public class CallCenter {
    public void answerIncomingCall(Call call) {
        if (!firstOperator.isOnCall()) {
            firstOperator.answer(call);
        } else if (!secondOperator.isOnCall()) {
            secondOperator.answer(call);
        } else if (!thirdOperator.isOnCall()) {
            thirdOperator.answer(call);
        } else if (!fourthOperator.isOnCall()) {
            fourthOperator.answer(call);
            // и так далее
        }
    }
}
```
Данный подход весьма громоздкий и не очень гибкий: если потребуется добавить нового оператора call-центра, 
то это можно будет сделать только в классе `CallCenter`, модифицировав его код.

Альтернативный вариант решения задачи - выстроить всех операторов друг за другом в цепочку, а затем направить звонок первому в этой цепочке. 
Тогда первый оператор либо ответит на звонок, либо, если он окажется занят, он передаст звонок далее по цепочке, то есть второму оператору. 
Если второй оператор будет занят, то звонок будет передан третьему и так далее. 
В конце цепочки можно поставить оператора-робота, который попросит звонящего перезвонить позже. 

Реализация call-центра с цепочкой обязанностей:
```java
public class CallCenter {

    private Operator firstOperator;

    public CallCenter() {
        Operator firstOperator = new HumanOperatorImpl("Вася");
        Operator secondOperator = new HumanOperatorImpl("Маша");
        Operator thirdOperator = new HumanOperatorImpl("Петя");
        Operator fourthOperator = new HumanOperatorImpl("Оля");
        Operator robotOperator = new RobotOperatorImpl();

        firstOperator.setNextOperator(secondOperator);
        secondOperator.setNextOperator(thirdOperator);
        thirdOperator.setNextOperator(fourthOperator);
        fourthOperator.setNextOperator(robotOperator);
    }

    public void answerIncomingCall(Call call) {
        firstOperator.answer(call);
    }
}
```

В этом примере, как и в предыдущем, цепочка операторов строится в конструкторе. 
Однако в первом примере она "зашита" в коде, а во втором её можно динамически изменять прямо во время работы программы. 
Например, можно вставить в середину нового оператора. Или изменить очерёдность операторов в цепочке.
Ведь по сути цепочка операторов - это структура данных `Список`.

Реализация операторов:
```java
public interface Operator {
    void answer(Call call);
    Boolean isOnCall();
    void setNextOperator(Operator next);
}

public class HumanOperatorImpl implements Operator {
    private String name;
    private Boolean onCall; // Сейчас оператор говорит по телефону?
    private Operator next; // Следующий оператор в цепочке

    public HumanOperatorImpl(String name) {
        this.name = name;
    }

    @Override
    public boolean isOnCall() {
        return onCall;
    }

    @Override
    public void setNextOperator(Operator next) {
        this.next = next;
    }

    @Override
    public void answer(Call call) {
        if (this.onCall) {
            // Текущий оператор говорит по телефону - переводим звонок на следующего
            next.answer(call);
        }
        System.out.println("Здравствуйте, оператор " + this.name + " слушает"); // Ответ на звонок
    }
}

public class RobotOperatorImpl implements Operator {
    @Override
    public void answer(Call call) {
        System.out.println("Нет свободных операторов, перезвоните позже.");
    }

    @Override
    public Boolean isOnCall() {
        return false;
    }

    @Override
    public void setNextOperator(Operator next) {
        throw new UnsupportedOperationException("Робот - это последний оператор");
    }
}
```

## Стратегия

Стратегия (Strategy) - это шаблон проектирования, который определяет набор алгоритмов, 
инкапсулирует каждый из них и обеспечивает их взаимозаменяемость. 
В зависимости от ситуации мы можем легко заменить один используемый алгоритм другим. 
При этом замена алгоритма происходит независимо от клиента — объекта, 
который использует данный алгоритм.

Иными словами, когда мы используем Стратегию, у нас есть набор алгоритмов: 
алгоритм А, алгоритм Б, алгоритм В и так далее.
И все они взаимозаменяемы, то есть в зависимости от ситуации можно подключать 
алгоритм А, или алгоритм Б, или любой другой из этого набора.
Эти алгоритмы отделены от кода, который их использует (клиента).
Поэтому при изменении (либо добавлении) алгоритма, нет необходимости модифицировать клиентский код - 
он остаётся неизменным, а меняются лишь сами алгоритмы.

Рассмотрим применение данного шаблона на примере реализации поведения автомобиля.
Предположим, у автомобиля пока есть лишь одна функциональная возможность - ехать.

В нашей программе могут быть разные автомобили, но с ними, как это часто бывает, нужно работать 
неким единообразным образом. Поэтому мы создаём интерфейс `Car` и реализуем его:

```java
interface Car {
   void drive();
}

class CarImpl {
   void drive() {...}
}
```
Пока у нас одна реализация `Car`, всё просто. Но со временем появляются автомобили с АБС.
Тогда мы добавляем в интерфейс `Car` новый метод `slowdown` и делим нашу единственную реализацию на 2 класса, 
реализуя в каждом из них свой алгоритм торможения:

```java
class ABSCarImpl {
   void drive() {...}
   void slowdown() {...}
}

class NoABSCarImpl {
   void drive() {...}
   void slowdown() {...}
}
```
С течением времени появляется требование добавить новую функциональность:
поворачивать с использованием гидроусилителя руля, электроусилителя руля или без усилителя.
Причём усилителем могут быть (или не быть) оборудованы любые существующие автомобили - как с АБС, так и без.
Поэтому нам уже нужно 6 реализации интерфейса `Car` (3 варианта усилителя руля * 2 варианта АБС):

```java
class HydraulicSteeringABSCarImpl {
   void drive() {...}
   void slowdown() {...}
   void steer() {...}
}

class ElectricSteeringABSCarImpl {
   void drive() {...}
   void slowdown() {...}
   void steer() {...}
}

class MechanicalSteeringABSCarImpl {
   void drive() {...}
   void slowdown() {...}
   void steer() {...}
}

class HydraulicSteeringNoABSCarImpl {
   void drive() {...}
   void slowdown() {...}
   void steer() {...}
}

class ElectricSteeringNoABSCarImpl {
   void drive() {...}
   void slowdown() {...}
   void steer() {...}
}

class MechanicalSteeringNoABSCarImpl {
   void drive() {...}
   void slowdown() {...}
   void steer() {...}
}
```

Такой подход ведёт к резкому увеличению количества классов, а также к дублированию кода,
ведь методы поворота и торможения у некоторых реализаций оказываются одинаковые.
Разумеется, можно вынести повторяющийся код в отдельные классы, а можно применить шаблон "Стратегия".

Для каждой функциональности автомобиля (тормозить, поворачивать) создадим соответствующий интерфейс:

```java
interface SlowdownStrategy {
   void slowdown();
}

interface SteerStrategy {
   void steer();
}
```

Теперь создадим нужные реализации этих интерфейсов (это те самые "алгоритмы", о которых мы говорили в начале):

```java
class HydraulicSteeringStrategy implements SteeringStrategy {
   void steer() { 
       // Поворот с гидроусилителем 
   }
} 

class ElectricSteeringStrategy implements SteeringStrategy {
   void steer() {
      // Поворот с электроусилителем
   }
}

class MechanicalSteeringStrategy implements SteeringStrategy {
   void steer() {
      // Поворот без усилителя
   }
}

class ABSSlowdownStrategy implements SlowdownStrategy {
   void slowdown() {
      // Торможение с АБС 
   }
} 

class NoABSSlowdownStrategy implements SlowdownStrategy {
   void slowdown() {
      // Торможение без АБС 
   }
}
```

Далее можно создавать объекты-автомобили, комбинируя стратегии любым образом, 
указывая их, к примеру, через конструктор или сеттер.

Благодаря использованию шаблона "Стратегия" мы соблюдаем принцип [OCP](https://ru.wikipedia.org/wiki/%D0%9F%D1%80%D0%B8%D0%BD%D1%86%D0%B8%D0%BF_%D0%BE%D1%82%D0%BA%D1%80%D1%8B%D1%82%D0%BE%D1%81%D1%82%D0%B8/%D0%B7%D0%B0%D0%BA%D1%80%D1%8B%D1%82%D0%BE%D1%81%D1%82%D0%B8) - O из SOLID.
А также избавляемся от необходимости менять клиентский код при добавлении новых стратегий.

"Стратегия" может пригодиться в тех ситуациях, когда программе необходимо использовать различные алгоритмы,
когда нужно менять поведение объектов на стадии выполнения (Runtime),
когда нужна возможность задавать поведение конкретным экземплярам класса.
А обращение к алгоритмам через интерфейс позволяет клиентскому коду ничего не знать о деталях их реализации.   