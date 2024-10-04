# Исключения

В любой программе возникают моменты, когда нам нужно обрабатывать исключительные ситуации.
Вернемся к классу `Airplane`. Допустим, мы не хотим, чтобы поле `brand` приобретало значение `null`: в этом случае нужно прервать создание экземпляра класса `Airplane`.
Как нам добиться этого эффекта? На помощь нам придут исключения.

Все исключения в Java - это обычные классы. То есть мы можем создавать экземпляры исключений, хранить в них поля и так далее.
Также в Java есть класс `Throwable`. Любой класс, который наследуется от `Throwable`, считается исключением.

> В Idea вы можете нажать клавишу `shift` два раза и вписать название класса, которое нужно найти.
> Среда разработки откроет вам его декларацию, чтобы вы могли с ней ознакомиться.

Тем не менее, наследоваться напрямую от `Throwable` считается плохим тоном.
В языке Java уже есть наследники от `Throwable`, которые мы будем применять:

1. `Exception` - проверяемое исключение, наследуется от `Throwable`.
2. `RuntimeException` - непроверяемое исключение, наследуется от `Exception`.
3. `Error` - ошибка, которая означает серьезную проблему в программе.
   Например, переполнения стека вызовов (`StackOverflowError`), или недостаток памяти (`OutOfMemoryError`).
   Наследуется от `Throwable`.

Разницу между проверяемыми и непроверяемыми исключениями мы обсудим позже. Пока давайте остановимся на `RuntimeException`.

## Выбрасывание и обработка исключений

Мы начали абзац с того, что мы не хотим допускать ситуации, когда поле `brand` у класса `Airplane` равно `null`.
Посмотрите на пример кода ниже:

```java
public class Airplane implements Vehicle {
    /* поля и конструктор */

    public Airplane(int year, String color) {
        if (color == null) {
            throw new RuntimeException("Color cannot be null");
        }
        this.year = year;
        this.color = color;
    }
}
```

Если мы пытаемся создать экземпляр `Airplane` с полем `color`, равным `null`, то выбрасывается `RuntimeException`.
Давайте попробуем это сделать в `main` и посмотрим, что получится:

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane = new Airplane(1980, null);
        airplane.print();
    }
}
```

При запуске вы увидите такое сообщение в консоли:

```
Exception in thread "main" java.lang.RuntimeException: Color cannot be null
	at org.example.Airplane.<init>(Airplane.java:12)
	at org.example.Main.main(Main.java:9)
```

Программа завершилась с ошибкой на той точке, где мы попытались создать `Airplane` с неожиданным значением поля `brand`.

Такое поведение логично, но зачастую не является желанным. В конце концов, если программа будет завершаться аварийно при каждой недопустимой ситуации,
ценность ее под вопросом. Поэтому помимо создания исключительных ситуаций, в Java еще есть механизм их _обработки_.
Посмотрите на пример кода ниже, где мы "ловим" исключение:

```java
public class Main {
    public static void main(String[] args) {
        try {
            Airplane airplane = new Airplane(1980, null);
            airplane.print();
        } catch (RuntimeException e) {
            System.out.println("Ошибка при создании Airplane");
        }
    }
}
```

Конструкция `try-catch` позволяет обрабатывать определенные типы исключений и перенаправлять поток выполнения программы.
В данном случае, если создание экземпляра `Airplane` происходит без ошибок, то код успешно доходит до строчки `airplane.print()`.
В иной же ситуации срабатывает блок `catch`, и выполнение переходит туда.
Если вы запустите этот фрагмент, то увидите в консоли сообщение: `Ошибка при создании Airplane`.

> Можно добавлять несколько блоков `catch` под разные виды исключений.

Также мы хотим напомнить про механизм наследования в Java. Исключения - это тоже классы. При этом `RuntimeException` наследует `Exception`.
Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        try {
            Airplane airplane = new Airplane(1980, null);
            airplane.print();
        } catch (Exception e) {
            System.out.println("Ошибка при создании Airplane");
        }
    }
}
```

Здесь в блоке `catch` мы ловим `Exception` вместо `RuntimeException`. Но поскольку `RuntimeException` наследуется от `Exception`,
этот фрагмент кода работает точно так же, как и предыдущий. Отсюда следует вывод, что можно ловить не только сам класс исключения, но и его предков.

Исключения можно использовать не только в конструкторах, но и в любых методах Java. Посмотрите на пример кода ниже.
В нем мы не даем создать новую копию `Airplane` через `withYear`, если переданное значение меньше нуля:

```java
public class Airplane implements Vehicle {
    /* поля и конструктор */
  
    public Airplane withYear(int newYear) {
        if (newYear < 0) {
            throw new RuntimeException("Year cannot be less than zero: " + newYear);
        }
        return new Airplane(newYear, this.color);
    }
}
```

