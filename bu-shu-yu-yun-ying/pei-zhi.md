# 配置

有状态函数包括少量特定于SDK的配置。这些都可以通过你工作的`flink-conf.yaml`来配置。

| 配置 | 默认 | 类型 | 描述 |
| :--- | :--- | :--- | :--- |
| **statefun.module.global-config。&lt;KEY&gt;** | \(none\) | String | 将给定的键/值对添加到“有状态功能”全局配置中。 |
| **statefun.message.serializer** | WITH\_PROTOBUF\_PAYLOADS | Message Serializer | 用于联机消息的序列化程序。配置有WITH\_PROTOBUF\_PAYLOADS，WITH\_KRYO\_PAYLOADS和WITH\_RAW\_PAYLOADS。 |
| **statefun.flink-job-name** | StatefulFunctions | String | 在Flink-UI中显示的名称。 |
| **statefun.feedback.memory.size** | 32 MB | Memory | 在反馈通道溢出到磁盘之前，在内存缓冲中使用的字节数。 |

