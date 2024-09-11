# Asserts

Давайте теперь подробнее посмотрим на главную задачу наших тестов - проверку того, что результат
тот, который мы ожидали. Для этого мы используем функции, называемые ассертами. Эти функции являются
всего лишь инструментом, который просто помогает вам выразить ваши намерения.

К примеру, если у вас вызывается функция assertEquals, это значит, что вы ожидаете, что два
параметра будут равны. Если это не так, значит все сломалось, у вас есть проблема, и ее
надо срочно чинить. Но пока ассерты получают на входе ожидаемое условие, тест будет проходить.

Все доступные ассерты в JUnit являются статическими методами
класса `org.junit.jupiter.api.Assertions`. Основные ассерты:

* `assertEquals` - проверяет, что переданные аргументы равны
* `assertNotEquals` - аналогично проверяет, что не равны
* `assertTrue` - условие истинно
* `assertFalse` - условие ложное
* `assertNull` - переданный аргумент равен null
* `assertNotNull` - не равен null
* `assertThrows` - принимает два параметра. Класс исключения и код для выполнения. Проверяет, что
  код выбросит заданное исключение.

В любой ассерт можно дополнительно передать строковый параметр `message`. Это бывает полезно, если
из кода теста не очевидно, зачем мы вообще хотим это проверить. Сообщение будет добавлено в отчет,
сформированный Junit'ом, если тест упадет, и будет проще разобраться, что же пошло не так.

*Важно!* Если вы используете свои объекты в ассертах, к примеру `assertEquals(course1, course2)`, то
у этого объекта должны быть переопределены методы `equals` и `hashcode`. Иначе объекты будут
сравниваться по ссылкам и тест проверит не то, что вы ожидаете.

Если вам кажется, что ассерты из предыдущих примеров сложно читаемые, то существуют внешние
библиотеки для ассертов. Самые популярные - *AssertJ* и *Hamcrest*.

## AssertJ

AssertJ - это библиотека, которая является попыткой сделать проверку результатов выполнения вашего
кода более читаемой, похожей на обычный английский язык. Давайте посмотрим на простом примере.
Пусть у нас есть тест, который проверяет, что записей в репозитории меньше 10.

```java
class MyTest {
    @Test
    void testRecordCount() {
        List<Course> all = repository.findAll();
        assertTrue(all.size() < 10);
    }
}
```

А вот так будет выглядеть тот же тест с использованием AssertJ.

```java
class MyTest {
    @Test
    void testRecordCountAssertJ() {
        List<Course> all = repository.findAll();
        assertThat(all).hasSizeLessThan(10);
    }
}
```

Это читается намного проще, практически как текст на английском языке.

Также AssertJ очень удобен для сложных проверок. Допустим, вам надо проверить, что среди всех курсов
не существует курсов за авторством конкретного человека. Как это сделать? Получаем все курсы из
репозитория, дальше в цикле достаем всех авторов и проверяем, не равен ли он нашему. С помощью
AssertJ это записывается простым декларативным способом:

```java
class MyTest {
    @Test
    void testAndreiDidNotWriteAnyCourse() {
        List<Course> all = repository.findAll();
        assertThat(all).extracting("author").doesNotContain("Андрей");
    }
}
```

Чем проще читается тест, тем он понятнее, и тем проще будет его чинить в будущем, если что-то пойдет не
так. [Ссылка на документацию](https://joel-costigliola.github.io/assertj/assertj-core-quick-start.html)
.

## Hamcrest

Если стиль AssertJ вам не нравится, то существует еще один инструмент, выполненный в подобном духе,
Hamcrest.

Он выполняет примерно ту же функцию — пытается сделать код тестов более читаемым, просто делает это
иным образом. В отличие от AssertJ, где код пишется как последовательность вызовов
у [builder'a](https://en.wikipedia.org/wiki/Builder_pattern), здесь используется иерархическое
дерево матчеров. Очень страшное название, но на деле это просто означает, что функции проверки иногда
бывают вложены друг в друга.

Те же примеры, но и использованием Hamcrest:

```java
class MyTest {
    @Test
    void testRecordCount() {
        List<Course> all = repository.findAll();
        assertThat(all.size(), is(lessThan(10)));
    }

    @Test
    void testAndreiDidNotWriteAnyCourse() {
        List<Course> all = repository.findAll();
        assertThat(all, is(not(contains(hasProperty("author", is("Андрей"))))));
    }
}
``` 

Результат очень похожий, ассерты выглядят как обычные предложения на английском.
[Ссылка на документацию](http://hamcrest.org/JavaHamcrest/tutorial).