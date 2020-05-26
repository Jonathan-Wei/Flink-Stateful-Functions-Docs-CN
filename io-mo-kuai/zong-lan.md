# 总览

状态函数的I / O模块允许函数接收消息并将消息发送到外部系统。基于Ingress（输入）和Egress（输出）点的概念，并基于ApacheFlink®连接器生态系统，I / O模块使函数可以通过消息传递的方式与外界进行交互。

## 入口

入口是一个输入点，数据从外部系统消耗并转发到零个或多个函数。它是通过入口标识符和入口spec定义的。

入口标识符，类似于函数类型，通过指定其输入类型、命名空间和名称来唯一地标识入口。

规范定义了如何连接到外部系统的详细信息，该系统特定于每个单独的I/O模块。每个标识符规范对都绑定到有状态函数模块内的系统。

{% tabs %}
{% tab title=" 嵌套模块" %}
```java
package org.apache.flink.statefun.docs.io.ingress;

import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.IngressIdentifier;

public final class Identifiers {

    public static final IngressIdentifier<User> INGRESS =
        new IngressIdentifier<>(User.class, "example", "user-ingress");
}
```

```java
package org.apache.flink.statefun.docs.io.ingress;

import java.util.Map;
import org.apache.flink.statefun.docs.io.MissingImplementationException;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.IngressIdentifier;
import org.apache.flink.statefun.sdk.io.IngressSpec;
import org.apache.flink.statefun.sdk.spi.StatefulFunctionModule;

public class ModuleWithIngress implements StatefulFunctionModule {

    @Override
    public void configure(Map<String, String> globalConfiguration, Binder binder) {
        IngressSpec<User> spec = createIngress(Identifiers.INGRESS);
        binder.bindIngress(spec);
    }

    private IngressSpec<User> createIngress(IngressIdentifier<User> identifier) {
        throw new MissingImplementationException("Replace with your specific ingress");
    }
}
```
{% endtab %}

{% tab title="远程模块" %}
```text
version: "1.0"

module:
     meta:
         type: remote
     spec:
         ingresses:
           - ingress:
               meta:
                 id: example/user-ingress
                 type: # ingress type
               spec: # ingress specific configurations
```
{% endtab %}
{% endtabs %}

## 路由器

路由器是一个无状态的操作符，它从入口获取每个记录并将其路由到零个或多个函数。路由器通过有状态函数模块绑定到系统，与其他组件不同，入口可以有任意数量的路由器。

{% tabs %}
{% tab title="嵌套模块" %}
```text
package org.apache.flink.statefun.docs.io.ingress;

import org.apache.flink.statefun.docs.FnUser;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.Router;

public class UserRouter implements Router<User> {

    @Override
    public void route(User message, Downstream<User> downstream) {
        downstream.forward(FnUser.TYPE, message.getUserId(), message);
    }
}
```

```text
package org.apache.flink.statefun.docs.io.ingress;

import java.util.Map;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.IngressIdentifier;
import org.apache.flink.statefun.sdk.io.IngressSpec;
import org.apache.flink.statefun.sdk.io.Router;
import org.apache.flink.statefun.sdk.spi.StatefulFunctionModule;

public class ModuleWithRouter implements StatefulFunctionModule {

    @Override
    public void configure(Map<String, String> globalConfiguration, Binder binder) {
        IngressSpec<User> spec = createIngressSpec(Identifiers.INGRESS);
        Router<User> router = new UserRouter();

        binder.bindIngress(spec);
        binder.bindIngressRouter(Identifiers.INGRESS, router);
    }

    private IngressSpec<User> createIngressSpec(IngressIdentifier<User> identifier) {
        throw new MissingImplementationException("Replace with your specific ingress");
    }
}
```
{% endtab %}

{% tab title="远程模块" %}
在yaml中定义时，路由器由函数类型列表定义。`id`地址的组成部分是从与其基础源实现中的每个记录关联的键中提取的。

```text
targets:
    - example-namespace/my-function-1
    - example-namespace/my-function-2
```
{% endtab %}
{% endtabs %}

## 出口

出口与入口相反；它是一个接收消息并将其写入外部系统的点。每个出口使用两个组件定义，一个出口标识器和一个出口规范。

出口标识符基于命名空间、名称和生成类型唯一地标识出口。出口规范定义了如何连接到外部系统的详细信息，这些详细信息特定于每个单独的I/O模块。每个标识符规范对都绑定到有状态函数模块内的系统。

{% tabs %}
{% tab title="嵌套模块" %}
```java
package org.apache.flink.statefun.docs.io.egress;

import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.EgressIdentifier;

public final class Identifiers {

    public static final EgressIdentifier<User> EGRESS =
        new EgressIdentifier<>("example", "egress", User.class);
}
```

```java
package org.apache.flink.statefun.docs.io.egress;

import java.util.Map;
import org.apache.flink.statefun.docs.io.MissingImplementationException;
import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.io.EgressIdentifier;
import org.apache.flink.statefun.sdk.io.EgressSpec;
import org.apache.flink.statefun.sdk.spi.StatefulFunctionModule;

public class ModuleWithEgress implements StatefulFunctionModule {

    @Override
    public void configure(Map<String, String> globalConfiguration, Binder binder) {
        EgressSpec<User> spec = createEgress(Identifiers.EGRESS);
        binder.bindEgress(spec);
    }

    public EgressSpec<User> createEgress(EgressIdentifier<User> identifier) {
        throw new MissingImplementationException("Replace with your specific egress");
    }
}
```

然后，有状态函数可以使用与向另一个函数发送消息相同的方式向出口发送消息，并将出口标识符作为函数类型传递。

```java
package org.apache.flink.statefun.docs.io.egress;

import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.StatefulFunction;

/** A simple function that outputs messages to an egress. */
public class FnOutputting implements StatefulFunction {

    @Override
    public void invoke(Context context, Object input) {
        context.send(Identifiers.EGRESS, new User());
    }
}
```
{% endtab %}

{% tab title="远程模块" %}
```text
version: "1.0"

module:
    meta:
        type: remote
    spec:
        egresses:
          - egress:
              meta:
                id: example/user-egress
                type: # egress type
              spec: # egress specific configurations
```

然后，有状态函数可以使用与向另一个函数发送消息相同的方式向出口发送消息，并将出口标识符作为函数类型传递。

```java
package org.apache.flink.statefun.docs.io.egress;

import org.apache.flink.statefun.docs.models.User;
import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.StatefulFunction;

/** A simple function that outputs messages to an egress. */
public class FnOutputting implements StatefulFunction {

    @Override
    public void invoke(Context context, Object input) {
        context.send(Identifiers.EGRESS, new User());
    }
}
```
{% endtab %}
{% endtabs %}

