# Логирование

Давайте наконец-то разберемся, что такое логирование в Java и зачем оно нужно. Логирование - это
фиксирование информации о жизненном цикле приложения во время его работы. Проще говоря, мы сообщаем,
какие действия сейчас выполняются: приняли запрос, пошли в базу данных, создали заказ, получили
ошибку и так далее. Такая информация очень помогает в обнаружении багов. Ведь если есть ошибка, но
нет контекста относительно того, когда и почему она возникла, исправить ее будет сложно. Так что
логирование - очень важная часть промышленной разработки, которую нельзя игнорировать.

Рассмотрим самый примитивный вариант - запись сообщений в консоль с помощью `println`. Ранее мы уже
видели такие примеры в курсе. Почему же мы не можем продолжать это использовать как есть? Дело в
том, что `println` выражения не полиморфны. Предположим, мы написали сервис и везде расставили запись в
консоль подобным образом. Через какое-то время у нас могут возникнуть следующие потребности:

1. Мы хотим писать сообщения не только в консоль, но и файл. Причем файлы должны автоматически
   дробиться по дням и удаляться, если прошел срок более 30 дней.
2. Нужно поменять формат логов: к каждому сообщению мы хотим добавить название потока, в котором запрос
   выполнялся.
3. Требуется отключить часть сообщений, а другую - оставить.
4. Вы написали сервис на Spark Framework и хотите, чтобы часть логов оттуда выводилась, а часть -
   нет.

Как вы понимаете, реализовать все эти пункты с помощью обычного `println` будет затруднительно.
Теоретически, первые три проблемы мы можем одолеть самостоятельно. Объявим такой интерфейс:

```java
public interface LoggingFacility {
  /* методы для логирования */
}
```

И будем инжектить нужную реализацию в тот или иной класс нашего приложения. Но вот последний пункт
так решить не получится. Ведь мы не имеем влияния на код сторонней библиотеки: мы его просто
используем. Так что обычными средствами настроить логирование для нее не получится.

К счастью, нет нужды разрабатывать инфраструктуру логирования самим. Воспользуемся уже готовым
решением, которое существует в экосистеме Java.

## Архитектура логирования в Java

