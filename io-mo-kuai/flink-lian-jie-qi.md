# Flink连接器

 Source-Sink I/O模块允许插入尚未集成到专用I/O模块中的现有或自定义Flink连接器。有关如何构建自定义连接器的详细信息，请参见正式的[Apache Flink文档](https://ci.apache.org/projects/flink/flink-docs-stable)。

## 依赖项

要使用自定义Flink连接器，请在pom中添加以下依赖项。

```markup
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>statefun-flink-io</artifactId>
    <version>2.0.0</version>
    <scope>provided</scope>
</dependency>
```

## Source规范

源函数规范从Flink源函数创建一个入口。

```java
package org.apache.flink.statefun.docs.io.flink;

import java.util.Map;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.flink.io.datastream.SourceFunctionSpec;
import org.apache.flink.statefun.sdk.io.IngressIdentifier;
import org.apache.flink.statefun.sdk.io.IngressSpec;
import org.apache.flink.statefun.sdk.spi.StatefulFunctionModule;

public class ModuleWithSourceSpec implements StatefulFunctionModule {

    @Override
    public void configure(Map<String, String> globalConfiguration, Binder binder) {
        IngressIdentifier<User> id = new IngressIdentifier<>(User.class, "example", "users");
        IngressSpec<User> spec = new SourceFunctionSpec<>(id, new FlinkSource<>());
        binder.bindIngress(spec);
    }
}
```

## Sink规范

Sink函数规范从Flink Sink函数创建出口。

```java
package org.apache.flink.statefun.docs.io.flink;

import java.util.Map;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.flink.io.datastream.SinkFunctionSpec;
import org.apache.flink.statefun.sdk.io.EgressIdentifier;
import org.apache.flink.statefun.sdk.io.EgressSpec;
import org.apache.flink.statefun.sdk.spi.StatefulFunctionModule;

public class ModuleWithSinkSpec implements StatefulFunctionModule {

    @Override
    public void configure(Map<String, String> globalConfiguration, Binder binder) {
        EgressIdentifier<User> id = new EgressIdentifier<>("example", "user", User.class);
        EgressSpec<User> spec = new SinkFunctionSpec<>(id, new FlinkSink<>());
        binder.bindEgress(spec);
    }
}
```

