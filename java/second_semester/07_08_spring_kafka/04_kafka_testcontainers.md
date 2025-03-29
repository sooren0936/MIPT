# Тестирование Kafka с помощью Testcontainers

Модуль Kafka также присутствует в Testcontainers, так что удобно внедрить его для тестирования. Мы
`начнем с тестирования producer'а (первого микросервиса), а затем перейдем к
consumer'у (второго).
`

## Тестирование producer'а

Добавьте Testcontainers Kafka в зависимости:

```xml

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
```

Тест для producer'a будет выглядеть так:

```java

@SpringBootTest(
        classes = {KafkaProducerService.class},
        properties = {"topic-to-send-message=your-topic-config"}
)
@Import({KafkaAutoConfiguration.class, ObjectMapperTestConfig.class})
@Testcontainers
class KafkaProducerServiceTest {

    @TestConfiguration
    static class ObjectMapperTestConfig {
        @Bean
        public ObjectMapper objectMapper() {
            return new ObjectMapper();
        }
    }

    @Container
    @ServiceConnection
    public static final KafkaContainer KAFKA = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @Autowired
    private KafkaProducerService kafkaProducerService;
    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void shouldSendMessageToKafkaSuccessfully() {
        DtoMessage testDtoMessage = new DtoMessage();

        assertDoesNotThrow(() -> kafkaProducerService.sendMessage(testDtoMessage));

        KafkaTestConsumer consumer = new KafkaTestConsumer(KAFKA.getBootstrapServers(), "some-group-id");
        consumer.subscribe(List.of("some-test-topic"));

        ConsumerRecords<String, String> records = consumer.poll();
        assertEquals(1, records.count());
        records.iterator().forEachRemaining(
                record -> {
                    DtoMessage message = objectMapper.readValue(record.value(), DtoMessage.class);
                    assertEquals(testDtoMessage, message);
                }
        );
    }
}

public class KafkaTestConsumer {

    private final KafkaConsumer<String, String> consumer;

    public KafkaTestConsumer(String bootstrapServers, String groupId) {
        Properties props = new Properties();

        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        this.consumer = new KafkaConsumer<>(props);
    }

    public void subscribe(List<String> topics) {
        consumer.subscribe(topics);
    }

    public ConsumerRecords<String, String> poll() {
        return consumer.poll(Duration.ofSeconds(5));
    }

}
```

Происходит здесь следующее:

1. С помощью аннотации `@Testcontainers` запускаем контейнер Kafka. Аннотация `@ServiceConnection`
   дает Spring команду переопределить стандартные конфиги для соединения с Kafka.
2. В тесте вызываем `kafkaProducerService.sendMessage`, что инициирует отправку сообщения в Kafka.
3. Создаем `KafkaConsumer` (для удобства добавили кастомную обертку `KafkaTestConsumer`).
4. Запрашиваем сообщения из topic (его мы задали через конфиги в `@SpringBootTest`).
5. Проверяем, что их количество равно 1.
6. Валидируем контент сообщения.

## Тестирование consumer'а

```java

@SpringBootTest(
        classes = {KafkaConsumerService.class},
        properties = {
                "topic-to-consume-message=your-test-topic",
                "spring.kafka.consumer.group-id=some-consumer-group"
        }
)
@Import({KafkaAutoConfiguration.class, ObjectMapperTestConfig.class})
@Testcontainers
class KafkaConsumerServiceTest {

    @TestConfiguration
    static class ObjectMapperTestConfig {
        @Bean
        public ObjectMapper objectMapper() {
            return new ObjectMapper();
        }
    }

    @Container
    @ServiceConnection
    public static final KafkaContainer KAFKA = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @MockBean
    private MessageProcessor messageProcessor;
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void shouldSendMessageToKafkaSuccessfully() {
        kafkaTemplate.send("some-test-topic", new DtoMessage());

        await().atMost(Duration.ofSeconds(5))
                .pollDelay(Duration.ofSeconds(1))
                .untilAsserted(() -> Mockito.verify(
                                messageProcessor, times(1))
                        .processMessage(eq(new DtoMessage()))
                );
    }
}
```

Сначала мы отправляем запрос в topic, а потом проверяем - `MessageProcessor` его обработал (это пример некоторого
абстрактного обработчика, которые идет уже после того как сообщение приняли). Поскольку consumer работает в отдельном
потоке, проверять условие синхронно сразу после отправки в topic будет неправильно.

Для этого мы используем библиотеку [awaitility](http://www.awaitility.org/): она проверяет условие несколько раз в
течение заданного периода, пока оно не будет выполнено.

Вместо messageProcessor можно реализовать логику по другому, например можно обращаться в бд и проверять созданные
сущности на их наличие