В Java есть большое количество библиотек по логированию. Они обладают немного разным API и набором
фичей. Чтобы унифицировать их, а также дать возможность прозрачно добавлять логи в ваше приложение,
даже если в фреймворке (например, в Spark Framework) используется иная библиотека, в Java-мире
разработали универсальный адаптер [SLF4J (Simple Logging for Java Applications)](https://www.slf4j.org/).

Хотя здесь и есть слово `logging` в названии, сама по себе она ничего не логирует. Это набор
интерфейсов, которые предоставляют лишь API. При этом конкретный провайдер (то есть та или
иная библиотека для логирования) должны их реализовывать на своей стороне. По сути, здесь мы видим
принцип инверсии зависимостей в действии. Есть абстракция (SLF4J), от которой мы зависим, а есть
реализации, которые нас не заботят.

В Java мире есть три самых популярных библиотеки для логирования:

1. [Logback](https://logback.qos.ch/).
2. [Log4j2](https://logging.apache.org/log4j/2.x/).
3. [Java Logging](https://docs.oracle.com/javase/8/docs/api/java/util/logging/package-summary.html) (библиотека встроена в стандартный набор классов Java).

> У всех трех решений есть интеграция с SLF4J.

## Настройка Logback. Уровни логирования

Мы остановимся на Logback. Сначала нужно добавить зависимость на `slf4j-api`:

```xml

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>2.0.5</version>
</dependency>
```

Теперь можно добавить конкретную библиотеку. Пример с Logback выглядит так:

```xml

<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.4.8</version>
</dependency>
```

> Пакет `logback-classic` - это реализация специально для SLF4J. Если вам нужен Logback без интеграции с SLF4J,
> то добавляем `logback-core`.

Осталось настроить Logback, чтобы видеть логи. Мы выберем простой вариант, когда логи отправляются в
консоль. Добавьте файл `src/main/resources/logback.xml` со следующим контентом:

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

> Больше примеров конфигурации `logback.xml` можете посмотреть в [этой статье](https://www.baeldung.com/logback).

Тег `<appender>` обозначает выходной поток для логов, а также их формат. Здесь мы направляем вывод в
консоль. Блок же `<root>` фиксирует:

1. Список appender'ов для вывода (их может быть несколько).
2. Уровень для корневого лога.

Рассмотрим последний пункт подробнее. У каждого `Logger` есть название, и, по конвенциям, оно должно
быть равно полному имени класса с пакетом. Например, `org.example.Main`. Чаще всего в коде `Logger`
создаются так:

```java
package org.example;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Main {

  private static final Logger LOG = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
    LOG.info("App started");
  }
}
```

В функцию `getLogger` мы передали класс, так что библиотека автоматически распознает его имя
как `org.example.Main`. Тем не менее, название можно указать и явно с помощью строки:

```java
package org.example;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Main {

  private static final Logger LOG = LoggerFactory.getLogger("org.example.Main");

  public static void main(String[] args) {
    LOG.info("App started");
  }
}
```

> Мы рекомендуем все же передавать класс при создании логера. Пример со строкой здесь нужен только для демонстрации.

Давайте разберемся с конфигом `<root level="info">`. Уровни логирования идут в следующем порядке:

1. `TRACE`
2. `DEBUG`
3. `INFO`
4. `WARN`
5. `ERROR`

Каждому уровню соответствует метод в классе `Logger` (`trace`, `debug`, `error` и так далее). Если
уровень установлен на `INFO`, то мы увидим все сообщения этого уровня и все, что находится **ниже**.
Уровни выше будут проигнорированы. Проведем эксперимент. Настройте конфигурацию Logback в
соответствии со структурой выше и добавьте такой код:

```java
package org.example;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Main {

  private static final Logger LOG = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
    LOG.debug("This is a debug statement");
  }
}
```

Запустите его: в консоли нет сообщения `This is a debug statement`. Потому что `root.level = INFO`,
а `DEBUG` находится выше. Теперь замените `LOG.debug` на `LOG.info` и запустите код повторно.
Сообщение в консоли появилось. Оно также отобразится, если вы напишите `LOG.warn` или `LOG.error`,
потому что эти уровни находятся ниже `INFO`.

Эта функциональность позволяет нам разделять события в логах на категории и вызывать соответствующий
метод у `Logger`. По принятым соглашениям назначение уровней распределяется так:

1. `TRACE` - подробная информация о каждом шаге, которое выполняет приложение. Обычно такие логи
   включают только в случае крайней необходимости, потому что они слишком подробные, а избыток
   информации может запутать.
2. `DEBUG` - похоже на `TRACE`, но предоставляет чуть менее подробные данные. Как правило, его включают
   в тех ситуациях, когда в приложении есть баг и нужно его отловить.
3. `INFO` - стандартные сообщения о штатном работе приложения. Зачастую в production выставляют
   именно такой уровень.
4. `WARN` - предупреждения о возможном некорректном поведении, которое не является ошибкой, но может
   потребовать обратить на себя внимание. Например, сторонний сервис постоянно отвечает `4xx`
   ошибкой при HTTP-запросе. Или клиент постоянно посылает данные в теле запроса, которые наш сервис
   считаем невалидным и не может распарсить.
5. `ERROR` - ошибки в работе приложения. В основном сюда логируются все `5xx` ошибки, которые мы
   возвращаем клиенту.

## Гибкая конфигурация уровней логирования

Хотя мы и можем выставить `root.level` в нужный уровень, это приведет к тому, что во всем приложении мы
увидим логи соответствующего уровня. Но зачастую это не то, чего мы хотим добиться. Что если мы
хотим выставить уровень в `DEBUG` лишь в конкретном классе или пакете, а в других оставить все без
изменений? Logback и с этой задачей справляется.

Рассмотрим пример. У нас есть класс `Main` и `Calculator`. Оба находятся в пакете `org.example`. При
этом для `Calculator` мы хотим выставить уровень логирования `TRACE`, а для `Main` оставить `INFO` 
(тот, что указан в `root.level`). Посмотрите на пример кода ниже:

```java
public class Main {

  private static final Logger LOG = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
    LOG.debug("App has started");
    System.out.println("Result: " + Calculator.sum(2, 3));
    LOG.debug("App has finished");
  }
}

public class Calculator {

  private static final Logger LOG = LoggerFactory.getLogger(Calculator.class);

  public static int sum(int a, int b) {
    LOG.debug("Preparing calculation");
    int res = a + b;
    LOG.debug("Calculation done");
    return res;
  }
}
```

Чтобы настроить `DEBUG` для класса `Calculator`, внесем изменения в `logback.xml`. Добавим новый
тег `<logger>`:

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

  <logger name="org.example.Calculator" level="DEBUG"/>

</configuration>
```

Все просто: указываем название логера и уровень. Запустите приложение, и вы увидите в консоли
следующие строчки:

```
15:09:28.976 [main] DEBUG org.example.Calculator - Preparing calculation
15:09:28.976 [main] DEBUG org.example.Calculator - Calculation done
Result: 5
```

Уровни `DEBUG` для класса `Calculator` были выведены, а для `Main` - проигнорированы.

> Строчка `Result: 5` будет выводиться всегда, потому что ее мы печатаем с помощью `println`.

Но главная фишка тега `<logger>` не в этом. Нам необязательно указывать название класса. Если мы
укажем только пакет, то у всех классов, которые в нем находятся, будет выставлен соответствующий
уровень. Откройте `logback.xml` и поменяйте `logger.name` на `org.example`. Запустите приложение и
увидите такой результат:

```
15:14:41.147 [main] DEBUG org.example.Main - App has started
15:14:41.149 [main] DEBUG org.example.Calculator - Preparing calculation
15:14:41.150 [main] DEBUG org.example.Calculator - Calculation done
Result: 5
15:14:41.157 [main] DEBUG org.example.Main - App has finished
```

Теперь мы видим результат логирования не только класса `Calculator`, но и `Main`, потому что они оба
находятся в одном `package`. Логика здесь вложенная: если указать в качестве `name`
значение `my.package`, то уровень выставится не только для тех классов, которые находятся непосредственно
внутри `my.package`, но также для вложений любого уровня (`my.package`, `my.package.other`, 
`my.package.other.once.another` и так далее). Это дает нам возможность выставить нужный уровень
логирования для любой библиотеки/фреймворка, которую мы используем. Например, можно указать `TRACE`
для `com.spark`, и вы увидите в консоли огромное количество сообщений от Spark Framework.

## Параметризованные логи

Обычно нам неинтересно логировать статические строки вида `Operation has completed`. Мы хотим знать
контекст: id запроса, DTO, которое получили на вход, и так далее. С SLF4J выполнить это несложно.
Посмотрите на пример кода ниже:

```java
public class Main {

  private static final Logger LOG = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
    LOG.debug("A = {}; B = {}; Calculation result: {}", 2, 3, Calculator.sum(2, 3));
  }
}
```

Вместо параметра используется плейсхолдер `{}`. Потом они указываются через запятую (количество
может быть любым). После выполнения в консоли вы увидите следующее:

```
15:21:11.408 [main] DEBUG org.example.Main - A = 2; B = 3; Calculation result: 5
```

Если вы передаете объект, то у него будет вызван `toString` (очень похоже на работу
метода `String.format`).

Отдельно хочется поговорить про логирование исключений. Посмотрите на пример кода ниже:

```java
public class Main {

  private static final Logger LOG = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
    try {
      doThing1();
    } catch (IllegalArgumentException e) {
      LOG.error("Main has failed", e);
    }
  }

  private static void doThing1() {
    try {
      doThing2();
    } catch (IllegalArgumentException e) {
      throw new IllegalArgumentException("Thing 1 has failed", e);
    }
  }

  private static void doThing2() {
    throw new IllegalArgumentException("Thing 2 has failed");
  }
}
```

> Это искусственный пример для демонстрации логирования исключений при их перехвате.

Функция `main` вызывает `doThing1`, а та в свою очередь вызывает `doThing2`. Оба метода бросают
исключения, но `doThing1` указывает `cause` при пробросе своего. Помните, что на важности этого мы акцентировали
внимание? Запустите код и посмотрите на результат в консоли. Вы увидите следующее:

```
15:25:25.246 [main] ERROR org.example.Main - Main has failed
java.lang.IllegalArgumentException: Thing 1 has failed
	at org.example.Main.doThing1(Main.java:24)
	at org.example.Main.main(Main.java:12)
Caused by: java.lang.IllegalArgumentException: Thing 2 has failed
	at org.example.Main.doThing2(Main.java:29)
	at org.example.Main.doThing1(Main.java:21)
	... 1 common frames omitted
```

Помимо сообщения об ошибке, мы еще получили подробный стектрейс с указанием конкретных строчек, где
исключение было поймано и перехвачено. Если бы мы не указали `cause` при пробросе исключения
в `doThing1`, то увидели бы только последнее исключение, но не изначальное. Так что помните о
важности этого момента.

Что если мы хотим и залогировать сообщение с дополнительными параметрами, и указать исключение,
чтобы увидеть стектрейс? Все просто: исключение всегда нужно указывать последним. Посмотрите на
пример ниже:

```java
public class Main {

  private static final Logger LOG = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
    int param = 458;
    try {
      doThing1(param);
    } catch (IllegalArgumentException e) {
      LOG.error("Main has failed with param = {}", param, e);
    }
  }

  private static void doThing1(int param) {
    try {
      doThing2(param);
    } catch (IllegalArgumentException e) {
      throw new IllegalArgumentException("Thing 1 has failed", e);
    }
  }

  private static void doThing2(int param) {
    throw new IllegalArgumentException("Thing 2 has failed");
  }
}
```

При запуске вы увидите сообщение:

```
15:30:12.032 [main] ERROR org.example.Main - Main has failed with param = 458
java.lang.IllegalArgumentException: Thing 1 has failed
	at org.example.Main.doThing1(Main.java:23)
	at org.example.Main.main(Main.java:13)
Caused by: java.lang.IllegalArgumentException: Thing 2 has failed
	at org.example.Main.doThing2(Main.java:28)
	at org.example.Main.doThing1(Main.java:21)
	... 1 common frames omitted
```

Заметьте, что для исключения мы **не** указываем плейсхолдер `{}`. Указываем только для параметров.

## Антипаттерн - конкатенация при логировании

Иногда может возникнуть соблазн использовать конкатенацию или `String.format` вместо
плейсхолдеров `{}`. Посмотрите на пример ниже:

```java
public class Main {

  private static final Logger LOG = LoggerFactory.getLogger(Main.class);

  public static void main(String[] args) {
    LOG.debug(
        String.format(
            "A = %s; B = %s; Calculation result: %s",
            2,
            3,
            Calculator.sum(2, 3)
        )
    );
  }
}
```

Если запустите код, то в консоли увидите тот же результат, что и раньше:

```
15:33:11.319 [main] DEBUG org.example.Main - A = 2; B = 3; Calculation result: 5
```

Хотя визуально разницы нет, дьявол кроется в деталях. В случае использования плейсхолдеров строка
статична и создается один раз. Более того, если в коде некоторые константные строки повторяются,
Java создает один объект и переиспользует его.

> Подробнее об этой оптимизации можете почитать в статье про [Java String Pool](https://www.baeldung.com/java-string-pool).

Если же вы конкатенируете строки или вызываете `String.format`, то **каждый раз**, когда вызов
доходит до строчки с логом, Java вынуждена создать новую строку, потому что она составлена
динамически. Если в вашем приложении много логирующих выражений, это может повлиять на
производительность. Так что обращайте внимание, чтобы в логах у вас не использовалась конкатенация
или `String.format`.