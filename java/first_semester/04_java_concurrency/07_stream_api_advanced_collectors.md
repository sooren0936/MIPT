# Сложные коллекторы в Stream API.

На одном из предыдущих занятий мы познакомились с такой терминальной операцией, как коллектор, и
увидели несколько базовых реализаций. На этом уроке рассмотрим более сложные варианты.

## groupingBy

groupingBy - это коллектор, который работает аналогично выражению `GROUP BY` в SQL. Чтобы
использовать его, нам нужно указать, по какому свойству нужно группировать элементы. В результате
элементы стрима будут сгруппированы по этому свойству и помещены в Map.

Предположим, у нас есть список объектов класса `Book`:

```java
class Book {

  private String author;
  private String title;
  private String genre;
  private Integer pageCount;
  // конструктор, геттеры, сеттеры
}

class Main {

  public static void main(String[] args) {
    List<Book> books = List.of(
        new Book("Пушкин", "Евгений Онегин", "Роман", 352),
        new Book("Лермонтов", "Герой нашего времени", "Роман", 224),
        new Book("Гоголь", "Мёртвые души", "Поэма", 560),
        new Book("Гоголь", "Шинель", "Повесть", 64)
    );
  }
}
```

### Простая группировка

Для группировки книг по жанру:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, List<Book>> byGenre = books.stream()
        .collect(Collectors.groupingBy(Book::getGenre));
    // byGenre = {Поэма=['Мёртвые души'], Роман=['Евгений Онегин', 'Герой нашего времени'], Повесть=['Шинель']} 
  }
}
```

Если нужно получить сумму из свойств сгруппированных элементов, то можно воспользоваться такими
методами из класса `Collectors`:

1. `summintInt()`
2. `summingLong()`
3. `summingDouble()`

Например, для получения суммарного количества страниц во всех книгах в определённом жанре можно
написать такой код:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, Integer> totalLengthByGenre =
        books.stream()
            .collect(
                groupingBy(
                    Book::getGenre,
                    Collectors.summingInt(Book::getPageCount)
                )
            );
    // totalLengthByGenre = {Поэма=560, Роман=576, Повесть=64}
  }
}
```

Можно определить среднее количество страниц в книге в определённом жанре:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, Double> averageLength = books.stream()
        .collect(
            groupingBy(
                Book::getGenre,
                Collectors.averagingInt(Book::getPageCount)
            )
        );
    System.out.println("averageLength: " + averageLength);

    // averageLength: {Поэма=560.0, Роман=288.0, Повесть=64.0}
  }
}
```

Можно собрать статистику о количестве страниц в книгах по жанрам:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, IntSummaryStatistics> statisticsByGenre = books.stream()
        .collect(
            groupingBy(
                Book::getGenre,
                Collectors.summarizingInt(Book::getPageCount)
            )
        );
    /*
    statisticsByGenre = {
      Поэма=IntSummaryStatistics{
        count=1, sum=560, min=560, average=560.000000, max=560}, 
      Роман=IntSummaryStatistics{
        count=2, sum=576, min=224, average=288.000000, max=352}, 
      Повесть=IntSummaryStatistics{
        count=1, sum=64, min=64, average=64.000000, max=64}
    }
    */
  }
}


```

Можно найти самую большую книгу у каждого автора:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, Optional<Book>> maxLengthByAuthor = books.stream()
        .collect(
            groupingBy(
                Book::getAuthor,
                Collectors.maxBy(Comparator.comparingInt(Book::getPageCount))
            )
        );

    // maxLengthByAuthor = {Пушкин=Optional['Евгений Онегин'], Гоголь=Optional['Мёртвые души'], Лермонтов=Optional['Герой нашего времени']}
  }
}
```

Обратите внимание, что в результате мы получаем Map, состоящий из `Optional`. Дело в том,
что `maxBy` и `minBy` рассчитаны на работу с пустыми коллекциями. Соответственно, максимума (или
минимума) в пустой коллекции не будет. Отсюда и получается `Optional`.

## mapping

Сгруппированные элементы можно преобразовать, применив дополнительный коллектор `mapping`. К
примеру, если нужно получить все жанры, в которых работал автор:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, Set<String>> genresByAuthor = books.stream()
        .collect(
            groupingBy(
                Book::getAuthor,
                Collectors.mapping(Book::getGenre, toSet())
            )
        );

    // genresByAuthor = {Пушкин=[Роман], Гоголь=[Поэма, Повесть], Лермонтов=[Роман]}
  }
}
```

