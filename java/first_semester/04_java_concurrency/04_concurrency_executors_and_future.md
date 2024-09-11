# Управление задачами

При построении программных систем мы так или иначе выстраиваем логику выполнения определенных _задач_. 
Это может быть запрос в базу данных, выполнение сложного алгоритма, отправка уведомления
другому серверу и так далее. Предположим, что мы пишем собственный веб-сервер. Наивная реализация
может выглядеть следующим образом:

```java
class WebServer {

  public static void main(String[] args) throws IOException {
    ServerSocket socket = new ServerSocket(80);
    while (true) {
      Socket connection = socket.accept();
      handleRequest(connection);
    }
  }
}
```

В данном случае все запросы обрабатываются последовательно в однопоточном режиме. Следующая задача
начинает выполняться только после завершения предыдущей. Понятно, что подобная реализация будет не
эффективной. Но что если каждая задача будет выполняться в отдельном потоке?

```java
class WebServer {

  public static void main(String[] args) throws IOException {
    ServerSocket socket = new ServerSocket(80);
    while (true) {
      Socket connection = socket.accept();
      new Thread(() -> handleRequest(connection)).start();
    }
  }
}
```

На каждый новый запрос создается новый поток. Такой вариант действительно даст прирост в
производительности. Однако у бесконтрольного создания потоков есть ряд недостатков.

1. Создание нового потока является дорогой операцией с точки зрения ОС.
2. Активные потоки потребляют значительное количество ресурсов (особенно памяти).
3. Максимальное количество потоков в ОС ограничено. Теоретически при большом количестве запросов
   программа может завершиться аварийно.

Чтобы избежать подобных проблем, используют _пулы потоков_. Это объекты, которые инкапсулируют в
себе определенное количество потоков и предоставляют интерфейс по запуску задач. Но после завершения
задачи поток не удаляется, а остается внутри и может переиспользоваться для следующих запросов.

Начиная с Java 5 предоставляется интерфейс `ExecutorService`, который стандартизирует API для
запуска задач на пуле потоков.

## Future

Когда мы используем `ExecutorService`, важно понимать значение такого объекта, как `Future`.

```java
class FutureExample {

  public void createFuture() {
    Future<Response> future = executorService.submit(() -> sendRequestToRemoteServer());
  }
}
```

`Future` олицетворяет контейнер, значение в котором либо уже присутствует, либо появится со
временем. Чтобы получить результат, достаточно вызвать метод `get`.

```java
public class FutureResponseExample {

  public Response retrieveResponse() {
    Future<Response> future = executorService.submit(() -> sendRequestToRemoteServer());
    /* какие-то другие действия */
    Response response = future.get();
    return response;
  }
}
```

Однако проблема в том, что метод `get` является _блокирующим_. Это значит, что поток будет в
ожидании результата до тех пор, пока он не появится. Поэтому есть вариация `get` с timeout.

```java
public class FutureResponseExample {

  public Response retrieveResponse() throws TimeoutException {
    Future<Response> future = executorService.submit(() -> sendRequestToRemoteServer());
    /* какие-то другие действия */
    Response response = future.get(3, TimeUnit.SECONDS);
    return response;
  }
}
```

В данном случае результат будет ожидаться не более трех секунд. В противном случае
выбросится `TimeoutException`.

## Стандартные реализации ExecutorService

В классе `Executors` также присутствуют несколько имплементаций `ExecutorService`, которые покрывают
большую часть потребностей в контексте многопоточного программирования. Вот некоторые из них:

1. `newFixedThreadPool`
2. `newCachedThreadPool`
3. `newScheduledThreadPool`

### Fixed Thread Pool

Fixed Thread Pool - это наиболее простая в понимании концепция. При вызове `newFixedThreadPool`
создается объект с фиксированным количеством потоков (количество передается в качестве параметра).
При поступлении новой задачи из пула запрашивается свободный поток. Если таковой отсутствует, задача
ставится в очередь.

