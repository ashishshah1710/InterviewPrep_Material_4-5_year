Certainly! Below is an **extremely detailed guide** for Java developers wanting to master **Apache Kafka**, with thorough coverage of its internal mechanisms, advanced features, and robust coding examples. This is a large document â use the Table of Contents to jump to the desired section.

---

# ð Apache Kafka for Java Developers: A Deep Dive

## **Table of Contents**
1. [Introduction to Kafka](#introduction)
2. [Kafka Producer Internals](#producer-internals)
    - [Producing Records (Basic Example)](#producer-basic)
    - [Producer Configuration](#producer-config)
    - [Producer Internals: Batching, Buffering, & Retries](#producer-advanced)
    - [Idempotence and Exactly-Once Guarantees](#producer-eos)
3. [Kafka Consumer Internals](#consumer-internals)
    - [Consuming Records (Basic Example)](#consumer-basic)
    - [Consumer Configuration](#consumer-config)
    - [Consumer Groups & Partition Rebalancing](#consumer-groups)
    - [Offsets, Commit, Manual Control](#consumer-offsets)
4. [Topic Partitioning Strategy](#partitioning)
    - [Partitioning Algorithm](#partitioning-algo)
    - [Custom Partitioners](#custom-partitioner)
5. [Exactly Once Semantics (EOS)](#eos)
6. [Kafka Streams](#kafka-streams)
    - [Architecture & DSL](#streams-arch)
    - [Word Count Example](#streams-wordcount)
    - [Windowed Aggregation](#streams-windowing)
    - [Interactive Queries](#streams-iq)
7. [Kafka Connect](#kafka-connect)
    - [Source and Sink Connectors](#source-sink)
    - [Running a Connector via REST](#connect-rest)
8. [Schema Registry](#schema-registry)
    - [Avro Schemas](#schema-avro)
    - [Protobuf/JSON Schemas](#schema-other)
    - [Java Serialization Process](#schema-java)
9. [Spring for Apache Kafka](#spring-kafka)
    - [Spring Producer/Consumer](#spring-producer)
    - [Error Handling, Batch, and Transactions](#spring-error-handling)
10. [Appendix: Useful Kafka CLI commands](#cli)
---

<a name="introduction"></a>
## 1. Introduction to Kafka

**Kafka** is a distributed, partitioned, replicated commit-log service for real-time data streams that powers event-driven microservices, data integration pipelines, and more.

- **Producer**: Publishes records (events) into Kafka topics.
- **Consumer**: Subscribes to topics and processes records.
- **Broker**: A single Kafka server.
- **Topic**: Category/feed name to which records are published.
- **Partition**: Segment of a topic.

---

<a name="producer-internals"></a>
## 2. Kafka Producer Internals

The **Producer** is responsible for sending data to Kafka topics, choosing partitions, and ensuring delivery guarantees.

<a name="producer-basic"></a>
### Producing Records (Basic Example)

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Properties;
import java.util.concurrent.Future;

public class SimpleProducer {
    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        ProducerRecord<String, String> rec = new ProducerRecord<>("my-topic", "my-key", "my-value");

        Future<RecordMetadata> future = producer.send(rec);
        RecordMetadata metadata = future.get(); // Waits for async send completion

        System.out.printf("Produced record to partition %d with offset %d%n",
                metadata.partition(), metadata.offset());

        producer.close();
    }
}
```

<a name="producer-config"></a>
### Producer Configuration

- **bootstrap.servers**: Broker address(es)
- **acks**: "0", "1", "all" (leader + in-sync replicas)
- **retries**: How many times to retry failed sends
- **linger.ms**: Max time to wait for more records before sending batch
- **batch.size**: Max batch size(bytes) to buffer before sending
- **compression.type**: "snappy", "gzip", etc.

**Example**:

```java
props.put("acks", "all");
props.put("retries", 3);
props.put("linger.ms", 5);
props.put("batch.size", 32768);
```

<a name="producer-advanced"></a>
### Producer Internals: Batching, Buffering, & Retries

- Each partition gets its own **buffer & batch**.
- `send()` is async; batches sent when size (`batch.size`) or time (`linger.ms`) reached.
- If a batch send fails (`retries`), temporarily held in memory, retried according to **backoff** policies.
- **In-flight requests**: Limit concurrent requests per broker for guaranteed ordering (`max.in.flight.requests.per.connection`).

<a name="producer-eos"></a>
### Idempotence, Transactions, Exactly Once Semantics

- Enable idempotence:  
  ```java
  props.put("enable.idempotence", "true");
  ```
  This lets the producer retry safely (no duplicates).

- **Transactions** allows atomic writes to multiple partitions and/or topics.
  ```java
  props.put("transactional.id", "my-transactional-producer");
  KafkaProducer<String, String> producer = new KafkaProducer<>(props);
  producer.initTransactions();
  producer.beginTransaction();
  producer.send(new ProducerRecord<>("topicA", "key", "valueA"));
  producer.send(new ProducerRecord<>("topicB", "key", "valueB"));
  producer.commitTransaction();
  producer.close();
  ```

- **Enabling Exactly-Once:**
    - On Producer: `enable.idempotence = true`, `acks=all`
    - On Consumer: commit offsets **in the same transaction**

---

<a name="consumer-internals"></a>
## 3. Kafka Consumer Internals

<a name="consumer-basic"></a>
### Consuming Records (Basic Example):

```java
import org.apache.kafka.clients.consumer.*;
import java.util.*;

public class SimpleConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("group.id", "my-consumer-group");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("auto.offset.reset", "earliest"); // or "latest"

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("my-topic"));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(java.time.Duration.ofMillis(100));
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("Received: %s-%s: %s%n", record.partition(), record.offset(), record.value());
            }
        }
    }
}
```

<a name="consumer-config"></a>
### Consumer Configuration

- **group.id**: logical consumer group name
- **auto.offset.reset**: what to do when there is no offset? ("earliest", "latest")
- **enable.auto.commit**: auto-commit offsets? (default: true)
- **max.poll.records**: batch size per poll

<a name="consumer-groups"></a>
### Consumer Groups & Partition Rebalancing

- Consumers in the same group **share** the partitions of the subscribed topic.
- Partition assignment:
    - If partitions > consumers: some consumers get >1 partition
    - If consumers > partitions: some idle consumers
- On consumer join/leave, **rebalance** triggers partition reassignment.
- Rebalancing can be controlled by `partition.assignment.strategy` (e.g., "range", "roundrobin").

<a name="consumer-offsets"></a>
### Offsets, Commit, Manual Control

- By default, the consumer **auto-commits** offsets after `auto.commit.interval.ms`.
- For **at-least-once**: process, then commit offsets (could process twice if crashed before commit).
- For **at-most-once**: commit offset before processing (risk data loss).
- **Manual commit** example:

```java
consumer.commitSync(); // synchronous commit
// or
consumer.commitAsync();
```

<div>
<b>Manual offset per record:</b>
</div>

```java
Map<TopicPartition, OffsetAndMetadata> commitMap = new HashMap<>();
commitMap.put(new TopicPartition("my-topic", 0), new OffsetAndMetadata(123L));
consumer.commitSync(commitMap);
```

---

<a name="partitioning"></a>
## 4. Topic Partitioning Strategy

### Why Partition?

- **Scalability**: More partitions = more parallelism.
- **Ordering**: Order guaranteed only within a partition.

<a name="partitioning-algo"></a>
### Default Partitioning Algorithm

- If a **key** is provided: `partition = hash(key) % numPartitions`
- Else: round-robin across partitions.

**Controlling partition:**
```java
ProducerRecord<String, String> record = new ProducerRecord<>("my-topic", null, "noKey", "value");
```

<a name="custom-partitioner"></a>
### Custom Partitioner Example

**Define a Partitioner:**

```java
import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;

import java.util.Map;

public class MyPartitioner implements Partitioner {
    @Override
    public void configure(Map<String, ?> configs) {}
    @Override
    public void close() {}

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        if (key != null && key.equals("critical")) {
            return 0; // Always send 'critical' keys to partition 0
        }
        return Math.abs(key.hashCode()) % cluster.partitionCountForTopic(topic);
    }
}
```

**Register it:**
```java
props.put("partitioner.class", "com.example.MyPartitioner");
```

---

<a name="eos"></a>
## 5. Exactly-Once Semantics (EOS)

- Enable Producer idempotence (`enable.idempotence=true`)
- Enable transactions (`transactional.id`, `initTransactions()`)
- For consumer-to-producer chains: use **read-process-write** with transactions

**Transactional Consumer-to-Producer Example:**

```java
props.put("enable.idempotence", "true");
props.put("transactional.id", "tx-producer-1");
KafkaProducer<String, String> producer = new KafkaProducer<>(props);
KafkaConsumer<String, String> consumer = ...

producer.initTransactions();
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    producer.beginTransaction();
    for (ConsumerRecord<String, String> record : records) {
        // Process and produce to another topic
        producer.send(new ProducerRecord<>("out-topic", record.key(), record.value()));
    }
    // Send offsets in the transaction
    Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
    for (TopicPartition p : records.partitions()) {
        long lastOffset = records.records(p).get(records.records(p).size() - 1).offset();
        offsets.put(p, new OffsetAndMetadata(lastOffset + 1));
    }
    producer.sendOffsetsToTransaction(offsets, "my-group");
    producer.commitTransaction();
}
```

---

<a name="kafka-streams"></a>
## 6. Kafka Streams

Kafka Streams is a **Java library** for stream processing on top of Kafka.

<a name="streams-arch"></a>
### Architecture & DSL

- **Topology:** Compose stream transformations.
- Fault-tolerant, distributed, embedded in your JVM.

**Stream API vs Table (KTable) API:**  
- `KStream`: Immutable sequence of records (think: event stream).
- `KTable`: Latest value for each key (think: changelog table).

<a name="streams-wordcount"></a>
### Word Count Example

```java
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;

import java.util.Arrays;
import java.util.Properties;

public class WordCountExample {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("application.id", "streams-wordcount");
        props.put("default.key.serde", Serdes.String().getClass().getName());
        props.put("default.value.serde", Serdes.String().getClass().getName());

        StreamsBuilder builder = new StreamsBuilder();

        KStream<String, String> textLines = builder.stream("TextLinesTopic");
        KTable<String, Long> wordCounts = textLines
                .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
                .groupBy((key, word) -> word)
                .count();

        wordCounts.toStream().to("WordsWithCountsTopic", Produced.with(Serdes.String(), Serdes.Long()));

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();

        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

<a name="streams-windowing"></a>
### Windowed Aggregation Example

```java
KTable<Windowed<String>, Long> windowedCounts = textLines
    .flatMapValues(v -> Arrays.asList(v.toLowerCase().split("\\W+")))
    .groupBy((k, word) -> word)
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
    .count();

windowedCounts.toStream().foreach((windowedKey, count) -> {
    System.out.println("Window: " + windowedKey.key() + " @ " +
        windowedKey.window().start() + "-" + windowedKey.window().end() + " count: " + count);
});
```

<a name="streams-iq"></a>
### Interactive Queries

Kafka Streams applications expose **local state stores** via interactive queries.

```java
ReadOnlyKeyValueStore<String, Long> store = streams.store(
  StoreQueryParameters.fromNameAndType("my-store", QueryableStoreTypes.keyValueStore()));

Long count = store.get("word");
```

---

<a name="kafka-connect"></a>
## 7. Kafka Connect

**Kafka Connect** provides pluggable **source/sink connectors** for moving data between Kafka and various systems (DBs, files, etc.)

<a name="source-sink"></a>
### Example: File Source Connector

```json
{
  "name": "local-file-source",
  "config": {
    "connector.class": "FileStreamSource",
    "tasks.max": "1",
    "file": "/tmp/test.txt",
    "topic": "test-file-topic"
  }
}
```

<a name="connect-rest"></a>
### Running Connectors via REST

Deploy with:
```bash
curl -X POST -H "Content-Type: application/json" --data @/path/to/connector.json http://localhost:8083/connectors
```

- **Cluster Mode:** Supports distributed/fault-tolerant workers
- **Transformations:** Single Message Transforms (SMTs) for logic on the fly
- **Converters:** Avro/JSON/String

---

<a name="schema-registry"></a>
## 8. Schema Registry

**Confluent Schema Registry** manages Avro, Protobuf, or JSON schemas for topic data, providing compatibility guarantees.

<a name="schema-avro"></a>
### Avro Schema Example

**Avro schema:**

```json
{
  "type": "record",
  "name": "User",
  "namespace": "com.example",
  "fields": [
    {"name": "name", "type": "string"},
    {"name": "favorite_number", "type": "int"},
    {"name": "favorite_color", "type": ["null", "string"], "default": null}
  ]
}
```

Auto-generate Java classes via Maven plugin:
```xml
<plugin>
  <groupId>org.apache.avro</groupId>
  <artifactId>avro-maven-plugin</artifactId>
  <version>1.11.1</version>
  <executions>
    <execution>
      <goals>
        <goal>schema</goal>
      </goals>
      <configuration>
        <sourceDirectory>${project.basedir}/src/main/avro</sourceDirectory>
        <outputDirectory>${project.basedir}/src/main/java</outputDirectory>
      </configuration>
    </execution>
  </executions>
</plugin>
```

<a name="schema-other"></a>
### Protobuf/JSON Schema

- Similar plugins/tools generate POJOs from schema definitions.

<a name="schema-java"></a>
### Producer/Consumer with Avro and Schema Registry

Add Maven dependencies:
```xml
<dependency>
  <groupId>io.confluent</groupId>
  <artifactId>kafka-avro-serializer</artifactId>
  <version>7.3.0</version>
</dependency>
<dependency>
  <groupId>org.apache.avro</groupId>
  <artifactId>avro</artifactId>
  <version>1.11.1</version>
</dependency>
```

**Producer Example:**

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("value.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("schema.registry.url", "http://localhost:8081");

KafkaProducer<String, User> producer = new KafkaProducer<>(props);
User user = User.newBuilder().setName("Alice")
    .setFavoriteNumber(42)
    .setFavoriteColor("blue").build();

producer.send(new ProducerRecord<>("avro-users", "aliceKey", user));
producer.close();
```

**Consumer Example:**
```java
props.put("key.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("value.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("schema.registry.url", "http://localhost:8081");
props.put("specific.avro.reader", "true");

KafkaConsumer<String, User> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("avro-users"));
while (true) {
    ConsumerRecords<String, User> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, User> record : records) {
        System.out.println(record.value().getName());
    }
}
```

---

<a name="spring-kafka"></a>
## 9. Spring for Apache Kafka

Spring Kafka provides high-level abstractions for working with Kafka in Spring Boot.

<a name="spring-producer"></a>
### Spring Kafka Producer Example

**pom.xml:**
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.1.2</version>
</dependency>
```

**Producer service:**
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class MyProducerService {
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void send(String topic, String key, String value) {
        kafkaTemplate.send(topic, key, value);
    }
}
```

<a name="spring-error-handling"></a>
### Spring Kafka Consumer Example

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class MyConsumer {
    @KafkaListener(topics = "my-topic", groupId = "my-group")
    public void listen(String message) {
        System.out.println("Received: " + message);
    }
}
```

**Error Handling:**
```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
      ConsumerFactory<String, String> consumerFactory) {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.setErrorHandler(new SeekToCurrentErrorHandler());
    return factory;
}
```

**Batch Consumption:**
```java
@KafkaListener(topics = "my-topic", batch = true)
public void listen(List<String> messages) {
    ...
}
```

**Transactional Producer:**
```java
@Bean
public KafkaTemplate<String, String> kafkaTemplate(ProducerFactory<String, String> pf) {
    KafkaTemplate<String, String> template = new KafkaTemplate<>(pf);
    template.setTransactionalIdPrefix("tx-");
    return template;
}
```

---

<a name="cli"></a>
## 10. Appendix: Useful Kafka CLI Commands

**Create Topic**:
```bash
kafka-topics.sh --create --topic my-topic --bootstrap-server localhost:9092 --partitions 6 --replication-factor 1
```
**List Topics**:
```bash
kafka-topics.sh --list --bootstrap-server localhost:9092
```
**Produce Messages (CLI)**:
```bash
kafka-console-producer.sh --broker-list localhost:9092 --topic my-topic
```
**Consume Messages (CLI)**:
```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-topic --from-beginning
```
**Describe Topic**:
```bash
kafka-topics.sh --describe --topic my-topic --bootstrap-server localhost:9092
```

---

# **Conclusion**

This guide covered Kafka's core APIs for Java, producer & consumer internals, partitioning, EOS, Kafka Streams, Connect, Schema Registry, and high-level abstraction with Spring Kafka. For real-world use, consult each sectionâs reference documentation for advanced configuration and tuning.

For an even more exhaustive repository of working examples, visit the open-source projects:
- [confluentinc/examples](https://github.com/confluentinc/examples)
- [spring-projects/spring-kafka-samples](https://github.com/spring-projects/spring-kafka/tree/main/samples)
- [apache/kafka-streams-examples](https://github.com/apache/kafka-streams-examples)

If you need live, ready-to-run example projects for anything above, let me know!
