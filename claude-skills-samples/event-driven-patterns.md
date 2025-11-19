# Event-Driven Architecture Patterns Skill

## Description
Enforce Home Depot's Kafka event-driven architecture standards, including event schemas, topic naming, consumer patterns, and error handling. Ensures consistency across all services publishing and consuming events.

**Use when:** Creating Kafka producers/consumers, defining event schemas, setting up event-driven workflows, or integrating services via events.

**Activate for phrases like:** "Kafka", "event", "publish", "subscribe", "topic", "consumer", "producer", "message", "event-driven", "async", "stream", "Avro", "schema"

## Instructions

### 1. Topic Naming Convention

Use the standard format for all Kafka topics:

```
Pattern: {domain}.{entity}.{event-type}
Examples:
  - order.checkout.completed
  - inventory.position.updated
  - customer.profile.created
  - payment.transaction.processed
  - fulfillment.shipment.dispatched
```

**Naming Rules:**
- All lowercase
- Dots as separators
- Domain = business domain (order, inventory, customer, etc.)
- Entity = the thing that changed
- Event type = past tense verb (created, updated, deleted, processed)

**Topic Configuration:**
```java
@Configuration
public class KafkaTopicConfig {

    // Standard topic settings for Home Depot
    public static final int DEFAULT_PARTITIONS = 12;
    public static final short DEFAULT_REPLICATION = 3;
    public static final String RETENTION_MS = "604800000"; // 7 days

    @Bean
    public NewTopic orderCompletedTopic() {
        return TopicBuilder.name("order.checkout.completed")
            .partitions(DEFAULT_PARTITIONS)
            .replicas(DEFAULT_REPLICATION)
            .config(TopicConfig.RETENTION_MS_CONFIG, RETENTION_MS)
            .config(TopicConfig.CLEANUP_POLICY_CONFIG, "delete")
            .build();
    }

    // Compacted topic for state (e.g., latest inventory position)
    @Bean
    public NewTopic inventoryPositionTopic() {
        return TopicBuilder.name("inventory.position.current")
            .partitions(DEFAULT_PARTITIONS)
            .replicas(DEFAULT_REPLICATION)
            .config(TopicConfig.CLEANUP_POLICY_CONFIG, "compact")
            .config(TopicConfig.MIN_COMPACTION_LAG_MS_CONFIG, "3600000") // 1 hour
            .build();
    }
}
```

### 2. Event Schema Standards (Avro)

Use the Home Depot event envelope:

```avro
// Standard event envelope - ALL events must use this
{
  "type": "record",
  "name": "EventEnvelope",
  "namespace": "com.homedepot.events",
  "fields": [
    {"name": "eventId", "type": "string", "doc": "UUID for idempotency"},
    {"name": "eventType", "type": "string", "doc": "Full event type name"},
    {"name": "eventTime", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "source", "type": "string", "doc": "Service that produced event"},
    {"name": "correlationId", "type": "string", "doc": "Trace correlation ID"},
    {"name": "version", "type": "string", "doc": "Schema version"},
    {"name": "payload", "type": "bytes", "doc": "Avro-encoded domain event"}
  ]
}
```

Domain event schema example:

```avro
// order.checkout.completed event payload
{
  "type": "record",
  "name": "OrderCompletedEvent",
  "namespace": "com.homedepot.events.order",
  "doc": "Published when customer completes checkout",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "storeId", "type": ["null", "string"], "default": null},
    {"name": "orderTotal", "type": {
      "type": "record",
      "name": "Money",
      "fields": [
        {"name": "amount", "type": "bytes", "logicalType": "decimal", "precision": 10, "scale": 2},
        {"name": "currency", "type": "string", "default": "USD"}
      ]
    }},
    {"name": "items", "type": {
      "type": "array",
      "items": {
        "type": "record",
        "name": "OrderItem",
        "fields": [
          {"name": "skuId", "type": "string"},
          {"name": "quantity", "type": "int"},
          {"name": "unitPrice", "type": "Money"}
        ]
      }
    }},
    {"name": "fulfillmentType", "type": {
      "type": "enum",
      "name": "FulfillmentType",
      "symbols": ["SHIP_TO_HOME", "BOPIS", "BOSS", "DELIVERY"]
    }},
    {"name": "metadata", "type": {"type": "map", "values": "string"}, "default": {}}
  ]
}
```

