# 状态引导

通常情况下，应用程序需要文件、数据库或其他系统中的历史数据提供的初始状态。因为状态是由Apache Flink的快照机制管理的，对于有状态函数应用程序，这意味着将初始状态写入一个可用于启动作业的保存点。用户可以使用Flink的 [State Processor API](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/libs/state_processor_api.html)和StatefulFunctionSavepointCreator来引导有状态函数应用程序的初始状态。

首先，在你的应用程序中需要包含以下库：

```markup
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>statefun-flink-state-processor</artifactId>
  <version>2.0.0</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-state-processor-api_2.11</artifactId>
  <version>1.10.0</version>
</dependency>
```

{% hint style="info" %}
 **注意：**保存点创建器当前仅支持初始化Java模块的状态。
{% endhint %}

## 状态引导函数

StateBootstrapFunction定义如何使用给定输入引导StatefulFunction实例的状态。

每个引导函数实例直接对应于一个`StatefulFunction`类型。同样，每个实例都由一个地址唯一标识，该地址由正在引导的函数的类型和ID表示。引导功能实例保留的任何状态将对`StatefulFunction`具有相同地址的相应活动实例可用。

例如，考虑以下状态引导程序功能：

```java
public class MyStateBootstrapFunction implements StateBootstrapFunction {

	@Persisted
	private PersistedValue<MyState> state = PersistedValue.of("my-state", MyState.class);

	@Override
	public void bootstrap(Context context, Object input) {
		state.set(extractStateFromInput(input));
	}
 }
```

## 创建一个保存点

通过定义某些元数据（例如最大并行度和状态后端）来创建保存点。默认状态后端是[RocksDB](https://ci.apache.org/projects/flink/flink-docs-stable/ops/state/state_backends.html#the-rocksdbstatebackend)。

```text
int maxParallelism = 128;
StatefulFunctionsSavepointCreator newSavepoint = new StatefulFunctionsSavepointCreator(maxParallelism);
```

每个输入数据集都通过[路由器](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/io-module/index.html#router)在保存点创建[器](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/io-module/index.html#router)中注册，该[路由器](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/io-module/index.html#router)将每个记录路由到零个或多个功能实例。然后，可以向保存点创建者注册任意数量的函数类型，类似于在有状态函数模块中注册函数的方式。最后，为生成的保存点指定输出位置。

```text
// Read data from a file, database, or other location
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

final DataSet<Tuple2<String, Integer>> userSeenCounts = env.fromElements(
	Tuple2.of("foo", 4), Tuple2.of("bar", 3), Tuple2.of("joe", 2));

// Register the dataset with a router
newSavepoint.withBootstrapData(userSeenCounts, MyStateBootstrapFunctionRouter::new);

// Register a bootstrap function to process the records
newSavepoint.withStateBootstrapFunctionProvider(
		new FunctionType("apache", "my-function"),
		ignored -> new MyStateBootstrapFunction());

newSavepoint.write("file:///savepoint/path/");

env.execute();
```

有关如何使用Flink `DataSet`API的完整详细信息，请查看官方[文档](https://ci.apache.org/projects/flink/flink-docs-stable/dev/batch/)。

## 部署方式

创建新的savpepoint之后，可以使用它来为有状态功能应用程序提供初始状态。

{% tabs %}
{% tab title="镜像部署" %}
在基于镜像进行部署时，请将`-s`命令传递给Flink [JobMaster](https://ci.apache.org/projects/flink/flink-docs-stable/concepts/glossary.html#flink-master)镜像。

```text
version: "2.1"
services:
  master:
    image: my-statefun-application-image
    command: -s file:///savepoint/path
```
{% endtab %}

{% tab title="会话部署" %}
部署到Flink会话群集时，请在Flink CLI中指定savepoint参数。

```text
$ ./bin/flink run -s file:///savepoint/path stateful-functions-job.jar
```
{% endtab %}
{% endtabs %}

