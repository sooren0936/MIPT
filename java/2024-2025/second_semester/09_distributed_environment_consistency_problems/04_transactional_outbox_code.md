# Пример transactional outbox в коде

Давайте рассмотрим пример, как можно реализовать Transactional Outbox в коде.

Сначала создадим таблицу `outbox`, где будут храниться записи для отправки в Kafka:

```sql
CREATE TABLE outbox
(
    id   BIGSERIAL PRIMARY KEY,
    data TEXT NOT NULL
)
```

Добавим соответствующую Hibernate Entity и репозиторий:

```java

@Table(name = "outbox")
@Entity
public class OutboxRecord {
    @Id
    @GeneratedValue(strategy = IDENTITY)
    private Long id;

    @NotNull
    private String data;

    // getters, setters, constructors
}

public interface OutboxRepository extends JpaRepository<OutboxRecord, Long> {
}
```

Если рассматривать пример с `RoleService` из начала модуля, то с применением паттерна Transactional
Outbox он будет выглядеть так:

```java

@Service
public class RolesService {
    private final UserRepository userRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    @Transactional
    public void updateRoles(Long userId, Set<Role> roles) {
        User user = userRepository.findById(userId).orElseThrow();
        user.changeRoles(roles);
        userRepository.save(user);
        outboxRepository.save(new OutboxRecord(
            objectMapper.writeValueAsString(
                new UserRolesUpdated(userId, roles)
            )
        ));
    }
}

@EnableScheduling
@Configuration
public class SchedulerConfig {

}

@Service
public class OutboxScheduler {
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final String topic;
    private final OutboxRepository outboxRepository;

    @Transactional
    @Scheduled(fixedDelay = 10000)
    public void processOutbox() {
        List<OutboxRecord> result = outboxRepository.findAll();
        for (OutboxRecord outboxRecord : result) {
            CompletableFuture<SendResult<String, String>> sendResult = kafkaTemplate.send(topic, outboxRecord.getData());
            // block on sendResult until finished
        }
        outboxRepository.deleteAll(result);
    }
}
```

> По умолчанию планирование задач по расписанию в Spring Boot выключено. Так что явно
> добавляем `SchedulerConfig`.

В `RolesService.updateRoles` мы выполняем бизнес-логику (обновление ролей) и добавляем запись
в `outbox`. Оба изменения коммитятся в рамках одной транзакции.

Метод же `OutboxScheduler.processOutbox` запускается раз в 10 секунд (`fixedDelay = 10000`). Он
выбирает существующие записи `outbox`, для каждой из них отправляет сообщения в Kafka, а потом
удаляет отработанные записи.

> Необязательно каждый раз реализовать паттерн `outbox` самостоятельно.
> Есть [хорошая библиотека](https://github.com/gruelbox/transaction-outbox), у которой также
> присутствует интеграция со Spring.
> Можете применить ее в финальном задании, но в практике этого модуля паттерн Transactional Outbox
> нужно реализовать самостоятельно для лучшего понимания.

## Доменные события

Ранее мы упоминали про доменные события. Если вы используете паттерн богатой доменной модели,
события могут быть удобны для добавления записей в outbox.

Посмотрите на исправленный вариант `User` ниже:

```java

@Entity
@Table(name = "users")
public class User extends AbstractAggregateRoot<User> {
    @Id
    @GeneratedValue(strategy = IDENTITY)
    private Long id;

    // fields, getters, setters

    public void changeRoles(Set<Role> roles) {
        this.setRoles(roles);
        registerEvent(new UserRolesUpdatedEvent(this.id, this.roles));
    }
}

public record UserRolesUpdatedEvent(Long id, Set<Role> roles) {
}
```

Класс [AbstractAggregateRoot](https://www.baeldung.com/spring-data-ddd) предоставляет
метод `registerEvent`. Хочется сразу отметить, что пока никакой публикации события не происходит.
Оно лишь добавляется в `List` в классе `AbstractAggregateRoot`.

Теперь можно отловить событие:

```java

@Service
public class RolesService {
    private final UserRepository userRepository;

    @Transactional
    public void updateRoles(Long userId, Set<Role> roles) {
        User user = userRepository.findById(userId).orElseThrow();
        user.changeRoles(roles);
        userRepository.save(user);
    }
}

@Service
public class UserEventListener {
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    @TransactionalEventListener(phase = BEFORE_COMMIT)
    public void onUserRolesUpdated(UserRolesUpdatedEvent event) {
        outboxRepository.save(new OutboxRecord(
            objectMapper.writeValueAsString(
                new UserRolesUpdated(event.getId(), event.getRoles())
            )
        ));
    }
}
```

При вызове метода `UserRepository.save` произойдет публикация событий, которые мы зарегистрировали в
агрегате `User`. А затем с помощью `UserEventListener.onUserRolesUpdated` мы можем отловить это
событие и выполнить необходимое действие. Таким образом, в `RolesService` отсутствуют
инфраструктурные детали относительно реализации outbox. Мы лишь публикуем
бизнес-событие `UserRolesUpdatedEvent`, а отлавливает его отдельный listener.

> Аннотация `@TransactionalEventListener(phase = BEFORE_COMMIT)` гарантирует, что метод будет вызван
> перед коммитом транзакции.
> Значит, запись в outbox будет добавлена в рамках текущей же транзакции.