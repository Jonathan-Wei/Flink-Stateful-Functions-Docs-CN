# 模块（Module）

有状态函数应用程序由一个或多个模块组成。模块是一组函数，由运行时加载并可用于发送消息。所有加载模块的函数都是多路复用的，可以自由地相互发送消息。

有状态函数支持两种类型的模块:嵌入式模块和远程模块。

## 嵌入式模块

嵌入式模块与ApacheFlink®运行时共存并嵌入其中。

此模块类型仅支持基于JVM的语言，并通过实现`StatefulFunctionModule`接口定义。嵌入式模块提供了一种单一的配置方法，其中有状态函数根据其函数类型绑定到系统。运行时配置可通过`globalConfiguration`获得，`globalConfiguration`是应用程序`flink-config.yaml`中所有前缀为statefun.module.global-config配置的集合和以key value形式传递的任何命令行参数。

```java
package org.apache.flink.statefun.docs;

import java.util.Map;
import org.apache.flink.statefun.sdk.spi.StatefulFunctionModule;

public class BasicFunctionModule implements StatefulFunctionModule {

	public void configure(Map<String, String> globalConfiguration, Binder binder) {

		// Declare the user function and bind it to its type
		binder.bindFunctionProvider(FnWithDependency.TYPE, new CustomProvider());

		// Stateful functions that do not require any configuration
		// can declare their provider using java 8 lambda syntax
		binder.bindFunctionProvider(Identifiers.HELLO_TYPE, unused -> new FnHelloWorld());
	}
}
```

 嵌入式模块利用[Java的服务提供商接口（SPI）](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)进行发现。这意味着每个JAR应在`META_INF/services`资源目录中包含一个`org.apache.flink.statefun.sdk.spi.StatefulFunctionModule`文件，该文件列出了它提供的所有可用模块。

```text
org.apache.flink.statefun.docs.BasicFunctionModule
```

## 远程模块

远程模块从ApacheFlink®运行时作为外部进程运行；在同一个容器中，与小车或其他外部位置放置在一起。

此模块类型可以支持任意数量的语言SDK。远程模块通过`YAML`配置文件在系统中注册。

### 规范

 远程模块配置由一个`meta`部分和一个`spec`部分组成。 `meta`包含有关模块的辅助信息。`spec`描述模块中包含的功能并定义其持久值。

### 定义函数

 `module.spec.functions`声明`function`由远程模块实现的对象的列表。函数通过许多属性来描述的。

* `function.meta.kind`
  * 与远程功能进行通信的协议。
  * 支持的值- `http`
* `function.meta.type`
  * 函数类型，定义为`<namespace>/<name>`。
* `function.spec.endpoint`
  * 该功能可以到达的端点。
* `function.spec.states`
  * 远程功能中已保留的持久值名称的列表。
* `function.spec.maxNumBatchRequests`
  * `address`在调用系统上的背压之前，特定功能可以处理的最大记录数。
  * 默认值-1000
* `function.spec.timeout`
  * 运行时在失败之前等待远程功能返回的最长时间。
  * 默认-1分钟

### 完整例子

```text
version: "1.0"

module:
  meta:
    type: remote
  spec:
    functions:
      - function:
        meta:
          kind: http
          type: example/greeter
        spec:
          endpoint: http://<host-name>/statefun
          states:
            - seen_count
          maxNumBatchRequests: 500
          timeout: 2min
```