Первый параметр `mapping` - это функция преобразования элемента стрима, которая работает
аналогично `Stream#map`. А второй параметр - это коллектор.

## filtering

Сгруппированные элементы можно отфильтровать, применив дополнительный коллектор `filtering`.
Например, чтобы получить короткие (количество страниц < 100) книги по каждому автору:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, Set<Book>> shortBooks = books.stream()
        .collect(
            groupingBy(
                Book::getAuthor,
                Collectors.filtering(b -> b.getPageCount() < 100, Collectors.toSet())
            )
        );

    // shortBooks = {Пушкин=[], Гоголь=['Шинель'], Лермонтов=[]}
  }
}
```

Может возникнуть вопрос: почему вместо этого способа нельзя просто отфильтровать элементы в стриме
через метод `filter`, а затем вызвать groupingBy? Дело в том, что в таком случае до этапа
группировки дойдут не все элементы. Поэтому если у какого-либо автора не окажется коротких
произведений, то информация по этому автору не попадёт в результирующую Map. На наших данных
результат был бы таким:

```java
// shortBooks = {Гоголь=['Шинель']}
```

## flatMapping

Допустим, в класс `Book` мы добавили поле `comments` для хранения комментариев:

```java
class Book {

  private List<String> comments = new ArrayList<>();
  /* другие поля */
}
```

И у нескольких книг они заполнены:

```
onegin.setComments(List.of("Отлично!","Хорошо"));
shinel.setComments(List.of("Нормально","Хорошо","Отлично!"));
```

Чтобы получить все комментарии по всем книгам автора, можно использовать такой код:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, List<List<String>>> collect =
        books.stream()
            .collect(
                Collectors.groupingBy(
                    Book::getAuthor,
                    Collectors.mapping(
                        Book::getComments,
                        Collectors.stoList()
                    )
                )
            );
  }
}
```

Но в этом случае получается список списков (`List<List<String>>`), что не всегда удобно. Если нам
нужно получить все комментарии одним списком, то нужно использовать метод `flatMapping`, который по
сути объединяет несколько стримов в один:

```java
public class Main {

  public static void main(String[] args) {
    Map<String, List<String>> authorComments = books.stream()
        .collect(
            Collectors.groupingBy(
                Book::getAuthor,
                Collectors.flatMapping(
                    book -> book.getComments().stream(), Collectors.toList()
                )
            )
        );

    // authorComments = {Пушкин=[Отлично!, Хорошо], Гоголь=[Нормально, Хорошо, Отлично!], Лермонтов=[]}
  }
}
```

# Пользовательские коллекторы в Stream API

В Stream API реализовано множество коллекторов (Collectors): toList, toSet, toMap и т.д. Однако
порой требуется что-то необычное, и штатных коллекторов уже не хватает.

Для таких случаев Java даёт возможность написать свой коллектор.

Предположим, что у нас есть стрим статей `Stream<Article>`, а в статьях есть метод `getWordCount`,
который возвращает количество слов в этой статье. И нам нужен такой коллектор, который просуммирует
количество слов во всех этих статьях. Давайте создадим его.

## Устройство коллектора

Коллектор - это класс, реализующий интерфейс Collector, поэтому для создания своего коллектора мы
можем либо имплементировать данный интерфейс стандартным образом, либо воспользоваться
методом `Collector.of`, который облегчает этот процесс. Второй способ короче - воспользуемся им.