### 3. Producer Implementation

```java
@Service
@Slf4j
public class OrderEventPublisher {

    private final KafkaTemplate<String, EventEnvelope> kafkaTemplate;
    private final SchemaRegistryClient schemaRegistry;
    private final MeterRegistry meterRegistry;

    private static final String TOPIC = "order.checkout.completed";

    public void publishOrderCompleted(Order order) {
        // Generate idempotent event ID
        String eventId = UUID.randomUUID().toString();

        // Build domain event
        OrderCompletedEvent domainEvent = OrderCompletedEvent.newBuilder()
            .setOrderId(order.getId())
            .setCustomerId(order.getCustomerId())
            .setStoreId(order.getStoreId())
            .setOrderTotal(toMoney(order.getTotal()))
            .setItems(toOrderItems(order.getItems()))
            .setFulfillmentType(toFulfillmentType(order.getFulfillmentType()))
            .setMetadata(Map.of(
                "channel", order.getChannel(),
                "deviceType", order.getDeviceType()
            ))
            .build();

        // Wrap in envelope
        EventEnvelope envelope = EventEnvelope.newBuilder()
            .setEventId(eventId)
            .setEventType("order.checkout.completed")
            .setEventTime(Instant.now().toEpochMilli())
            .setSource("order-service")
            .setCorrelationId(MDC.get("correlationId"))
            .setVersion("1.0")
            .setPayload(serializeToBytes(domainEvent))
            .build();

        // Use orderId as partition key for ordering guarantee
        String partitionKey = order.getId();

        // Publish with callback
        CompletableFuture<SendResult<String, EventEnvelope>> future =
            kafkaTemplate.send(TOPIC, partitionKey, envelope);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to publish order completed event", ex);
                meterRegistry.counter("kafka.publish.error",
                    "topic", TOPIC,
                    "error", ex.getClass().getSimpleName()
                ).increment();

                // Send to dead letter for retry
                deadLetterPublisher.send(envelope, ex);
            } else {
                log.info("Published order completed event: eventId={}, partition={}, offset={}",
                    eventId,
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset()
                );
                meterRegistry.counter("kafka.publish.success", "topic", TOPIC).increment();
            }
        });
    }
}
```

### 4. Consumer Implementation

```java
@Service
@Slf4j
public class OrderEventConsumer {

    private final OrderFulfillmentService fulfillmentService;
    private final ProcessedEventRepository processedEvents;
    private final MeterRegistry meterRegistry;

    @KafkaListener(
        topics = "order.checkout.completed",
        groupId = "fulfillment-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    @Transactional
    public void handleOrderCompleted(
            @Payload EventEnvelope envelope,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        String eventId = envelope.getEventId();
        Timer.Sample timer = Timer.start(meterRegistry);

        try {
            // Idempotency check - prevent duplicate processing
            if (processedEvents.exists(eventId)) {
                log.info("Skipping duplicate event: eventId={}", eventId);
                ack.acknowledge();
                return;
            }

            // Set correlation ID for distributed tracing
            MDC.put("correlationId", envelope.getCorrelationId());
            MDC.put("eventId", eventId);

            // Deserialize domain event
            OrderCompletedEvent event = deserialize(envelope.getPayload(), OrderCompletedEvent.class);

            log.info("Processing order completed event: orderId={}", event.getOrderId());

            // Process event
            fulfillmentService.initiateFullfillment(event);

            // Mark as processed (for idempotency)
            processedEvents.save(new ProcessedEvent(eventId, Instant.now()));

            // Acknowledge only after successful processing
            ack.acknowledge();

            timer.stop(meterRegistry.timer("kafka.consume.success",
                "topic", "order.checkout.completed",
                "consumer_group", "fulfillment-service"
            ));

        } catch (RetryableException e) {
            // Don't acknowledge - will be redelivered
            log.warn("Retryable error processing event: eventId={}", eventId, e);
            meterRegistry.counter("kafka.consume.retry", "topic", "order.checkout.completed").increment();
            throw e;

        } catch (Exception e) {
            log.error("Non-retryable error processing event: eventId={}", eventId, e);
            meterRegistry.counter("kafka.consume.error", "topic", "order.checkout.completed").increment();

            // Send to dead letter topic
            deadLetterPublisher.send("order.checkout.completed.dlq", envelope, e);

            // Acknowledge to prevent infinite loop
            ack.acknowledge();

        } finally {
            MDC.clear();
        }
    }
}
```

