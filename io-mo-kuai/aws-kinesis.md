# AWS Kinesis

 有状态功能提供了一个AWS Kinesis I/O模块，用于读取和写入Kinesis流。它基于Apache Flink的[Kinesis连接器](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/connectors/kinesis.html)。Kinesis I/O模块可以用Yaml或Java配置。

## 依赖项

要在Java中使用Kinesis I / O模块，请在pom中添加以下依赖项。

```markup
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>statefun-kinesis-io</artifactId>
    <version>2.0.0</version>
    <scope>provided</scope>
</dependency>
```

## Kinesis入口规范

`KinesisIngressSpec`声明要从Kinesis流中消费的入口规范。

它接受以下参数：

1. AWS区域
2. AWS凭证提供者
3. KinesisIngressDeserializer，用于从Kinesis反序列化数据\(仅限Java\)
4. 流开始位置
5. Kinesis客户端的属性
6. 要消费的流的名称

{% tabs %}
{% tab title="嵌入式模块" %}
```java
package org.apache.flink.statefun.docs.io.kinesis;

import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.IngressIdentifier;
import org.apache.flink.statefun.sdk.io.IngressSpec;
import org.apache.flink.statefun.sdk.kinesis.auth.AwsCredentials;
import org.apache.flink.statefun.sdk.kinesis.ingress.KinesisIngressBuilder;
import org.apache.flink.statefun.sdk.kinesis.ingress.KinesisIngressStartupPosition;

public class IngressSpecs {

  public static final IngressIdentifier<User> ID =
      new IngressIdentifier<>(User.class, "example", "input-ingress");

  public static final IngressSpec<User> kinesisIngress =
      KinesisIngressBuilder.forIdentifier(ID)
          .withAwsRegion("us-west-1")
          .withAwsCredentials(AwsCredentials.fromDefaultProviderChain())
          .withDeserializer(UserDeserializer.class)
          .withStream("stream-name")
          .withStartupPosition(KinesisIngressStartupPosition.fromEarliest())
          .withClientConfigurationProperty("key", "value")
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
                type: statefun.kinesis.io/routable-protobuf-ingress
                id: example-namespace/messages
              spec:
                awsRegion:
                  type: specific
                  id: us-west-1
                awsCredentials:
                  type: basic
                  accessKeyId: my_access_key_id
                  secretAccessKey: my_secret_access_key
                startupPosition:
                  type: earliest
                streams:
                  - stream: stream-1
                    typeUrl: com.googleapis/org.apache.flink.statefun.docs.models.User
                    targets:
                      - example-namespace/my-function-1
                      - example-namespace/my-function-2
                  - stream: stream-2
                    typeUrl: com.googleapis/org.apache.flink.statefun.docs.models.User
                    targets:
                      - example-namespace/my-function-1
                clientConfigProperties:
                  - SocketTimeout: 9999
                  - MaxConnections: 15
```
{% endtab %}
{% endtabs %}

入口还接受使用`KinesisIngressBuilder#withClientConfigurationProperty()`来直接配置Kinesis客户端的属性。请参阅Kinesis [客户端配置](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/ClientConfiguration.html)文档以获取可用属性的完整列表。请注意，使用命名方法传递的配置将具有更高的优先级，并在提供的属性中覆盖它们各自的设置。

### 启动位置

入口允许将启动位置配置为以下之一：

#### **最新-Latest \(default\)**

{% tabs %}
{% tab title="嵌入式模块" %}
```text
KinesisIngressStartupPosition#fromLatest();
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
startupPosition:
    type: latest
```
{% endtab %}
{% endtabs %}

#### **最早-Earlist**

从尽可能早的位置开始消费。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
KinesisIngressStartupPosition#fromEarliest();
```
{% endtab %}

{% tab title="远程模块" %}
```text
startupPosition:
    type: earliest
```
{% endtab %}
{% endtabs %}

#### **Date**

从摄取时间大于或等于指定日期的偏移量开始。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
KinesisIngressStartupPosition#fromDate(ZonedDateTime.now());
```
{% endtab %}

{% tab title="远程模块" %}
```text
startupPosition:
    type: date
    date: 2020-02-01 04:15:00.00 Z
```
{% endtab %}
{% endtabs %}

### Kinesis **反序列化器**

Kinesis 入口需要知道如何将Kinesis中的二进制数据转换为Java对象。`KinesisIngressDeserializer`允许用户指定这样的模式。为每个Kinesis记录调用`T deserialize(IngressRecord IngressRecord)`方法，传递来自Kinesis的二进制数据和元数据。

```java
package org.apache.flink.statefun.docs.io.kinesis;

import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.IOException;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.kinesis.ingress.IngressRecord;
import org.apache.flink.statefun.sdk.kinesis.ingress.KinesisIngressDeserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserDeserializer implements KinesisIngressDeserializer<User> {

  private static Logger LOG = LoggerFactory.getLogger(UserDeserializer.class);

  private final ObjectMapper mapper = new ObjectMapper();

  @Override
  public User deserialize(IngressRecord ingressRecord) {
    try {
      return mapper.readValue(ingressRecord.getData(), User.class);
    } catch (IOException e) {
      LOG.debug("Failed to deserialize record", e);
      return null;
    }
  }
}
```

