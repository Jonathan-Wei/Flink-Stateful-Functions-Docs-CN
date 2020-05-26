# 分布式架构

有状态函数部署由几个相互作用的组件组成。在这里，我们将描述这些组件及其相互之间的关系以及Apache Flink运行时。

## 高级视图

有状态函数部署由一组Apache Flink有状态函数进程和执行远程函数的各种部署\(可选\)组成。

![](../.gitbook/assets/image%20%288%29.png)

Flink工作进程（TaskManagers）从入口系统（Kafka，Kinesis等）接收事件，并将其路由到目标函数。它们调用函数并将结果消息路由到下一个各自的目标功能。指定用于出口的消息将被写入出口系统（再次，Kafka，Kinesis等）。

## 组件

繁重的工作由Apache Flink进程完成，该进程管理状态，处理消息传递并调用有状态功能。Flink群集通常由一个主服务器和多个工作器（TaskManagers）组成。

![](../.gitbook/assets/image%20%287%29.png)

 除了Apache Flink流程之外，完整部署还需要[ZooKeeper](https://zookeeper.apache.org/)（用于[主故障转移](https://ci.apache.org/projects/flink/flink-docs-stable/ops/jobmanager_high_availability.html)）和大容量存储（S3，HDFS，NAS，GCS，Azure Blob存储等）来存储Flink的[检查点](https://ci.apache.org/projects/flink/flink-docs-master/concepts/stateful-stream-processing.html#checkpointing)。反过来，部署不需要数据库，Flink进程也不需要持久卷。

## 逻辑协同，物理分离

## 函数的部署方式

### 远程函数

![](../.gitbook/assets/image%20%282%29.png)

### 协同函数

![](../.gitbook/assets/image%20%2811%29.png)

 有关详细信息，请参阅[Python SDK](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/python.html)和[远程模块](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/modules.html#remote-module)上的文档。

### 嵌入式函数

![](../.gitbook/assets/image%20%289%29.png)

