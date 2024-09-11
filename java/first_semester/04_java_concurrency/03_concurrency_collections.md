# Synchronized collections

Перейдем к потокобезопасным инструментам для работы с коллекциями. Начнем с синхронизированных
коллекций. Удобно, что API этих коллекций остается таким же, как и у обычных.

Помимо обычных непотокобезопасных классов из `java.util`: `HashMap`, `ArrayList`, `Hashset`, есть
еще коллекции, которые можно получить с помощью методов:

- `Collections.synchronizedCollection(Collection<T> c)`
- `Collections.synchronizedList(List<T> l)`
- `Collections.synchronizedMap(Map<K, V> m)`
- `Collections.synchronizedSet(Set<T> s)`

Эти методы возвращают синхронизированную wrapper над вашей коллекцией, методы которой
обернуты `synchronized`. Вот, например, несколько методов класса `SynchronizedCollection`:

```java
public class SynchronizedCollection {

  SynchronizedCollection(Collection<E> c) {
    this.c = Objects.requireNonNull(c);
    mutex = this;
  }

  public boolean add(E e) {
    synchronized (mutex) {
      return c.add(e);
    }
  }

  public boolean remove(Object o) {
    synchronized (mutex) {
      return c.remove(o);
    }
  }

  public boolean containsAll(Collection<?> coll) {
    synchronized (mutex) {
      return c.containsAll(coll);
    }
  }

  public boolean addAll(Collection<? extends E> coll) {
    synchronized (mutex) {
      return c.addAll(coll);
    }
  }
}
```

Здесь синхронизация происходит на один и тот же объект `mutex`, поэтому при конкурентном обращении
два потока не смогут одновременно выполнять разные методы одного экземпляра коллекции:
один из них будет ждать, пока другой закончит работать.

## Итерация по коллекции

Давайте представим, что мы в одном потоке итерируемся по коллекции, а в соседнем потоке произошло
некоторое изменение той же коллекции, что будет? Ответ зависит от того, какой мы используем
итератор, есть как минимум два вида:

- **fail-fast iterator** - при конкурентном изменении коллекции во время итерирования этот итератор
  бросает исключение `ConcurrentModificationException`.
- **fail-safe iterator** - не бросает исключение при конкурентном изменении коллекции во время
  итерирования.

В синхронизированной коллекции используется первый тип итератора. То есть нам остается только
надеяться, что никакой другой поток не изменит коллекцию, пока мы с ней работаем. Но можно самим
обернуть всю итерацию в блок `synchronized`:

```java
class SynchronizedExample {

  public void someMethod() {
    List<Integer> synchronizedList = Collections.synchronizedList(new ArrayList<>());
    synchronizedList.addAll(List.of(1, 2, 3, 4, 5));

    synchronized (synchronizedList) {
      for (Integer element : synchronizedList) {
        //some logic
      }
    }
  }

}
```

Тогда заблокируется вся коллекция до тех пор, пока по ней не отработает итерация. Это решение может
повлиять на производительность, ведь коллекции могут быть большими.

## Производительность

Подход, используемый в синхронизированных коллекциях, делает их потокобезопасными, но крайне
медленными, так как в одно время с коллекцией может взаимодействовать только один поток. Другие же
потоки ждут своей очереди на исполнение.

# Конкурентные коллекции

Конкурентные коллекции, содержащиеся в пакете `java.util.concurrent`, предоставляют лучшую
производительность, чем синхронизированные коллекции. Давайте посмотрим, какими механизмами это
достигается.

## Устройство коллекций

### ConcurrentHashMap

Эта коллекция не ограничивает число потоков, одновременно читающих данные. Для записи происходит
блокировка данных, но блокируется не вся коллекция целиком. Дело в том, что `ConcurrentHashMap`
разделена на сегменты (по умолчанию их 16), и если нужно записать данные в какой-то один сегмент, то
и блокируется один сегмент.

Такой подход позволяет увеличить количество и одновременно читающих, и одновременно пишущих потоков.

`ConcurrentHashMap` использует итератор, который не бросает
исключение `ConcurrentModificationException` при конкурентном изменении коллекции. Этот итератор
использует состояние коллекции на момент начала итерации и даже может подхватывать новое конкурентно
измененное состояние, но такое поведение не гарантируется.

### CopyOnWriteArrayList, CopyOnWriteArraySet

Две данные коллекции являются иммутабельными, то есть после создания их невозможно поменять, и чтобы
модифицировать коллекцию, нужно создать новый экземпляр и только потом записать новые данные.

Именно из-за иммутабельности нет необходимости синхронизировать разные версии коллекций и не нужно
производить дополнительные блокировки для чтения и итерации. Итератор этих коллекций отображает
состояние на момент начала итерации и не знает ничего о новых записях.

`CopyOnWriteArrayList, CopyOnWriteArraySet` выгодно использовать только в случаях, когда чтений
коллекции значительно больше, чем ее изменений, потому что копирование всех данных для их
последующего изменения - очень дорогая операция как для памяти, так и вычислительно.

## Сравнение с Синхронизированными коллекциями

|                                                    |                                Synchronized                                |                            Concurrent                             |
|----------------------------------------------------|:--------------------------------------------------------------------------:|:-----------------------------------------------------------------:|
| Блокировка                                         |                         Блокируется вся коллекция                          | Либо блокируется часть коллекции, либо блокировка не используется |
| Итерация                                           | Требует остановки всех соседних потоков, иначе есть вероятность исключения |                  Можно итерироваться конкурентно                  |
| Возможность ограничить доступ только на один поток |                                     да                                     |                                нет                                |

Итак, конкурентные коллекции в целом производительнее, чем синхронные, и зачастую стоит выбирать
именно их. Но синхронизированные коллекции предоставляют более строгие гарантии: ни чтение, ни
итератор не могут вывести "неактуальные" данные. Если вам критично использовать самое свежее
состояние коллекции, можно выбрать синхронизированные, но при этом помнить об их влиянии на
производительность.
