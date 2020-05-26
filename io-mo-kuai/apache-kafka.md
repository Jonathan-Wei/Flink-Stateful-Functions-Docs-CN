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

**Latest**

从最新的偏移量开始。

**指定偏移量**

从特定的偏移量开始，该偏移量定义为分区到目标起始偏移量的映射。

#### **Date**

### Kafka **反序列化器**

## **Kafka出口规范**

