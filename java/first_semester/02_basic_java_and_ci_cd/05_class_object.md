# Класс Object

Наверняка вы обратили внимание, что при попытке обращения к методу какого-либо класса Idea подсказывает те,
которые мы не добавляли: `equals`, `hashCode`, `toString` и так далее. Дело в том, что все классы в Java являются наследниками базового класса `Object`.
Если вы не писали `extends` в объявлении класса, то Java подставляет `extends Object` автоматически.

> Это значит, что если метод принимает `Object` в качестве параметра, вы можете передавать туда экземпляр абсолютно любого класса.

Класс `Object` предоставляет ряд методов, которые повсеместно используются в Java. Но мы обсудим лишь наиболее важные из них:

1. `equals`
2. `hashCode`
3. `toString`

## Метод equals

Сигнатура метода `equals` у `Object` выглядит следующим образом.

```java
class Object {
    /* other methods... */
  
    public boolean equals(Object obj) {
        return (this == obj);
    }
}
```

Он служит для сравнения двух объектов между собой. Например, если `airplane1.equals(airplane2)`, то `airplane1` равен `airplane2`.
Но по умолчанию метод сравнивает не содержимое объектов, а их ссылки: это то, что делает оператор `==`.
Если `this` и `obj` указывают на один и тот же объект в heap, то возвращается `true`. Иначе - `false`.

Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane1 = new Airplane(1980, "RED");
        Airplane airplane2 = airplane1;
        System.out.println(airplane1.equals(airplane2));
    }
}
```

В консоль выведется `true`, потому что переменные `airplane1` и `airplane2` указывают на один и тот же объект в heap.

Посмотрите на еще один пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane1 = new Airplane(1980, "RED");
        Airplane airplane2 = new Airplane(1980, "RED");
        System.out.println(airplane1.equals(airplane2));
    }
}
```

Здесь результат уже будет `false`. Потому что переменные `airplane1` и `airplane2` указывают на два разных объекта в heap.

Принцип понятен, но такое поведение не выглядит логичным. В конце концов, если у нас есть два объекта `Airplane` с одинаковым
набором полей, то метод `equals` должен возвращать `true`, ведь в этом его суть. К счастью, это можно исправить.
Как вы уже догадались, на помощь снова приходит наследование и полиморфизм. Поскольку метод `equals` определен
в дочернем классе `Object`, мы можем переопределить его в `Airplane`. Посмотрите на пример кода ниже.

```java
public class Airplane {
  /* поля и конструктор */ 
  
  @Override
  public boolean equals(Object o) {
    if (o instanceof Airplane airplane) {
      return year == airplane.year && Objects.equals(color, airplane.color);
    }
    return false;
  }
}
```

Поскольку метод `equals` принимает в качестве параметра `Object`, то объект может не быть экземпляром класса `Airplane`.

> Помните про то, что любой объект в Java наследует Object. Следовательно, `Airplane` точно является `Object`,
> но не каждый `Object` - `Airplane`.

Оператор `instanceof` служит для проверки того, является ли переменная экземпляром или наследником
указанного класса (в нашем случае, `Airplane`). Если условие выполняется, то в переменной `airplane`
будет представлен объект, приведенный к классу `Airplane`.

Теперь нужно реализовать нашу кастомную логику `equals`. Нас интересует сравнение полей `year` и `color`.
Поле `year` мы сравниваем через оператор `==`, потому что `year` - примитив. Поскольку их значения копируются при передаче,
то и сравниваются именно они, а не ссылки.

Для проверки равенства `color` мы используем утилитный метод из класса `Objects`. Его реализация представлена ниже:

```java
public class Objects {
    public static boolean equals(Object a, Object b) {
        return (a == b) || (a != null && a.equals(b));
    }
}
```

Сначала два объекта сравниваются на ссылочное равенство через `==`. Далее, если объект `a` не равен `null`,
то мы просто делегируем проверку на `equals` уже ему. Поскольку в классе `String` метод `equals` уже реализован правильно,
нам не нужно об этом беспокоиться.

> Мы обсудим концепцию `null` далее в курсе. Для текущего понимания - это специальное значение, которое отображает "ничто".
> Может быть присвоено любому объекту.

Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane1 = new Airplane(1980, "TU-154");
        Airplane airplane2 = new Airplane(1980, "TU-154");
        System.out.println(airplane1.equals(airplane2));
    }
}
```

Если вы все сделали правильно, то увидите `true` в консоли.

## Метод hashCode

Метод `hashCode` неразрывно связан с `equals`. Для соблюдения корректного контракта реализации, их **всегда** нужно переопределять вместе.

Контракт можно описать следующими пунктами:

1. Если объекты равны по `equals`, их `hashCode'ы` должны быть равны.
2. Если объекты НЕ равны по `equals`, их `hashCode'ы` могут как отличаться, так и быть равны.
3. Если объекты равны по `hashCode'у`, они могут быть как равны, так и не равны по `equals`.

Если вы поищите метод `hashCode` в классе `Object`, то найдете такую строчку:

```java
public class Object {
    public native int hashCode();
}
```

Это значит, что реализации `hashCode` спрятана внутри Java Virtual Machine, а не в виде обычного Java-кода.
Реализации зависит от JDK, которую вы используете. Один из вариантов - это указатель на адрес, где аллоцирован объект, преобразованный в `int`.

Чтобы посчитать `hashCode` для `Airplane`, мы воспользуемся решением, схожим с подсчетом `equals`.
То есть посчитаем `hashCode` для каждого поля и преобразуем значения в единый результат.
Посмотрите на пример кода ниже:

```java
public class Airplane {
  /* поля и конструктор */
  
    @Override
    public int hashCode() {
        return Objects.hash(year, color);
    }
}
```

