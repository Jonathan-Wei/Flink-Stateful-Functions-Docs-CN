# 指标

有状态函数包括许多特定于SDK的指标。与标准度量范围一样，有状态函数支持操作符范围之下一级的函数范围。

metrics.scope.function

* 默认值：`<host> .taskmanager.<tm_id>.<job_name>.<operator_name>.<subtask_index>.<function_namespace>.<function_name>`
* 应用于作用域范围内的所有度量。

| 指标 | 范围 | 描述 | 类型 |
| :--- | :--- | :--- | :--- |
| **in** | 函数 | 传入消息的数量。 | 计数器 |
| **inRate** | 函数 | 每秒平均传入消息数。 | 仪表 |
| **out-local** | 函数 | 发送到同一任务槽上的函数的消息数。 | 计数器 |
| **out-localRate** | 函数 | 每秒在同一任务插槽上发送给函数的消息的平均数量。 | 仪表 |
| **out-remote** | 函数 | 发送到另一个任务插槽上的函数的消息数。 | 计数器 |
| **out-remoteRate** | 函数 | 每秒在不同任务插槽上发送给函数的消息的平均数量。 | 仪表 |
| **out-egress** | 函数 | 发送到出口的消息数。 | 计数器 |
| **feedback.produced** | 函数 | 从反馈通道读取的消息数。 | 仪表 |
| **feedback.producedRate** | 函数 | 每秒从反馈通道读取的消息的平均数量。 | 仪表 |

