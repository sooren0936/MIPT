# Не возвращайте null

Ранее в курсе мы упоминали `null` как концепцию отсутствующего значения.
Это знание может дать соблазн использовать его там, где не следует. Например, посмотрите на пример кода ниже.
Это представление некоторого хранилища, в котором мы ищем `Customer` по `id`.

```java
public class CustomerRepository {
    /* поля и конструктор */
  
    public Customer findById(CustomerId id) {
        if (!existsById(id)) {
            return null;
        }
        /* логика поиска Customer по id */
    }
}
```

Если сущность `Customer` отсутствует по `id`, то мы возращаем `null`. Иначе - запускаем иную логику его поиска.

Кажется, что все правильно. Но `null` может сыграть с нами злую шутку. Посмотрите на пример кода ниже:

```java
public class Main {

  public static void main(String[] args) {
    CustomerRepository customerRepository = createCustomerRepository();
    Customer customer = customerRepository.findById(new CustomerId(1));
    System.out.println(customer.getSalary());
  }

  private static CustomerRepository createCustomerRepository() {
    /* логика создания CustomerRepository */
  }
}
```

Если `CustomerRepository` вернет `null`, то вызов `customer.getSalary()` приведет к `NullPointerException`.
Это очень известная проблема в Java. Чтобы избежать ее, необходимо **не** возвращать `null` в публичных методах.
Что же делать в тех ситуациях, когда значение отсутствует (мы не смогли найти `Customer` по переданному `CustomerId`)?
Есть несколько вариантов:

**Кидайте exception**

Самый простой и очевидный способ. Даже если вы его не отловите, то по крайней мере в информации об исключении будет понятное сообщение об ошибке.
`NullPointerException` же обычно не несет в себе никаких полезных сведений.

**Используйте паттерн Null Object**

Он означает, что вместо `null` мы возвращаем такой экземпляр `Customer` (или его наследника), который олицетворяет то, что значение отсутствует.
Например, можно вернуть экземпляр, где все поля `String` заполнены пустыми строками.

Такой способ не всегда подходит, но он гарантированно избавит вас от нежелательных `NullPointerException`.

**Оберните значение в Optional**

Класс `Optional` - это контейнер, который может содержать, а может не содержать значение определенного типа.
Идет в стандаратной библиотеке Java, так что нужно просто его использовать как есть. Посмотрите на пример кода ниже:

```java
public class CustomerRepository {
    /* поля и конструктор */
  
    public Optional<Customer> findById(CustomerId id) {
        if (!existsById(id)) {
            return Optional.empty();
        }
        /* логика поиска Customer по id */
        return Optional.ofNullable(customer);
    }
}

public class Main {

  public static void main(String[] args) {
      CustomerRepository customerRepository = createCustomerRepository();
      Optional<Customer> customer = customerRepository.findById(new CustomerId(1));
      if (customer.isPresent()) {
          Customer value = customer.get();
          System.out.println(customer.getSalary());
      }
  }

  private static CustomerRepository createCustomerRepository() {
    /* логика создания CustomerRepository */
  }
}
```

Строго говоря, класс `Optional` является монадой, что в свою очередь относится к функциональному программированию.
Но если вам хочется узнать об этом подробнее, можете ознакомиться с [этой статьей](https://habr.com/ru/companies/otus/articles/668794/).

> Класс `Optional` использует дженерики, так что метод `Optional.get()` сразу возвращает нужный тип.

# Пара слов о Lombok

В Java, как вы заметили, много [boilerplate'а](https://ru.wikipedia.org/wiki/%D0%A8%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD%D0%BD%D1%8B%D0%B9_%D0%BA%D0%BE%D0%B4).
Проблема решается с помощью [Lombok](https://projectlombok.org/). Это библиотека, которая позволяет генерировать инфраструктурный код и не писать его явно.
Вот неполный список того, что может сгенерировать Lombok:

1. Геттеры
2. Сеттеры
3. With-методы для создания копии объекта с измененным полем
4. Конструкторы
5. Методы `toString/equals/hashCode`

По нашему мнению, начинающим Java-разработчикам лучше не использовать Lombok, потому что он скрывает детали, которые могут быть не очевидны.
Но использовать его мы не запрещаем. Так что если вам интересно,
можете посмотреть [инструкцию установки для Maven](https://projectlombok.org/setup/maven) и попробовать его в действии.