Реализацию метода `Objects.hash` можете посмотреть в Idea подробнее. Суть же его в следующем:

1. Считаем `hashCode` для параметра, вызывая соответствующий метод на нем (если это `int`, то `hashCode` равен этому же числу), и умножаем на множитель.
2. Повторяем до тех пор, пока поля не закончатся.

Суть `hashCode-а` в том, что если у двух объектов набор полей с одинаковыми значениями, то и `hashCode-ы` у них будут совпадать.

Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane1 = new Airplane(1980, "TU-154");
        Airplane airplane2 = new Airplane(1980, "TU-154");
        System.out.println("HashCode для Airplane1: " + airplane1.hashCode());
        System.out.println("HashCode для Airplane2: " + airplane2.hashCode());
    }
}
```

Если вы все сделали правильно, то значения `hashCode-ов` для этих двух объектов будут совпадать:

```
HashCode для Airplane1: -1810167607
HashCode для Airplane2: -1810167607
```

> И `equals`, и `hashCode` можно переопределить с помощью Idea: нет необходимости писать их руками.
> Для этого нажмите на класс правой кнопкой мыши и выберите пункт: `Generate -> equals/hashCode`

## HashMap

Класс `HashMap` является каноничным примером необходимости переопределения `equals` и `hashCode`.
Про алгоритм работы этой структуры данных [можете почитать здесь](https://habr.com/ru/articles/421179/).

Идея структуры данных в том, чтобы хранить ассоциативные массивы. Иначе говоря, это отношения _ключ-значение_, где и ключ, и значение, могут быть любого типа.
Посмотрите на пример кода ниже.

```java
import java.util.HashMap;
import java.util.Map;

public class Main {
    public static void main(String[] args) {
        Airplane airplane1 = new Airplane(1980, "TU-154");
        Airplane airplane2 = new Airplane(1990, "Boeing");
        Map<Airplane, Integer> map = new HashMap<>();
        map.put(airplane1, 2);
        map.put(airplane2, 3);
  }
}
```

> Конструкция <Airplane, Integer> - это реализация механизма generic-типов в Java.
> Мы обсудим это подробнее позже. Пока можете считать, что это способ типизации: в map можно положить ключ лишь типа `Airplane` и получить значение `Integer`.

> Класс HashMap реализует интерфейс Map. Так что мы можем присвоить HashMap в Map.

В данном примере мы добавили в `Map` два отношения: `airplane1 -> 2` и `airplane -> 3`. Также мы можем искать значения по ключам с помощью метода `get`.
Интерес в том, что средняя сложность поиска по ключу в `HashMap` - `O(1)`. Достигается это за счет правильной реализации `equals` и `hashCode` (подробности читайте по ссылке выше).

Посмотрите на пример кода с поиском ключа в `HashMap` ниже.

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane1 = new Airplane(1980, "TU-154");
        Airplane airplane2 = new Airplane(1990, "Boeing");
        Map<Airplane, Integer> map = new HashMap<>();
        map.put(airplane1, 2);
        map.put(airplane2, 3);

        System.out.println(map.get(new Airplane(1980, "TU-154")));
        System.out.println(map.get(new Airplane(1990, "Boeing")));
  }
}
```

В качестве значений в консоли мы получим `2` и `3`. Несмотря на то, что в качестве параметра метода `get` мы передали новые объекты `Airplane`,
поля у них те же, что и у ключей на момент вставки. А значит, что корректное переопределение `equals` и `hashCode`
позволяет найти нам те значения, которые мы ожидаем получить.

## Метод toString

Смысл метода `toString` полностью соответствует его названию: приведение содержимого объекта к строке. Посмотрите на реализацию `toString` в классе `Object`:

```java
public class Object {
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }
}
```

По умолчанию вызов `toString` вернет конкатенацию из значений:

1. Название класса
2. Символ `@`
3. Значение `hashCode`, приведенное к строке в шестнадцатеричном виде.

В большинстве ситуаций полученное значение не будет информативным. Давайте переопределим метод `toString` в классе `Car`. Посмотрите на пример кода ниже:

```java
public class Car implements Vehicle {
    /* Поля и конструктор */
  
    @Override
    public String toString() {
        return String.format("Car[brand=%s, year=%d, color=%s]", this.brand, this.year, this.color);
    }
}
```

Мы возвращаем информацию о полях `Car` в виде строки.

> Метод String.format имеет такой же принцип работы, как и [функция `printf` в языке C](https://www.tutorialspoint.com/c_standard_library/c_function_printf.htm).
> То есть позволяет в удобном виде сконкатенировать несколько значений в одну строку. 

У функции `toString` есть еще одно важное свойство: Java вызывает ее автоматически, когда вы конкатенируете объект со строкой. Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        Car car = new Car("Toyota", 2009, "WHITE");

        System.out.println("Машина: " + car.toString());
        System.out.println("Машина: " + car);
    }
}
```

Первый и второй вывод в консоль дадут одинаковые результаты. Во втором случае Java автоматически вызовет метод `toString` у `Car`, чтоб сконкатенировать результат
со строкой `Машина: `.

Переопределение `toString` является хорошим тоном. Но мы хотим, чтобы вы поняли важный момент. Метод `toString` **не** должен использоваться для выполнения специфических бизнес-операций.
Другими словами, `toString` нужен, чтобы выводить техническую информацию об объекте в логи, но не для того, чтобы передавать ее клиенту напрямую.
Если вы хотите специфическим образом превратить объект в строку, лучше добавьте кастомный метод.
Метод же `toString` применяйте в тех ситуациях, когда вас интересует лишь техническая информация об объекте.

> Сгенерировать `toString` можно также с помощью Idea. 