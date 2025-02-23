# Значение null

Стоит упомянуть важный концепт в Java - `null`. Это особое значение, которое может быть присвоено любому объекту (примитивы не могут быть `null`).
Значение `null` используется в Java для того, чтобы записать отсутствующие данные какого-либо типа. Посмотрите на пример класса `Person` ниже.

```java
public class Person {
  private final String firstName;
  private final String lastName;
  private final String patronymic;

  /* конструктор */
}
```

Отчество (поле `patronymic`) у некоторых людей может отсутствовать. Что в этом случае записывать? Один из выходов - использовать `null`. Посмотрите на пример кода ниже.

```java
public class Main {
    public static void main(String[] args) {
        Person person = new Person("Петр", "Иванов", null);
        System.out.println(person);
    }
}
```

Значение `null` не имеет конкретного типа, и его можно передавать в качестве любого объекта. Тем не менее, с использованием `null` связаны определенные проблемы.
Посмотримте на пример кода с классом `Airplane` ниже.

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane = new Airplane(1980, "TU-154");
        airplane = null;
        airplane.print();
    }
}
```

Если вы запустите этот код, то вместо привычного вывода инфорации о самолете в консоль увидите следующее сообщение:

```
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "org.example.Airplane.print()" because "airplane" is null
	at org.example.Main.main(Main.java:8)
```

Ошибка произошла в строчке вызова `airplane.print()`. Дело в том, что `null` не является объектом сам по себе. Это лишь представление
для отсутствующего значения. Следовательно, у `null` не может быть ни метода `print`, ни какого-либо другого.
Поэтому попытка такого обращения приводит к исключению - `NullPointerException`.

> Про исключения и способы их обработки мы поговорим далее.

Если вы используете `null` в своей программе, вам следует иметь это ввиду.
