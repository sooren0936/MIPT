### Модель данных и CQL в Apache Cassandra

#### Введение

Рассмотрим модель данных Cassandra, включая ключевые концепции, такие как **Keyspace**, **Table**, **Partition Key**, **Clustering Key** и другие. Мы также изучим, как эти концепции применяются на практике, с примерами кода на CQL (Cassandra Query Language).

---

### 1. Keyspace (Пространство ключей)

**Keyspace** — это контейнер, который объединяет таблицы и определяет параметры репликации данных. Его можно сравнить с базой данных в реляционных СУБД. Каждый Keyspace имеет настройки репликации, которые определяют, как данные будут распределяться по узлам кластера.

#### Создание Keyspace

Пример создания Keyspace с использованием CQL:

```sql
CREATE KEYSPACE my_keyspace
WITH replication = {
    'class': 'SimpleStrategy',  -- Стратегия репликации
    'replication_factor': 3     -- Фактор репликации (количество копий данных)
};
```

- **`class`**: Определяет стратегию репликации. Например:
    - `SimpleStrategy` — для одного дата-центра.
    - `NetworkTopologyStrategy` — для нескольких дата-центров.
- **`replication_factor`**: Количество узлов, на которых будут храниться копии данных.

---

### 2. Table (Таблица)

**Table** — это структура, которая хранит данные в виде строк и столбцов. Каждая таблица принадлежит определенному Keyspace и имеет схему, которая определяет типы данных для каждого столбца.

#### Создание таблицы

Пример создания таблицы `users`:

```sql
CREATE TABLE my_keyspace.users (
    user_id UUID PRIMARY KEY,  -- Первичный ключ
    name TEXT,                 -- Текстовое поле
    email TEXT,                -- Текстовое поле
    created_at TIMESTAMP       -- Метка времени
);
```

- **`user_id`**: Первичный ключ (Partition Key), который уникально идентифицирует каждую строку.
- **`name`, `email`, `created_at`**: Столбцы таблицы.

---

### 3. Partition Key (Ключ партиции)

**Partition Key** — это часть первичного ключа, которая определяет, на каком узле кластера будут храниться данные. Cassandra использует хэш-функцию для вычисления значения Partition Key и распределения данных по узлам.

#### Пример Partition Key

```sql
CREATE TABLE my_keyspace.orders (
    user_id UUID,              -- Partition Key
    order_id UUID,             -- Clustering Key
    order_date TIMESTAMP,
    total_amount DECIMAL,
    PRIMARY KEY (user_id, order_id)
);
```

- **`user_id`**: Partition Key. Все заказы одного пользователя будут храниться на одном узле.
- **`order_id`**: Clustering Key (см. ниже).

---

### 4. Clustering Key (Ключ кластеризации)

**Clustering Key** — это часть первичного ключа, которая определяет порядок хранения строк внутри одной партиции. Clustering Key используется для сортировки данных на диске.

#### Пример Clustering Key

```sql
CREATE TABLE my_keyspace.orders (
    user_id UUID,              -- Partition Key
    order_id UUID,             -- Clustering Key
    order_date TIMESTAMP,
    total_amount DECIMAL,
    PRIMARY KEY ((user_id), order_id)
);
```

- **`order_id`**: Clustering Key. Заказы внутри одной партиции (для одного `user_id`) будут отсортированы по `order_id`.

---

### 5. Составной первичный ключ (Composite Primary Key)

Cassandra поддерживает составные первичные ключи, которые состоят из **Partition Key** и одного или нескольких **Clustering Keys**. Это позволяет группировать данные и управлять их сортировкой.

#### Пример составного первичного ключа

```sql
CREATE TABLE my_keyspace.sensor_data (
    sensor_id UUID,            -- Partition Key
    timestamp TIMESTAMP,       -- Clustering Key
    value DOUBLE,
    PRIMARY KEY (sensor_id, timestamp)
);
```

- **`sensor_id`**: Partition Key. Данные одного сенсора хранятся на одном узле.
- **`timestamp`**: Clustering Key. Данные внутри партиции сортируются по времени.

---

### 6. Типы данных в Cassandra

Cassandra поддерживает множество типов данных, включая:

- **Простые типы**: `TEXT`, `INT`, `BIGINT`, `FLOAT`, `DOUBLE`, `BOOLEAN`, `UUID`, `TIMESTAMP`.
- **Коллекции**: `LIST`, `SET`, `MAP`.
- **Пользовательские типы (UDT)**: Позволяют создавать сложные структуры данных.

#### Пример использования коллекций

```sql
CREATE TABLE my_keyspace.user_profiles (
    user_id UUID PRIMARY KEY,
    name TEXT,
    emails SET<TEXT>,          -- Множество email-адресов
    preferences MAP<TEXT, TEXT> -- Карта предпочтений
);
```

---

### 7. Индексы (Secondary Index)

Cassandra поддерживает вторичные индексы для поиска по неключевым столбцам. Однако их использование может снижать производительность, поэтому они рекомендуются только для столбцов с низкой кардинальностью.

#### Пример создания индекса

```sql
CREATE INDEX ON my_keyspace.users (email);
```

---

### 8. Лучшие практики проектирования модели данных

1. **Правильный выбор Partition Key**:
    - Partition Key должен равномерно распределять данные по узлам.
    - Избегайте "горячих" партиций (слишком больших или часто запрашиваемых).

2. **Использование Clustering Key для сортировки**:
    - Используйте Clustering Key для управления порядком данных внутри партиции.

3. **Ограничение использования вторичных индексов**:
    - Вторичные индексы могут снижать производительность. Используйте их только для столбцов с низкой кардинальностью.

4. **Денормализация данных**:
    - Cassandra оптимизирована для записи и чтения больших объемов данных. Денормализация данных может улучшить производительность.

5. **Пакетные операции**:
    - Используйте пакетные операции (`BATCH`) для уменьшения количества запросов.

---

### 9. Примеры запросов

#### Вставка данных

```sql
INSERT INTO my_keyspace.users (user_id, name, email, created_at)
VALUES (uuid(), 'Alice', 'alice@example.com', toTimestamp(now()));
```

#### Выборка данных

```sql
SELECT * FROM my_keyspace.users WHERE user_id = ?;
```

#### Обновление данных

```sql
UPDATE my_keyspace.users
SET email = 'alice.new@example.com'
WHERE user_id = ?;
```

#### Удаление данных

```sql
DELETE FROM my_keyspace.users
WHERE user_id = ?;
```

---


#### 10. Типовая ОШИБКА -  Использование ALLOW FILTERING в SELECT

**Ошибка**: Использование `ALLOW FILTERING` для выполнения запросов без Partition Key.

**Пример**:
```sql
SELECT * FROM my_keyspace.sensor_data ALLOW FILTERING;
```
- **Проблема**: `ALLOW FILTERING` приводит к полному сканированию таблицы, что может быть очень медленным.

**Решение**: Избегайте использования `ALLOW FILTERING`. Вместо этого проектируйте таблицы под конкретные запросы.
```sql

SELECT * FROM my_keyspace.sensor_data WHERE sensor_id = '123e4567-e89b-12d3-a456-426655440000';
```

---