## Kinesis出口规格

`KinesisEgressBuilder`声明用于将数据写出到Kinesis流的出口规范。

它接受以下参数：

1. 与此出口关联的出口标识符
2. AWS凭证提供者
3. 用于将数据序列化到Kinesis中的`KinesisEgressSerializer`\(仅限Java\)
4. AWS区域
5. Kinesis客户端的属性
6. 施加背压前的最大未完成记录数

{% tabs %}
{% tab title="嵌入式模块" %}
```java
package org.apache.flink.statefun.docs.io.kinesis;

import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.EgressIdentifier;
import org.apache.flink.statefun.sdk.io.EgressSpec;
import org.apache.flink.statefun.sdk.kinesis.auth.AwsCredentials;
import org.apache.flink.statefun.sdk.kinesis.egress.KinesisEgressBuilder;

public class EgressSpecs {

  public static final EgressIdentifier<User> ID =
      new EgressIdentifier<>("example", "output-egress", User.class);

  public static final EgressSpec<User> kinesisEgress =
      KinesisEgressBuilder.forIdentifier(ID)
          .withAwsCredentials(AwsCredentials.fromDefaultProviderChain())
          .withAwsRegion("us-west-1")
          .withMaxOutstandingRecords(100)
          .withClientConfigurationProperty("key", "value")
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
                type: statefun.kinesis.io/generic-egress
                id: example/output-messages
              spec:
                awsRegion:
                  type: custom-endpoint
                  endpoint: https://localhost:4567
                  id: us-west-1
                awsCredentials:
                  type: profile
                  profileName: john-doe
                  profilePath: /path/to/profile/config
                maxOutstandingRecords: 9999
                clientConfigProperties:
                  - ThreadingModel: POOLED
                  - ThreadPoolSize: 10
```
{% endtab %}
{% endtabs %}

请参阅Kinesis [Producer默认配置属性](https://github.com/awslabs/amazon-kinesis-producer/blob/master/java/amazon-kinesis-producer-sample/default_config.properties)文档以获取可用属性的完整列表。

### Kinesis序列化器

 Kinesis出口需要知道如何将Java对象转换为二进制数据。`KinesisEgressSerializer`允许用户指定这样的一个模式。为每条消息调用`EgressRecord serialize(T value)`方法，从而允许用户设置值和其他元数据。

```java
package org.apache.flink.statefun.docs.io.kinesis;

import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.IOException;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.kinesis.egress.EgressRecord;
import org.apache.flink.statefun.sdk.kinesis.egress.KinesisEgressSerializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserSerializer implements KinesisEgressSerializer<User> {

  private static final Logger LOG = LoggerFactory.getLogger(UserSerializer.class);

  private static final String STREAM = "user-stream";

  private final ObjectMapper mapper = new ObjectMapper();

  @Override
  public EgressRecord serialize(User value) {
    try {
      return EgressRecord.newBuilder()
          .withPartitionKey(value.getUserId())
          .withData(mapper.writeValueAsBytes(value))
          .withStream(STREAM)
          .build();
    } catch (IOException e) {
      LOG.info("Failed to serializer user", e);
      return null;
    }
  }
}
```

## AWS **区域**

Kinesis入口和出口都可以配置到特定的AWS区域。

### 默认提供商链 **\(default\)**

咨询AWS的默认提供商链，以确定AWS区域。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
AwsRegion.fromDefaultProviderChain();
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
awsCredentials:
    type: default
```
{% endtab %}
{% endtabs %}

### **指定**

使用区域的唯一ID指定一个AWS区域。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
AwsRegion.of("us-west-1");
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
awsCredentials:
    type: specific
    id: us-west-1
```
{% endtab %}
{% endtabs %}

### **自定义端点**

通过非标准的AWS服务终端节点连接到AWS区域。通常仅用于开发和测试目的。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
AwsRegion.ofCustomEndpoint("https://localhost:4567", "us-west-1");
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
awsCredentials:
    type: custom-endpoint
    endpoint: https://localhost:4567
    id: us-west-1
```
{% endtab %}
{% endtabs %}

## AWS ****凭证

Kinesis入口和出口都可以使用标准AWS凭证提供程序进行配置。

### **默认提供者链（默认）**

请咨询AWS的默认提供商链，以确定AWS凭证。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
AwsCredentials.fromDefaultProviderChain();
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
awsCredentials:
    type: default
```
{% endtab %}
{% endtabs %}

### **Basic**

直接使用提供的访问密钥ID和密钥字符串指定AWS凭证。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
AwsCredentials.basic("accessKeyId", "secretAccessKey");
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
awsCredentials:
    type: basic
    accessKeyId: access-key-id
    secretAccessKey: secret-access-key
```
{% endtab %}
{% endtabs %}

### **Profile**

使用AWS配置配置文件以及配置文件的配置路径来指定AWS凭证。

{% tabs %}
{% tab title="嵌入式模块" %}
```text
AwsCredentials.profile("profile-name", "/path/to/profile/config");
```
{% endtab %}

{% tab title="远程模块" %}
```yaml
awsCredentials:
    type: basic
    profileName: profile-name
    profilePath: /path/to/profile/config
```
{% endtab %}
{% endtabs %}