Эта реализация `ExecutorService` является наиболее популярной и используется чаще всего.

### Cached Thread Pool

Кэшированный пул потоков, в отличие от фиксированного, не ставит верхнюю границу на количество
потоков. Вместо этого используется следующий алгоритм:

1. Если есть свободный поток, задача присваивается ему
2. Если нет, создается новый поток

Эта реализация подходит для тех случаев, когда не ожидается «всплесков» нагрузки. Потому что в
противном случае может быть создано чересчур много потоков, что отрицательно скажется на
производительности.

> Строго говоря, максимальное количество потоков в Cached Thread Pool не является бесконечным.
> Оно равно `Integer.MAX_VALUE`.

### Schedule Thread Pool

Вызов `newScheduledThreadPool` возвращает интерфейс `ScheduledExecutorService`, который является
расширением `ExecutorService`.
`ScheduledExecutorService` позволяет запускать задачи с заданной задержкой и интервалом.

## Выключение ExecutorService

После выполнения необходимых задач `ExecutorService` требуется выключить. Для этого есть следующие методы:

1. `shutdown`
2. `shutdownNow`
3. `awaitTermination`

`shutdown` отправляет запрос на выключение, после которого новые задачи более не принимаются.
Поведение `shutdownNow` похоже, однако также происходит попытка прервать все задачи, которые сейчас
выполняются. А `awaitTermination` ждет указанное количество времени, пока `ExecutorService` не будет
выключен.

# CompletableFuture

## Future

`Future API`, введенная в Java 5, предоставила возможность асинхронного программирования. С помощью
нее можно запускать задачи параллельно в соседних потоках, главный же поток не блокируется на время
выполнения задачи, а только получает уведомление о статусе работы.

`Future` - самая важная сущность API и представляет собой ссылку на результат выполнения асинхронной
задачи. С помощью методов `isDone`, `isCanceled` можно проверить состояние задачи, а с
помощью `get()` получить результат.

```java
public class FutureExample {

  public int calculateInteger() {
    // Какая-то долгая и сложная задача, которая возвращает Future<Integer>
    Future<Integer> future = new Counter.countSomethingBig();

    while (!future.isDone()) {
      System.out.println("Counting...");
      Thread.sleep(300);
    }

    Integer result = future.get();
    return result;
  }
}
```

`Future` - большой шаг в сторону удобства работы с асинхронностью, но у нее есть несколько
недостатков.

- нельзя завершить задачу самостоятельно
- метод `get()` блокирует поток до момента получения результата
- нельзя скомбинировать несколько `Future`
- нельзя выстроить цепочку из фьюч для последовательного выполнения

Поэтому в Java 8 добавили более удобную надстройку над `Future` - `CompletableFuture`.

## CompletableFuture

Новая `CompletableFuture` уже умеет решать все указанные выше проблемы. Давайте разберем, как
работать с ней и как она борется с этими недостатками.

### Создание задачи

#### supplyAsync

Если нужно асинхронно выполнить задачу и вернуть результат, можно использовать есть метод `supplyAsync`:

```java
public class Main {

  public static void main(String[] args) {
    GroundControl groundControl = new GroundControl();
    CompletableFuture<String> completableFuture =
        CompletableFuture
            .supplyAsync(() -> groundControl.getStatus());
  }
}
```

#### runAsync

Если нам нужно выполнить какую-то работу без возврата результата выполнения, можно
воспользоваться `runAsync`:

```java
public class Main {

  public static void main(String[] args) {
    MajorTom majorTom = new MajorTom();
    CompletableFuture<Void> completableFuture =
        CompletableFuture
            .runAsync(() -> MajorTom.sendStatus());
  }
}
```

## Executor

Мы уже знаем, что `supplyAsync` и `runAsync` запускают задачи в новом потоке. Но откуда берется
новый поток?

Ответ: он получается из общего `ForkJoinPool.commonPool()`. Но мы можем использовать и свой пул:

