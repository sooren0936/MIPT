### Абстрактный пример по проектированию таблицы для аудита

Данные примеры являются как иллюстрацией, для того чтобы вы могли примерно понимать что от вас требуется.

- **Partition Key**: Должен равномерно распределять данные по узлам. В случае аудита пользователей, хорошим выбором
  может быть `user_id`.
- **Clustering Key**: Используется для сортировки данных внутри партиции. В случае аудита это может быть `event_time`.

```sql
CREATE TABLE my_keyspace.user_audit (
    user_id UUID,               -- Идентификатор пользователя (данные будут распределены по юзерам)
    event_time TIMESTAMP,       -- Время события (используется для уникальности так и для сортировки)
    event_type TEXT,            -- Тип события (например, "INSERT", "DELETE")
    event_details TEXT,         -- Детали события
    PRIMARY KEY ((user_id), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC)
   AND default_time_to_live = 2592000;

```

---

#### Пример вставки данных

```java
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.cql.PreparedStatement;
import com.datastax.oss.driver.api.core.cql.BoundStatement;

@Service
public class UserAuditService {

    @Autowired
    private CqlSession session;

      enum Action {
        SELECT, UPDATE, INSERT, DELETE, DROPPED_DATABASE
      }

    public void insertUserAction() {
        // Подготовка запроса
        PreparedStatement preparedStatement = session.prepare(
                "INSERT INTO my_keyspace.user_audit (user_id, event_time, event_type, event_details) " +
                        "VALUES (?, ?, ?, ?)"
        );

        // Создание BoundStatement с параметрами
        BoundStatement boundStatement = preparedStatement.bind(
                java.util.UUID.fromString("123e4567-e89b-12d3-a456-426614174000"), // user_id
                java.time.Instant.now(),                                           // event_time
                Action.DROPPED_DATABASE.toString(),                                // event_type
                "User DROPPED DATABASE from IP 192.168.1.1"                        // event_details
        );

        // Выполнение запроса
        session.execute(boundStatement);
    }
}
```
