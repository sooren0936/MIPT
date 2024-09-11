# Дженерики

Дженерики - очень мощная концепция в Java, которая не только позволяет писать более безопасный с точки зрения типов код, но и избежать дублирования.

## Проблема недостаточной полиморфности

Чтобы понять необходимость дженериков, давайте рассмотрим простой пример. Предположим, что мы хотим создать контейнер для хранения объекта, который
печатает свое содержимое в консоль, а также отдает содержимое наружу. Посмотрите на пример кода ниже:

```java
public class Container {
    private final Car value;

    public Container(Car value) {
        this.value = value;
    }

    public boolean isPresent() {
        return value != null;
    }

    public void print() {
        if (isPresent()) {
            System.out.println("Container value is: " + value);
        } else {
            System.out.println("Container is empty");
        }
    }

    public Car getValue() {
        return value;
    }
}
```

Метод `isPresent` возвращает статус того, есть ли какое-то значение в `Container-е`. Метод `print` печатает содержимое в консоль в зависимости от статуса `isPresent`.
А метод `getValue` возвращает само значение `Car`. Все работает хорошо, но что если мы хотим использовать `Container` и для других классов?
Более того, не каждый из них даже может имплементировать интерфейс `Vehicle`. Например, в Container-е мы можем хранить строки, числа, другие кастомные объекты и так далее.
Написание нового класса под каждый отдельный тип данных `Container` (`ContainerCar`, `ContainerString` и так далее) выглядит чрезмерно сложным и ненужным.
Но мы же с вами помним, что все классы в Java наследуются от `Object`. Значит, в качестве `value` мы можем указать `Object`, и это даст нам возможность
использовать один и тот же класс для хранения совершенно разных объектов! Посмотрите на пример кода ниже:

```java
public class Container {
    private final Object value;

    public Container(Object value) {
        this.value = value;
    }

    public boolean isPresent() {
        return value != null;
    }

    public void print() {
        if (isPresent()) {
            System.out.println("Container value is: " + value);
        } else {
            System.out.println("Container is empty");
        }
    }

    public Object getValue() {
        return value;
    }
}
```

Вроде как, мы решили проблему. Правда, тот код, который будет использовать `Container`, сильно усложнится. Посмотрите на пример ниже:

```java
public class Main {
    public static void main(String[] args) {
        Container container = new Container(new Car(...));
        /* некоторая логика работы */
        Car car = (Car) container.getValue();
    }
}
```

Обратите внимание на строчку, где мы получаем значение из `Container`. В качестве типа там хранится `Object`, но нам нужен `Car`, потому что именно его мы положили туда.
Значит, нам приходится делать _down cast_ (приведение предка к потомку).

В то время как _up cast_ (приведение потомка к предку) всегда безопасен, down cast таковым не является. Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        String str = "abc";
        Object obj = (Object) str;
        LocalDate date = (LocalDate) obj;
    }
}
```

Сначала мы присваем строку в переменную `str`. Далее выполняем up cast к `Object` и присваиваем результат в переменную `obj`. 
Поскольку любой класс в Java наследуется от `Object`, эта операция безопасна.
В конце концов, мы пытаемся привести `Object` с помощью каста к типу `LocalDate`.

Класс `LocalDate` не является предком `String`, то есть они никак не связаны. Что же произойдет в этом случае? Давайте запустим программу и посмотрим:

```
Exception in thread "main" java.lang.ClassCastException: class java.lang.String cannot be cast to class java.time.LocalDate (java.lang.String and java.time.LocalDate are in module java.base of loader 'bootstrap')
	at org.example.Main.main(Main.java:10)
```

Мы получили исключение `ClassCastException`. Отсюда можно сделать вывод, что использование `Object` в качестве значение не является безопасным, потому что может привести
к неожиданным ошибкам.

## Внедрение дженериков

Давайте немного перепишем класс `Container`. Посмотрите на код ниже:

```java
public class Container<T> {
    private final T value;

    public Container(Object value) {
        this.value = value;
    }

    public boolean isPresent() {
        return value != null;
    }

    public void print() {
        if (isPresent()) {
            System.out.println("Container value is: " + value);
        } else {
            System.out.println("Container is empty");
        }
    }