### 5. Consumer Group Configuration

```java
@Configuration
@EnableKafka
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, EventEnvelope> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaBootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "fulfillment-service");

        // Exactly-once semantics
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        // Consumer settings
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 min

        // Schema registry
        props.put(KafkaAvroDeserializerConfig.SCHEMA_REGISTRY_URL_CONFIG, schemaRegistryUrl);
        props.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, true);

        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, EventEnvelope> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, EventEnvelope> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());

        // Manual acknowledgment for at-least-once delivery
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);

        // Concurrent consumers (one per partition typically)
        factory.setConcurrency(3);

        // Error handling
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),
            new FixedBackOff(1000L, 3L)  // 3 retries, 1 second apart
        ));

        return factory;
    }
}
```

### 6. Dead Letter Queue Handling

```java
@Service
@Slf4j
public class DeadLetterQueueProcessor {

    // DLQ naming: original-topic.dlq
    private static final String DLQ_SUFFIX = ".dlq";

    @KafkaListener(
        topicPattern = ".*\\.dlq",
        groupId = "dlq-processor"
    )
    public void processDeadLetter(
            ConsumerRecord<String, EventEnvelope> record,
            @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) String errorMessage,
            @Header(KafkaHeaders.DLT_ORIGINAL_TOPIC) String originalTopic) {

        EventEnvelope envelope = record.value();

        log.error("Processing dead letter: eventId={}, originalTopic={}, error={}",
            envelope.getEventId(), originalTopic, errorMessage);

        // Alert on critical events
        if (isCriticalEvent(originalTopic)) {
            alertService.sendPagerDutyAlert(
                "Critical event in DLQ",
                Map.of(
                    "eventId", envelope.getEventId(),
                    "topic", originalTopic,
                    "error", errorMessage
                )
            );
        }

        // Store for manual review
        dlqRepository.save(DeadLetterRecord.builder()
            .eventId(envelope.getEventId())
            .originalTopic(originalTopic)
            .payload(envelope)
            .errorMessage(errorMessage)
            .receivedAt(Instant.now())
            .status(DLQStatus.PENDING_REVIEW)
            .build()
        );
    }

    // Replay mechanism for fixed events
    public void replayEvent(String dlqRecordId) {
        DeadLetterRecord record = dlqRepository.findById(dlqRecordId)
            .orElseThrow();

        kafkaTemplate.send(record.getOriginalTopic(), record.getPayload());

        record.setStatus(DLQStatus.REPLAYED);
        record.setReplayedAt(Instant.now());
        dlqRepository.save(record);
    }
}
```

### 7. Event Versioning and Schema Evolution

```java
// Schema evolution rules:
// - Adding optional fields: ALLOWED
// - Removing fields: REQUIRES new major version
// - Changing field types: REQUIRES new major version
// - Renaming fields: REQUIRES new major version

@Service
public class SchemaCompatibilityChecker {

    public void validateSchemaChange(String subject, Schema newSchema) {
        // Check compatibility with existing schema
        CompatibilityLevel level = schemaRegistry.getCompatibilityLevel(subject);

        boolean isCompatible = schemaRegistry.testCompatibility(subject, newSchema);

        if (!isCompatible) {
            throw new SchemaIncompatibleException(
                "New schema is not " + level + " compatible with existing schema"
            );
        }
    }
}

// Handling multiple schema versions in consumer
@KafkaListener(topics = "order.checkout.completed")
public void handleOrderWithVersioning(@Payload GenericRecord record) {
    String version = record.getSchema().getFullName();

    switch (version) {
        case "com.homedepot.events.order.OrderCompletedEventV1":
            handleV1(record);
            break;
        case "com.homedepot.events.order.OrderCompletedEventV2":
            handleV2(record);
            break;
        default:
            log.warn("Unknown event version: {}", version);
            // Forward to DLQ for manual handling
    }
}
```

