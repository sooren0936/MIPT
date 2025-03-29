# Новый микросервис

## Настраиваем Kafka

Свяжем наш микросервис, который делали в прошлом уроке.

В качестве брокера сообщения мы будем использовать [Apache Kafka](https://kafka.apache.org/). Далее
в модуле мы
рассмотрим принцип его работы, но пока наша задача –
сделать [MVP (minimum viable product)](https://en.wikipedia.org/wiki/Minimum_viable_product).

Теперь запустим Kafka локально. Проще всего это сделать
через [Docker Compose](https://docs.docker.com/compose/).

> Убедитесь предварительно, что у вас установлен [Docker](https://www.docker.com/).

Теперь добавьте в корень проекта следующий файл с названием `docker-compose.yml`:

```yaml
version: '3.9'

services:

  zookeeper:
    image: confluentinc/cp-zookeeper:7.0.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.0.1
    depends_on:
      - zookeeper
    ports:
      - '29093:29093'
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_LISTENERS: INTERNAL://0.0.0.0:9092,EXTERNAL://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:9092,EXTERNAL://localhost:29093
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_LOG_RETENTION_MS: 100000
      KAFKA_LOG_RETENTION_CHECK_INTERVAL_MS: 5000
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: true
```

Достаточно лишь выполнить команду `docker-compose up` в директории с этим файлом, чтобы запустить
Kafka (если у вас Idea Ultimate, можно запускать `docker-compose` прямо из IDE). Kafka будет
доступна по `localhost:29093`.

> Параметр `KAFKA_LOG_RETENTION_MS` означает, что сообщения старше 100 секунд будут автоматически
> удаляться из Kafka.

Топики при этом будут создаваться автоматически, когда producer попытается отправить сообщение в
Kafka.

> Далее в модуле мы разберем, что такое `топик` в Kafka.
> Сейчас можете считать, что это именованный поток данных в Kafka.
> Например, если producer отправил сообщение в топик `my-topic`, то consumer может прочитать
> сообщение из этого же топика.

## Добавляем зависимости

```xml

<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

> Такая зависимость должны быть в обоих сервисах.

## Построение логики producer-consumer

Сделаем такой вариант:

1. Пользователь вызывает например операцию. `POST` endpoint.
2. Основной сервис produce'ит сообщение в Kafka.
3. Второй сервис consume'ит сообщение и совершает с ним дальнейшие действия по бизнес логике.

Откройте `application.properties` и пропишите там параметры:

```properties
spring.kafka.bootstrap-servers=localhost:29093
topic-to-send-message=your-topic-name
```

Тут мы указываем адрес, на котором запущена Kafka. Также заполняем топик, в который будем отправлять
сообщение.

Аналогично сделайте для второго сервиса, но там нам понадобится еще несколько
параметров:

```properties
spring.kafka.bootstrap-servers=localhost:29093
spring.kafka.consumer.group-id=your-topic-name-group
spring.kafka.consumer.enable-auto-commit=true
topic-to-consume-message=your-topic-name
```

Второй параметр нужен для consumer'ов, дальше мы рассмотрим его необходимость. Enable auto commit
означает, что consumer сам периодически будет отмечать сообщения как обработанные. Это нужно для
того, чтобы при повторном запуске сервиса не читали одни и те же сообщения снова.

> Топик, из которого читаем сообщение, тот же, куда основной сервис их отправляет.

Пример kafka Producer'a

```java

@Service
public class KafkaProducerService {
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private final String topic;

    public KafkaProducerService(KafkaTemplate<String, String> kafkaTemplate,
                                ObjectMapper objectMapper,
                                @Value("{topic-to-send-message}") String topic) {
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
        this.topic = topic;
    }
    
    public void sendMessage(DtoMessage dtoMessage) {
        String message = objectMapper.writeValueAsString(dtoMessage);
        
        // Kafka Producer отправляет сообщения также асинхронно, подробнее об этом далее в модуле
        CompletableFuture<SendResult<String, String>> sendResult = kafkaTemplate.send(topic, message);
    }
}

```

Пример kafka Consumer'a

```java

@Service
public class KafkaConsumerService {
    private static final Logger LOGGER = LoggerFactory.getLogger(KafkaListener.class);

    private ObjectMapper objectMapper;

    public KafkaConsumerService(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = {"${topic-to-consume-message}"})
    public void consumeMessage(String message) {
        DtoMessage parsedMessage = objectMapper.readValue(message, DtoMessage.class);
        LOGGER.info("Retrieved message {}", message);
    }
}

```

Запустите оба микросервиса и вызовите HTTP endpoint на стороне основного сервиса.
Если в аудит-сервисе вы увидите сообщение в логах/дебаге, значит, все настройки верны.