## Проверяемые исключения

Класс `Exception` олицетворяет проверяемые исключения. Они работают точно так же, как и `RuntimeException`,
но с той разницей, что для выбрасывания `Exception` из метода нам нужно явно прописать его в сигнатуре.
Посмотрите на пример кода ниже с методом `Airplane.withYear`.

```java
public class Airplane implements Vehicle {
    /* поля и конструктор */
  
    public Airplane withYear(int newYear) {
        if (newYear < 0) {
            throw new Exception("Year cannot be less than zero: " + newYear);
        }
        return new Airplane(newYear, this.color);
    }
}
```

Удивительно, но этот фрагмент не компилируется. Чтобы выбрасывать проверяемые исключения, мы должны явно сообщить об этом в сигнатуре метода.
Посмотрите на исправленный вариант ниже:

```java
public class Airplane implements Vehicle {
    /* поля и конструктор */
  
    public Airplane withYear(int newYear) throws Exception {
        if (newYear < 0) {
            throw new Exception("Year cannot be less than zero: " + newYear);
        }
        return new Airplane(newYear, this.color);
    }
}
```

Рядом с названием метода мы добавили `throws Exception`. Может показаться, что эта деталь незначительна.
Однако давайте попробуем вызвать `Airplane.withYear` в функции `main`. Посмотрите на пример кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane = new Airplane(1980, "RED");
        Airplane newAirplane = airplane.withYear(2000);
        newAirplane.print();
    }
}
```

Сюрприз, но это фрагмент кода тоже не компилируется! Дело в том, что когда мы обращаемся к методу, у которого в сигнатуре есть `throws` с проверяемым исключением,
то компилятор обязывает нас выполнить одно из следующих действий:

1. Отловить исключение с помощью `catch`.
2. Пробросить его дальше, указав уже в текущем методе `throws ...`

Посмотрите на исправленный вариант кода ниже:

```java
public class Main {
    public static void main(String[] args) {
        Airplane airplane = new Airplane(1980, "RED");
        try {
            Airplane newAirplane = airplane.withYear(2000);
            newAirplane.print();
        } catch (Exception e) {
            System.out.println("Ошибка при создании нового Airplane");
        }
    }
}
```

Проще говоря, проверяемые исключения принуждают нас либо обработать их на месте, либо пробросить дальше, чтобы их отловил кто-то, кто вызвал нашу функцию.

> Хотя идея с проверяемыми исключениями выглядит привлекательно на первый взгляд, в современной разработке они практически не используются.
> Причины этого станут понятны в следующем семестре, когда мы начнем изучение Spring Framework.
> Но если вам интересен этот вопрос, можете ознакомиться со [следующей статьей](https://phauer.com/2015/checked-exceptions-are-evil/).

## Кастомные типы исключений

Выбрасывать и отлавливать `RuntimeException` или `Exception` считается не очень хорошим подходом. Вместо этого лучше бросать наследника, который
покажет конкретный тип ошибки. Если вы откроете декларацию класса `RuntimeException` в Idea и нажмете на стрелку слева от названия класса,
то увидите список наследников: они предоставляются стандартной библиотекой Java. Конкретно в нашем случае стоит использовать `IllegalArgumentException`
вместо `RuntimeException`. Также вы можете создать свой класс, отнаследовать его от `RuntimeException` и бросать/отлавливать его как собственный тип исключения.

Отлавливать `Exception` или `RuntimeException` вместо конкретного типа исключения плохо по той причине,
что вы можете "поймать" не только ту ошибку, на которую рассчитывали, но и другую, которая возникла без вашего ведома.
Обычно это означает, что в коде есть какая-то исключительная ситуация, которую вы не обработали. Поэтому мы рекомендуем вам избегать таких подходов.

> Помните про `null`? Если вы присвоили это значение в переменную какого-то типа, а затем пытаетесь обратиться к методу, то Java
> выбрасывает `NullPointerException` - наследника `RuntimeException`.

## Тип Error

У класса `Throwable` два прямых наследника: `Exception` и `Error`. С первым мы разобрались, а второй требует дополнительных разъяснений.
С точки зрения кода `Error` ведет себя как обычный `RuntimeException`. Но у этого типа есть один нюанс. Наследники `Error` характеризуют
ошибки, которые вы как программист не сможете обработать в коде. Например, переполнения стека вызовов (`StackOverflowError`) или невозможность
создать новый объект в heap, потому что не хватает памяти (`OutOfMemoryError`). Иначе говоря, это ошибки не прикладного кода, а самой JVM.
Подход следующий: если возникает `Error`, мы не отлавливаем его, а просто даем программе упасть. Звучит контринтуитивно, но в этом есть логика.
Например, какие будут ваши действия в случае `OutOfMemoryError`? Java напрямую вам говорит: "У меня закончилась память, чтобы создавать новые объекты".
Поскольку Java - это язык со сборщиком мусора, где нельзя вручную (как в C/C++) выделять и освобождать память, такая ситуация окажется для вас патовой.
Решения никакого нет, следовательно, лучше дать программе упасть, чтобы потом разобраться с причинами.

> Нюанс еще и в том, что спецификация Java не гарантирует дальнейшей корректной работы программы при возникновении `Error`.
> Поэтому отлавливать такие исключения не имеет смысла.

Здесь нужно сделать еще один очень важный вывод. **Никогда** не отлавливайте `Throwable` в блоке `try-catch`. Потому что `Throwable` является предком
для класса `Error`. Следовательно, в `catch` попадут ошибки, которые не должны быть обработаны.

## StackTrace

Еще одно важное понятие исключений в Java - StackTrace. Это стек вызовов, который показывает, в каких классах и на каких строчках кода произошла ошибка.
Давай посмотрим еще раз на пример с выбрасыванием исключения в конструкторе `Airplane` и последующим аварийным заверешением программы в `main`.

```java
public class Airplane implements Vehicle {
    /* поля и конструктор */