Основные методы интерфейса `Collector`, которые нам нужны — следующие:

### Supplier<A> supplier()

Создаёт объект, в котором будет храниться результат работы коллектора (контейнер):

```
() -> new MutableInteger(0)
 ```

В методе `supplier` необходимо создать контейнер для результата, который в ходе работы стрима будет
передаваться в качестве аргумента в метод `accumulator`. В методе `accumulator` будет обновляться
значение в контейнере. Обычный Integer в качестве контейнера не подойдёт, т.к. в таком случае при
выходе из метода `accumulator` исходное значение контейнера не изменится.

Создадим класс-контейнер:

  ```java
public class MutableInteger {

  private int value;

  public MutableInteger(int value) {
    this.value = value;
  }

  public void add(int value) {
    this.value += value;
  }

  public int getValue() {
    return this.value;
  }
}
```

### BiConsumer<A, T> accumulator()

Возвращает функцию, которая будет использоваться для обновления результата в контейнере на основе
элемента стрима. В нашем случае — добавит количество слов в статье:

```
(accumulator,article) -> accumulator.add(article.getWordCount())
```

### BinaryOperator<A> combiner()

Возвращает функцию, которая будет использоваться при объединении нескольких контейнеров с
результатами:

```
(left,right) -> {
    left.add(right.getValue());
    return left;
}
 ```

Остановимся отдельно на последнем пункте. При последовательной обработке элементов стрима
методы `supplier` и `accumulator` отработают нормально. Но для того чтобы поддерживать параллельную
работу стрима, нам нужна функция объединения нескольких результатов. Дело в том, что в случае
параллельной работы стрим делится на части и каждая часть обрабатывается параллельно. А в конце все
полученные результаты объединяются в один при помощи функции объединения, которая описывается в
методе `combiner`.

## Реализация

```java
public class Main {

  public static void main(String[] args) {
    Collector<Article, Integer, Integer> collector = Collector.of(
        () -> new MutableInteger(0),
        (accumulator, article) -> accumulator.add(article.getWordCount()),
        (left, right) -> {
          left.add(right.getValue());
          return left;
        }
    );
  }
}
```

Обратите внимание: `combiner` вызывается только для параллельных стримов, поэтому если заранее
известно, что со стримом будет вестись только последовательная работа, то данный метод можно не
реализовывать.

Если в конце работы коллектора нам нужно выполнить какие-либо преобразования над контейнером с
результатом, то для этого есть четвёртый метод в интерфейсе `Collector` — `finisher`.

В нашем случае нам необходимо превратить `MutableInteger` в `int`. Обновим код коллектора:

```java
public class Main {

  public static void main(String[] args) {
    Collector<Article, Integer, Integer> collector = Collector.of(
        () -> new MutableInteger(0),
        (accumulator, article) -> accumulator.add(article.getWordCount()),
        (left, right) -> {
          left.add(right.getValue());
          return left;
        },
        accumulator -> accumulator.getValue()
    );
  }
}
```

## Оптимизация

При создании коллектора можно указать его характеристики, которые будут использоваться для
внутренней оптимизации работы коллектора. Эти характеристики можно задать в пятом методе
интерфейса `Collector` — `characteristics`. А в `Collectors.of` эта информация передаётся
через `varargs`:

```java
public class Main {

  public static void main(String[] args) {
    Collector.of(
        // supplier,
        // accumulator,
        // combiner,
        // finisher, 
        Collector.Characteristics.CONCURRENT,
        Collector.Characteristics.IDENTITY_FINISH,
        // ...
    );
  }
}
```

Существует 3 характеристики:

- `CONCURRENT` — Указывает, что контейнер с результатом может использоваться при параллельной работе.
- `UNORDERED` — Указывает, результат работы коллектора не учитывает исходный порядок элементов.
- `IDENTITY_FINISH` — Указывает, что функция `finisher` возвращает контейнер с результатом без
  каких-либо преобразований, следовательно, её можно не вызывать. 