```java
public class Main {

  public static void main(String[] args) {
    Executor executor = Executors.newCachedThreadPool();
    MajorTom majorTom = new MajorTom();

    CompletableFuture<Void> completableFuture =
        CompletableFuture
            .runAsync(() -> MajorTom.sendStatus(), executor);
  }
}
```

## Завершение задачи

Одна из центральных возможностей, которая даже фигурирует в названии - `CompletableFuture`.

```java
public class Main {

  public static void main(String[] args) {
    CompletableFuture<String> completableFuture = new CompletableFuture<>();

    try {
      // заблокирует главный поток на 100 миллисекунд
      completableFuture.get(100, TimeUnit.MILLISECONDS);
    } catch (Exception e) {
      logger.warn("Completing future with default value", e);
    } finally {
      completableFuture.complete("Устал ждать");
    }
  }
}
```

Ура, мы смогли завершить фьючу руками и вернуть результат "Устал ждать".

## Обработка результата

У `CompletableFuture` метод `get()` тоже блокирующий, как и у обычной фьючи. Но зато мы можем
навесить на `CompletableFuture` колбэк, который вызовется, когда вернется результат задачи.

### thenApply

```java
public class Main {

  public static void main(String[] args) {
    Counter counter = new Counter();
    CompletableFuture<String> createBalanceInfo =
        CompletableFuture.supplyAsync(() -> {
          long amountOfMoney = counter.countMoney();
          return amountOfMoney;
        }).thenApply(amountOfMoney -> "There is a lot of money:" + amountOfMoney);
  }
}
```

`thenApply` возвращает `CompletableFuture<T>`, поэтому можно сделать цепочку из `thenApply`, которые
последовательно будут преобразовывать результат.

### thenAccept

`thenAccept`, в отличие от `thenApply`, не возвращает результата выполнения, он просто делает
некоторую полезную работу:

```java
public class Main {

  public static void main(String[] args) {
    Counter counter = new Counter();
    CompletableFuture<Void> createBalanceInfo =
        CompletableFuture.supplyAsync(() -> {
          long amountOfMoney = counter.countMoney();
          return amountOfMoney;
        }).thenAccept(amountOfMoney -> send(amountOfMoney));
  }
}
```

### thenRun

То же, что и `thenAccept`, только не имеет доступа к результату задачи.

```java
public class Main {

  public static void main(String[] args) {
    Counter counter = new Counter();
    CompletableFuture<Void> createBalanceInfo =
        CompletableFuture.supplyAsync(() -> {
          long amountOfMoney = counter.countMoney();
          return amountOfMoney;
        }).thenRun(() -> logger.info("Created balance info"));
  }
}
```

### async методы

`thenApply`, `thenAccept`, `thenRun` выполняются в том же потоке, в котором и
сама `CompletableFuture`, но если мы хотим запустить их в отдельных потоках - мы можем
воспользоваться async методами: `CompletableFuture` предоставляет целых две реализации асинхронных
методов для всех колбеков.

```java
public class CompletableFuture {

  public <U> CompletableFuture<U> thenApply(
      Function<? super T, ? extends U> fn
  ) {
    return uniApplyStage(null, fn);
  }

  public <U> CompletableFuture<U> thenApplyAsync(
      Function<? super T, ? extends U> fn
  ) {
    return uniApplyStage(defaultExecutor(), fn);
  }

  public <U> CompletableFuture<U> thenApplyAsync(
      Function<? super T, ? extends U> fn, Executor executor
  ) {
    return uniApplyStage(screenExecutor(executor), fn);
  }
}
```

Перепишем пример сверху на async:

```java
public class Main {

  public static void main(String[] args) {
    Counter counter = new Counter();
    CompletableFuture<String> createBalanceInfo =
        CompletableFuture.supplyAsync(() -> {
          long amountOfMoney = counter.countMoney();
          return amountOfMoney;
        }).thenApplyAsync(amountOfMoney -> "There is a lot of money:" + amountOfMoney);
  }
}
```

