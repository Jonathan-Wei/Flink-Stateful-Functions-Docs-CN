# 打包部署

有状态功能应用程序可以打包为独立应用程序或可以提交到集群的Flink作业。

## 镜像

对于有状态功能应用程序，建议的部署模式是构建Docker镜像。这样，用户代码不需要打包任何Apache Flink组件。提供的基本镜像使团队可以快速将其应用程序打包为具有所有必需的运行时依赖项。

以下是一个示例Dockerfile，用于为一个名为`statefun-example`的应用程序同时构建一个包含[嵌入式模块](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/modules.html#embedded-module)和[远程模块](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/modules.html#remote-module)的有状态功能镜像。

```text
FROM flink-statefun:2.0.0

RUN mkdir -p /opt/statefun/modules/statefun-example
RUN mkdir -p /opt/statefun/modules/remote

COPY target/statefun-example*jar /opt/statefun/modules/statefun-example/
COPY module.yaml /opt/statefun/modules/remote/module.yaml
```

## Flink Jar

如果希望打包成作业提交给现有的Flink集群，只需将其`statefun-flink-distribution`作为对应用程序的依赖即可。

```markup
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>statefun-flink-distribution</artifactId>
	<version>2.0.0</version>
</dependency>
```

它包括所有有状态功能的运行时依赖项，并配置应用程序的主要入口点。

{% hint style="info" %}
 **注意：**该发行版必须捆绑在您的应用程序fat JAR中，以便于它在Flink的[用户代码类加载器上](https://ci.apache.org/projects/flink/flink-docs-stable/monitoring/debugging_classloading.html#inverted-class-loading-and-classloader-resolution-order)
{% endhint %}

```text
./bin/flink run -c org.apache.flink.statefun.flink.core.StatefulFunctionsJob ./statefun-example.jar
```

要运行StateFun应用程序，必须严格执行以下配置。

```text
classloader.parent-first-patterns.additional: org.apache.flink.statefun;org.apache.kafka;com.google.protobuf
```