### 8. Monitoring and Observability

```java
@Component
public class KafkaMetrics {

    private final MeterRegistry registry;

    // Track consumer lag
    @Scheduled(fixedRate = 30000)
    public void recordConsumerLag() {
        adminClient.listConsumerGroupOffsets("fulfillment-service")
            .partitionsToOffsetAndMetadata()
            .thenAccept(offsets -> {
                offsets.forEach((tp, offset) -> {
                    registry.gauge("kafka.consumer.lag",
                        Tags.of("topic", tp.topic(), "partition", String.valueOf(tp.partition())),
                        offset.offset()
                    );
                });
            });
    }

    // Track publish latency
    public void recordPublishLatency(String topic, Duration latency) {
        registry.timer("kafka.publish.latency", "topic", topic)
            .record(latency);
    }

    // Track processing time
    public void recordProcessingTime(String topic, String consumerGroup, Duration duration) {
        registry.timer("kafka.consume.processing_time",
            "topic", topic,
            "consumer_group", consumerGroup
        ).record(duration);
    }
}

// Splunk logging for event tracking
@Slf4j
public class EventAuditLogger {

    public void logEventPublished(EventEnvelope envelope, String topic) {
        log.info(
            "EVENT_PUBLISHED topic={} eventId={} eventType={} correlationId={} timestamp={}",
            topic,
            envelope.getEventId(),
            envelope.getEventType(),
            envelope.getCorrelationId(),
            envelope.getEventTime()
        );
    }

    public void logEventConsumed(EventEnvelope envelope, String consumerGroup, Duration processingTime) {
        log.info(
            "EVENT_CONSUMED eventId={} consumerGroup={} processingTimeMs={} correlationId={}",
            envelope.getEventId(),
            consumerGroup,
            processingTime.toMillis(),
            envelope.getCorrelationId()
        );
    }
}
```

### 9. Testing Event-Driven Code

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"order.checkout.completed"})
class OrderEventIntegrationTest {

    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;

    @Autowired
    private KafkaTemplate<String, EventEnvelope> kafkaTemplate;

    @Autowired
    private OrderEventConsumer consumer;

    @Test
    void shouldProcessOrderCompletedEvent() throws Exception {
        // Given
        EventEnvelope envelope = createTestEnvelope();

        // When
        kafkaTemplate.send("order.checkout.completed", envelope).get();

        // Then
        await().atMost(10, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                verify(fulfillmentService).initiateFullfillment(any());
            });
    }

    @Test
    void shouldHandleDuplicateEvents() throws Exception {
        // Given
        EventEnvelope envelope = createTestEnvelope();

        // When - send same event twice
        kafkaTemplate.send("order.checkout.completed", envelope).get();
        kafkaTemplate.send("order.checkout.completed", envelope).get();

        // Then - should only process once
        await().atMost(10, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                verify(fulfillmentService, times(1)).initiateFullfillment(any());
            });
    }

    @Test
    void shouldSendToDeadLetterOnFailure() throws Exception {
        // Given
        EventEnvelope envelope = createTestEnvelope();
        doThrow(new RuntimeException("Test failure"))
            .when(fulfillmentService).initiateFullfillment(any());

        // When
        kafkaTemplate.send("order.checkout.completed", envelope).get();

        // Then
        ConsumerRecord<String, EventEnvelope> dlqRecord = KafkaTestUtils.getSingleRecord(
            dlqConsumer, "order.checkout.completed.dlq"
        );
        assertThat(dlqRecord.value().getEventId()).isEqualTo(envelope.getEventId());
    }
}
```

## Output Format

When creating event-driven code, always include:
1. Topic naming following domain.entity.event pattern
2. Avro schema with EventEnvelope wrapper
3. Idempotent consumers with duplicate detection
4. Dead letter queue handling
5. Proper acknowledgment patterns
6. Metrics and observability
7. Integration tests with EmbeddedKafka

## Tools Available
All tools (Read, Edit, Write, Bash, Grep, Glob)
