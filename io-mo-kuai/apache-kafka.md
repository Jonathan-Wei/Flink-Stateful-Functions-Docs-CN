# Apache Kafka

 有状态功能提供了一个Apache Kafka I/O模块，用于读取和写入Kafka Topic。它基于Apache Flink的通用[Kafka连接器，](https://ci.apache.org/projects/flink/flink-docs-stable/dev/connectors/kafka.html)并提供一次精确的处理语义。Kafka I/O模块可以用Yaml或Java配置。

## 依赖项

要在Java中使用Kafka I/O模块，请在pom中加入以下依赖项。

```text
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>statefun-kafka-io</artifactId>
	<version>2.0.0</version>
	<scope>provided</scope>
</dependency>
```

## Kafka入口规范

 `KafkaIngressSpec`声明了使用Kafka集群的入口规范。

它接受以下参数：

1. 与该入口相关的入口标识符
2. Topic名称/Topic名称列表
3. 引导服务器的地址
4. 要使用的消费者组ID
5. 用于从Kafka反序列化数据的`KafkaIngressDeserializer`（仅限Java）
6. 开始消费的位置

{% tabs %}
{% tab title="嵌入模块" %}
```java
package org.apache.flink.statefun.docs.io.kafka;

import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.IngressIdentifier;
import org.apache.flink.statefun.sdk.io.IngressSpec;
import org.apache.flink.statefun.sdk.kafka.KafkaIngressBuilder;
import org.apache.flink.statefun.sdk.kafka.KafkaIngressStartupPosition;

public class IngressSpecs {

  public static final IngressIdentifier<User> ID =
      new IngressIdentifier<>(User.class, "example", "input-ingress");

  public static final IngressSpec<User> kafkaIngress =
      KafkaIngressBuilder.forIdentifier(ID)
          .withKafkaAddress("localhost:9092")
          .withConsumerGroupId("greetings")
          .withTopic("my-topic")
          .withDeserializer(UserDeserializer.class)
          .withStartupPosition(KafkaIngressStartupPosition.fromLatest())
          .build();
}
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
version: "1.0"

module:
    meta:
    type: remote
spec:
    ingresses:
    - ingress:
        meta:
            type: statefun.kafka.io/routable-protobuf-ingress
            id: example/user-ingress
        spec:
            address: kafka-broker:9092
            consumerGroupId: routable-kafka-e2e
            startupPosition:
                type: earliest
            topics:
              - topic: messages-1
                typeUrl: org.apache.flink.statefun.docs.models.User
                targets:
                  - example-namespace/my-function-1
                  - example-namespace/my-function-2
```
{% endtab %}
{% endtabs %}

入口还接受Propertie，使用`KafkaIngressBuilder`与属性（properties）直接配置Kafka客户端。有关可用属性的完整列表，请参阅Kafka消费者配置文档。请注意，使用命名方法（如`KafkaIngressBuilder#withConsumerGroupId（String）`）传递的配置将具有更高的优先级，并覆盖所提供属性中各自的设置。

### 启动位置

入口允许将启动位置配置为以下之一：

**组偏移量（默认）**

从指定消费者组提交给Kafka的偏移量开始。

{% tabs %}
{% tab title="嵌入式模块" %}
```java
KafkaIngressStartupPosition#fromGroupOffsets();
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
startupPosition:
    type: group-offsets
```
{% endtab %}
{% endtabs %}

#### **Earlist**

从最早的偏移量开始

{% tabs %}
{% tab title="嵌入式模块" %}
```text
KafkaIngressStartupPosition#fromEarliest();
```
{% endtab %}

{% tab title="远程模块" %}
```text
startupPosition:
    type: earliest
```
{% endtab %}
{% endtabs %}

**Latest**

从最新的偏移量开始。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
KafkaIngressStartupPosition#fromLatest();
```
{% endtab %}

{% tab title="远程模块" %}
```text
startupPosition:
    type: latest
```
{% endtab %}
{% endtabs %}

**指定偏移量**

从特定的偏移量开始，该偏移量定义为分区到目标起始偏移量的映射。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
Map<TopicPartition, Long> offsets = new HashMap<>();
offsets.add(new TopicPartition("user-topic", 0), 91);
offsets.add(new TopicPartition("user-topic", 11), 11);
offsets.add(new TopicPartition("user-topic", 8), 8);

KafkaIngressStartupPosition#fromSpecificOffsets(offsets);
```
{% endtab %}

{% tab title="远程模块" %}
```text
startupPosition:
    type: specific-offsets
    offsets:
        - user-topic/0: 91
        - user-topic/1: 11
        - user-topic/2: 8
```
{% endtab %}
{% endtabs %}

#### **Date**

从提取时间大于或等于指定日期的偏移量开始。

{% tabs %}
{% tab title="嵌入式模式" %}
```text
KafkaIngressStartupPosition#fromDate(ZonedDateTime.now());
```
{% endtab %}

{% tab title="远程模式" %}
```text
startupPosition:
    type: date
    date: 2020-02-01 04:15:00.00 Z
```
{% endtab %}
{% endtabs %}

 启动时，如果分区的指定启动偏移量超出范围或不存在（如果将入口配置为从组偏移量，特定偏移量或日期开始，则可能是这种情况）将回退到使用所配置的位置`KafkaIngressBuilder#withAutoOffsetResetPosition(KafkaIngressAutoResetPosition)`。默认情况下，此位置设置为最新位置。

### Kafka **反序列化器**

 使用Java api时，Kafka入口需要知道如何将Kafka中的二进制数据转换为Java对象。在`KafkaIngressDeserializer`允许用户指定这样的一个模式。每条Kafka消息都会调用`T deserialize(ConsumerRecord<byte[], byte[]> record)`方法，并从Kafka传递键，值和元数据。

```java
package org.apache.flink.statefun.docs.io.kafka;

import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.IOException;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.kafka.KafkaIngressDeserializer;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserDeserializer implements KafkaIngressDeserializer<User> {

	private static Logger LOG = LoggerFactory.getLogger(UserDeserializer.class);

	private final ObjectMapper mapper = new ObjectMapper();

	@Override
	public User deserialize(ConsumerRecord<byte[], byte[]> input) {
		try {
			return mapper.readValue(input.value(), User.class);
		} catch (IOException e) {
			LOG.debug("Failed to deserialize record", e);
			return null;
		}
	}
}
```

## **Kafka出口规范**

`KafkaEgressBuilder`声明用于将数据写出到Kafka集群的出口规范。

它接受以下参数：

1. 与此出口关联的出口标识符
2. 引导服务器的地址
3. `KafkaEgressSerializer`序列化数据到Kafka（仅适用于Java的）
4. 容错语义
5. Kafka Producer的属性

{% tabs %}
{% tab title="嵌入式模块" %}
```java
package org.apache.flink.statefun.docs.io.kafka;

import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.EgressIdentifier;
import org.apache.flink.statefun.sdk.io.EgressSpec;
import org.apache.flink.statefun.sdk.kafka.KafkaEgressBuilder;

public class EgressSpecs {

  public static final EgressIdentifier<User> ID =
      new EgressIdentifier<>("example", "output-egress", User.class);

  public static final EgressSpec<User> kafkaEgress =
      KafkaEgressBuilder.forIdentifier(ID)
          .withKafkaAddress("localhost:9092")
          .withSerializer(UserSerializer.class)
          .build();
}
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
version: "1.0"

module:
    meta:
    type: remote
spec:
    egresses:
      - egress:
          meta:
            type: statefun.kafka.io/generic-egress
            id: example/output-messages
          spec:
            address: kafka-broker:9092
            deliverySemantic:
              type: exactly-once
              transactionTimeoutMillis: 100000
            properties:
              - foo.config: bar
```
{% endtab %}
{% endtabs %}

 请参阅Kafka [Producer配置](https://docs.confluent.io/current/installation/configuration/producer-configs.html)文档以获取可用属性的完整列表。

### Kafka出口和容错

启用容错功能后，Kafka出口可以提供准确的一次交付保证。可以选择三种不同的操作模式。

#### **None**

没有任何保证，产生的记录可能会丢失或重复。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
KafkaEgressBuilder#withNoProducerSemantics();
```
{% endtab %}

{% tab title="远程模块" %}
```text
deliverySemantic:
    type: none
```
{% endtab %}
{% endtabs %}

#### **至少一次**

有状态函数将确保不会丢失任何记录，但可以重复记录。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
KafkaEgressBuilder#withAtLeastOnceProducerSemantics();
```
{% endtab %}

{% tab title="远程模块" %}
```text
deliverySemantic:
    type: at-least-once
```
{% endtab %}
{% endtabs %}

#### **恰好一次**

有状态函数使用Kafka事务提供一次精确的语义。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
KafkaEgressBuilder#withExactlyOnceProducerSemantics(Duration.minutes(15));
```
{% endtab %}

{% tab title="远程模块" %}
```text
deliverySemantic:
    type: exactly-once
    transactionTimeoutMillis: 900000 # 15 min
```
{% endtab %}
{% endtabs %}

### Kafka序列化器

 使用Java api时，Kafka出口需要知道如何将Java对象转换为二进制数据。在`KafkaEgressSerializer`允许用户指定这样的一个模式。`ProducerRecord<byte[], byte[]> serialize(T out)`为每条消息调用该方法，从而允许用户设置键，值和其他元数据。

```java
package org.apache.flink.statefun.docs.io.kafka;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.kafka.KafkaEgressSerializer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserSerializer implements KafkaEgressSerializer<User> {

  private static final Logger LOG = LoggerFactory.getLogger(UserSerializer.class);

  private static final String TOPIC = "user-topic";

  private final ObjectMapper mapper = new ObjectMapper();

  @Override
  public ProducerRecord<byte[], byte[]> serialize(User user) {
    try {
      byte[] key = user.getUserId().getBytes();
      byte[] value = mapper.writeValueAsBytes(user);

      return new ProducerRecord<>(TOPIC, key, value);
    } catch (JsonProcessingException e) {
      LOG.info("Failed to serializer user", e);
      return null;
    }
  }
}
```