    public T getValue() {
        return value;
    }
}
```

Значение `T`, которое мы указали в скобках возле название класса, - это и есть дженерик-параметр.
Он означает, что при создании экземпляра этого класса мы явно указываем, какой тип хранится внутри.
Поскольку он же и будет возращаться в `getValue`, мы избавляем себя от необходимости иметь дело с `Object`.
Посмотрите на пример ниже с использованием `Container<T>`:

```java
public class Main {
    public static void main(String[] args) {
        Container<Car> container = new Container<>(new Car(...));
        /* некоторая логика работы */
        Car car = container.getValue();
    }
}
```

Теперь `container.getValue()` сразу возвращает тип `Car`.

> Возможно, вы обратили внимание, что мы создали `Container` с помощью `new Container<>`, а не `new Container<Car>`,
> хотя присваиваем значение именно в переменную типа `Container<Car>`. Начиная с Java 8 при создании объекта
> через `new` не обязательно указывать значения дженерик-параметров, если вы уже указали их в типе переменной.

Дженерик обладают еще одним важным свойством. Если какой-то метод принимает на вход `Container<Car>`, 
то мы не сможем туда передать переменную `Container<Airplane>`, `Container<String>` или любой другой тип, отличный от `Container<Car>`.
Это значит, что многие ошибки можно будет проверить на этапе компиляции программы, а не во время ее выполнения.

> Дженерики в Java - большая и сложная тема. Мы рассмотрели лишь основы, но также есть очень много нюансов, которые выходят за рамки курса.
> Но если вам интересно чуть глубже погрузиться в эту тему, [ознакомьтесь с этой статьей](https://dev.to/kirekov/java-generics-advanced-cases-3iah).

## Обертки вокруг примитивов

Возможно, вы уже обратили внимание, что помимо примитивов вроде `int`, `double` и `boolean` есть некие классы `Integer`, `Double` и `Boolean`.
Это иммутабельные классы-обертки вокруг соответствующих примитивов.

> Если вам непонятно значение слова "иммутабельный", вернитесь к параграфу про инкапсуляцию в ООП.

Они нужны для того, чтобы использовать их в дженериках. Потому что код вроде `Container<int>` не скомпилируется, а вот `Container<Integer>` - да.
Также интересен механизм _autoboxing_, который построен на этих обертках в Java. Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        int value = getInt();
        printInt(value);
    }
  
    private static Integer getInt() {
        return 10;
    }
  
    private static void printInt(Integer value) {
        System.out.println(value);
    }
}
```

На первый взгляд вообще не понятно, как этот код компилируется. Посмотрите на метод `getInt`. Возвращаемое значение - `Integer`, а в `return` мы указываем примитив `int`.
Причем `Integer` - это класс, а `int` - примитив. Как это может работать?

Дело в том, что Java знает об этих обертках и этом, каким примитивам они соответствуют. Поэтому выполняет преобразования от одного типа к другому автоматически.
Чтобы было понятно, посмотрите на альтернативный вариант кода ниже. Он работает точно так же, но теперь все преобразования видны явно:

```java
public class Main {
    public static void main(String[] args) {
        Integer valueObj = getInt();
        int value = valueObj.intValue();
        printInt(Integer.valueOf(value));
    }
  
    private static Integer getInt() {
        return Integer.valueOf(10);
    }
  
    private static void printInt(Integer value) {
        System.out.println(value);
    }
}
```

### Проблемы NullPointerException при autoboxing

В момент autoboxing возможны `NullPointerException`. К примитивам нельзя присвоить `null`: будет ошибка компиляции. Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        printInt(null);
    }
  
    private static void printInt(int value) {
        System.out.println(value);
    }
}
```

Это код даже не скомпилируется. С другой стороны, другой пример кода скомпилируется без проблем, но завершится с `NullPointerException`:

```java
public class Main {
    public static void main(String[] args) {
        Integer valueObj = null;
        printInt(valueObj);
    }
  
    private static void printInt(int value) {
        System.out.println(value);
    }
}
```

В момент передачи параметра типа `Integer` в функцию `printInt` произойдет попытка unboxing-а:
преобразование объекта `Integer` к примитиву `int` (по сути у объекта типа `Integer` будет вызван метод `intValue`).
И как раз в этот момент и произойдет `NullPointerException`. Запустите программу, и вы увидите такое сообщение об ошибке:

```
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "java.lang.Integer.intValue()" because "valueObj" is null
	at org.example.Main.main(Main.java:9)
```