Теперь колбэк будет выполняться в новом потоке, полученном из пула `ForkJoinPool.commonPool()`. Если
мы хотим получить поток из своего пула, это можно сделать, указав его вторым аргументом
в `thenXXXAsync`:

```java
public class Main {

  public static void main(String[] args) {
    Executor executor = Executors.newFixedThreadPool(2);
    Counter counter = new Counter();

    CompletableFuture<String> createBalanceInfo =
        CompletableFuture.supplyAsync(() -> {
          long amountOfMoney = counter.countMoney();
          return amountOfMoney;
        }).thenApplyAsync(amountOfMoney -> "There is a lot of money:" + amountOfMoney, executor);
  }
}
```

## Комбинирование результатов

### Комбинирование двух задач

Допустим, у нас есть две зависимые задачи:

```java
public class ServiceInfo {

  CompletableFuture<ClientInfo> getClientInfo(String clientId) {
    return CompletableFuture.supplyAsync(() -> ClientService.getClientInfo(clientId));
  }

  CompletableFuture<BalanceInfo> getBalanceInfo(ClientInfo clientInfo) {
    return CompletableFuture.supplyAsync(() -> {
      int balanceId = clientInfo.getBalanceId();
      return BalanceService.getBalanceInfo(balanceId);
    });
  }
}
```

Мы хотим, чтобы вторая запускалась с результатами первой, тогда можно использовать метод `thenCompose()`:

```java
public class Main {

  public static void main(String[] args) {
    ServiceInfo serviceInfo = new ServiceInfo();
    CompletableFuture<BalanceInfo> balanceInfo = serviceInfo.getClientInfo(clientId)
        .thenCompose(clientInfo -> serviceInfo.getBalanceInfo(clientInfo));
  }
}
```

Если же у нас есть две независимые задачи, и нам нужно обработать их результат, то можно вызвать
метод `thenCombine()`:

```java
public class Main {

  public static void main(String[] args) {
    CompletableFuture<String> completableFuture =
        CompletableFuture.supplyAsync(() -> "Ground Control to ")
            .thenCombine(CompletableFuture.supplyAsync(() -> "Major Tom"), String::concat);
  }
}
```

### Комбинирование нескольких задач

Теперь нам надо запустить несколько задач параллельно и подождать, пока они все завершатся: будем
использовать `allOf()`. Этот метод возвращает `CompletableFuture<Void>`, а чтобы получить результат,
мы можем или вызвать `get()`, или воспользоваться `CompletableFuture.join()`:

```java
public class Main {

  public static void main(String[] args) {
    CompletableFuture<String> future1 = CompletableFuture
        .supplyAsync(() -> "Ground Control to");
    CompletableFuture<String> future2 = CompletableFuture
        .supplyAsync(() -> "Major Tom");
    CompletableFuture<String> future3 = CompletableFuture
        .supplyAsync(() -> "Take your protein pills and put your helmet on");

    CompletableFuture<Void> completableFuture =
        CompletableFuture.allOf(future1, future2, future3);

    String combined = Stream.of(future1, future2, future3)
        .map(CompletableFuture::join)
        .collect(
            Collectors.joining(" ")
        );
    // Ground Control to Major Tom Take your protein pill and put your helmet on
  }
}
```

Если нужен результат только первой закончившей фьючи, то подойдет метод `anyOf()`. Единственное,
этот метод всегда возвращает `CompletableFuture<Object>`:

```java
public class Main {

  public static void main(String[] args) {
    CompletableFuture<String> future1 = CompletableFuture
        .supplyAsync(() -> "Boris Volynov");
    CompletableFuture<String> future2 = CompletableFuture
        .supplyAsync(() -> "Major Tom");
    CompletableFuture<String> future3 = CompletableFuture
        .supplyAsync(() -> "Valentina Tereshkova");

    CompletableFuture<Object> completableFuture =
        CompletableFuture.anyOf(future1, future2, future3);

    completableFuture.get();
  }
}
```