    public Airplane(int year, String color) {
        if (color == null) {
            throw new IllegalArgumentException("Color cannot be null");
        }
        this.year = year;
        this.color = color;
    }
}

public class Main {
   public static void main(String[] args) {
      Airplane airplane = new Airplane(1980, null);
      airplane.print();
   }
}
```

В консоли вы увидите следующее сообщение об ошибке:

```
Exception in thread "main" java.lang.RuntimeException: Color cannot be null
	at org.example.Airplane.<init>(Airplane.java:12)
	at org.example.Main.main(Main.java:6)
```

То есть исключение хранит информацию о том, в каком месте оно было выброшено: через какие строчки кода в этом случае оно прошло.
Этот инструмент очень полезен, чтобы находить причины ошибок. Действительно, если бы получили просто сообщение `Color cannot be null`
без каких-либо уточнений о том, где оно случилось, найти источник проблемы было бы очень непросто.

И здесь мы хотим сделать важную ремарку. Если с исключениями работать неправильно, возможна потеря stacktrace'а.
Чтобы понять проблему, посмотрите на пример кода ниже:

```java
public class Airplane implements Vehicle {
    /* поля и конструктор */

    public Airplane(int year, String color) {
        if (color == null) {
            throw new IllegalArgumentException("Color cannot be null");
        }
        this.year = year;
        this.color = color;
    }
}

public class Main {
   public static void main(String[] args) {
     try {
         Airplane airplane = new Airplane(1980, null);
         airplane.print();
     }
     catch (IllegalArgumentException e) {
         throw new IllegalStateException("Couldn't create Airplane");
     }
   }
}
```

В функции `main` мы отлавливаем возможное `IllegalArgumentException` и пробрасываем в этом случае `IllegalStateException` далее по стеку вызовов.
Если вы запустите эту программу, то увидите следующую ошибку в консоли:

```
Exception in thread "main" java.lang.IllegalStateException: Couldn't create Airplane
	at org.example.Main.main(Main.java:11)
```

Информация о функции `main` присутствует. Тем не менее, мы потеряли сведения о классе `Airplane`, который и породил изначальное `IllegalArgumentException`.
Чтобы избежать этой проблемы, достаточно вторым параметром конструктора `IllegalStateException` передать объект того исключения, который мы отловили изначально.
Посмотрите на пример кода ниже:

```java
public class Airplane implements Vehicle {
    /* поля и конструктор */

    public Airplane(int year, String color) {
        if (color == null) {
            throw new IllegalArgumentException("Color cannot be null");
        }
        this.year = year;
        this.color = color;
    }
}

public class Main {
   public static void main(String[] args) {
     try {
         Airplane airplane = new Airplane(1980, null);
         airplane.print();
     }
     catch (IllegalArgumentException e) {
         // в переменной 'e' хранится объект исключения, который был выброшен
         throw new IllegalStateException("Couldn't create Airplane", e); // передаем 'e' вторым параметром
     }
   }
}
```

Теперь вывод в консоли отличается:

```
Exception in thread "main" java.lang.IllegalStateException: Couldn't create Airplane
	at org.example.Main.main(Main.java:11)
Caused by: java.lang.IllegalArgumentException: Color cannot be null
	at org.example.Airplane.<init>(Airplane.java:12)
	at org.example.Main.main(Main.java:7)
```

Сразу видно, что `IllegalStateException` возникло из-за другого исключения - `IllegalArgumentException`. Можно посмотреть класс и метод, где оно было выброшено.

Механизм `cause` в исключениях позволяет выстраивать длинные цепочки причинно-следственных связей ошибок. Не стоит игнорировать их использование,
потому что их наличие позволяет разбираться в проблемах намного проще и быстрее.