# Python演练

有状态功能提供了一个用于构建健壮的，有状态事件驱动的应用程序的平台。它提供了对状态和时间的细粒度控制，从而可以构建高级系统。在本分步指南中，您将学习如何使用Stateful Functions API构建有状态的应用程序。

## 你在构建什么？

像所有出色的软件介绍一样，本演练将从头开始进行介绍。应用程序将运行一个简单的函数，函数接受请求并以问候语响应。它不会尝试涵盖应用程序开发的所有复杂性，而将重点放在构建有状态功能上-你将在其中实现业务逻辑。

## 先决条件

本演练假定你对Python有一定的了解，但是即使你会使用其他编程语言，也应该能够继续学习。

## 救命，我被卡住了！

 如果遇到困难，请查看[社区支持资源](https://flink.apache.org/gettinghelp.html)。特别是，Apache Flink的[用户邮件列表](https://flink.apache.org/community.html#mailing-lists)一直被评为所有Apache项目中最活跃的[邮件列表](https://flink.apache.org/community.html#mailing-lists)之一，并且是快速获得帮助的好方法。

## 如何遵循

 如果要继续学习，则需要一台具有[Python 3](https://www.python.org/)和[Docker](https://www.docker.com/)的计算机。

{% hint style="info" %}
 **注意：**为简便起见，本演练中的每个代码块可能不包含完整的类。完整的代码位于[本页底部](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/getting-started/python_walkthrough.html#full-application)。
{% endhint %}

你可以通过单击[此处](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/downloads/walkthrough.zip)下载带有框架项目的zip文件。

解压缩软件包后，你会发现许多文件。其中包括dockerfiles和数据生成器，可在本地自包含环境中运行此演练。

```text
$ tree statefun-walkthrough
statefun-walkthrough
├── Dockerfile
├── docker-compose.yml
├── generator
│   ├── Dockerfile
│   ├── event-generator.py
│   └── messages_pb2.py
├── greeter
│   ├── Dockerfile
│   ├── greeter.py
│   ├── messages.proto
│   ├── messages_pb2.py
│   └── requirements.txt
└── module.yaml
```

## 从事件开始

有状态函数是一个事件驱动的系统，因此开发从定义事件开始。Greet应用程序将使用协议缓冲区定义其事件。当接收到特定用户的问候请求时，它将被路由到相应的函数，返回响应的问题语。 第三种类型`SeenCount`是实用程序类，以后将用于帮助管理到目前为止用户被看到的次数。

```python
syntax = "proto3";

package example;

// External request sent by a user who wants to be greeted
message GreetRequest {
    // The name of the user to greet
    string name = 1;
}
// A customized response sent to the user
message GreetResponse {
    // The name of the user being greeted
    string name = 1;
    // The users customized greeting
    string greeting = 2;
}
// An internal message used to store state
message SeenCount {
    // The number of times a users has been seen so far
    int64 seen = 1;
}
```

## 我们的首要职能

在底层，消息是使用有状态函数处理的，它是绑定到StatefulFunction运行时的任意两个参数函数。函数通过@function绑定到运行时。绑定装饰。当绑定一个函数时，它被注释为一个函数类型。这是在向该函数发送消息时用来引用它的名称。

当你打开文件greeter/greeter.py时，你应该会看到以下代码。

```java
from statefun import StatefulFunctions

functions = StatefulFunctions()

@functions.bind("example/greeter")
def greet(context, greet_request):
    pass
```

有状态函数接受两个参数，一个上下文和一条消息。上下文提供对有状态函数运行时特性\(如状态管理和消息传递\)的访问。在本演练中，您将探索其中的一些特性。

另一个参数是传递给这个函数的输入消息。默认情况下，消息作为protobuf Any传递。如果函数只接受已知类型，则可以使用python3类型语法覆盖消息类型。这样就不需要打开消息或检查类型。

```python
from messages_pb2 import GreetRequest
from statefun import StatefulFunctions

functions = StatefulFunctions()

@functions.bind("example/greeter")
def greet(context, greet_request: GreetRequest):
    pass
```

## 发送一个响应

有状态功能接受消息，也可以将其发送出去。消息可以发送到其他函数，也可以发送到外部系统（或[egress](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/io-module/index.html#egress)）。

比较常用 的外部系统是[Apache Kafka](http://kafka.apache.org/)。首先，让我们更新`greeter/ greter .py`中的函数，通过向Kafka Topic发送问候语来响应每个输入。

```python
from messages_pb2 import GreetRequest, GreetResponse
from statefun import StatefulFunctions

functions = StatefulFunctions()

@functions.bind("example/greeter")
def greet(context, message: GreetRequest):
    response = GreetResponse()
    response.name = message.name
    response.greeting = "Hello {}".format(message.name)
    
    egress_message = kafka_egress_record(topic="greetings", key=message.name, value=response)
    context.pack_and_send_egress("example/greets", egress_message)
```

对于每个消息，将构造响应并将其发送到按名称分区的kafka主题 `greetings` 。egress\_message被发送到一个名为example/ greeting的出口。这个标识符指向一个特定的Kafka集群，并在下面的部署中配置。

## 有状态的Hello

这是一个很好的开始，但并没有显示出有状态函数的真正威力——使用状态。假设您希望根据每个用户发送请求的次数为其生成个性化响应。

```text
def compute_greeting(name, seen):
    """
    Compute a personalized greeting, based on the number of times this @name had been seen before.
    """
    templates = ["", "Welcome %s", "Nice to see you again %s", "Third time is a charm %s"]
    if seen < len(templates):
        greeting = templates[seen] % name
    else:
        greeting = "Nice to see you at the %d-nth time %s!" % (seen, name)

    response = GreetResponse()
    response.name = name
    response.greeting = greeting

    return response
```

 为了“记住”多条问候消息中的信息，你需要将一个持久值字段（`seen_count`）与Greet函数进行关联。对于每个用户，函数现在可以跟踪他们被查看过多少次。

```python
@functions.bind("example/greeter")
def greet(context, greet_message: GreetRequest):
    state = context.state('seen_count').unpack(SeenCount)
    if not state:
        state = SeenCount()
        state.seen = 1
    else:
        state.seen += 1
    context.state('seen_count').pack(state)

    response = compute_greeting(greet_request.name, state.seen)

    egress_message = kafka_egress_record(topic="greetings", key=greet_request.name, value=response)
    context.pack_and_send_egress("example/greets", egress_message)
```

 状态`seen_count`始终是当前名称的范围，因此它可以独立跟踪每个用户。

## 把它们连在一起

有状态函数应用程序使用http与Apache Flink运行时通信。Python SDK附带了一个`RequestReplyHandler`，它根据RESTful HTTP POSTS自动分派函数调用。`RequestReplyHandler`可以使用任何HTTP框架公开。

一个流行的Python web框架是Flask。它可以用于快速、轻松地将应用程序公开到Apache Flink运行时。

```python
from statefun import StatefulFunctions
from statefun import RequestReplyHandler

functions = StatefulFunctions()

@functions.bind("walkthrough/greeter")
def greeter(context, message: GreetRequest):
    pass

handler = RequestReplyHandler(functions)

# Serve the endpoint

from flask import request
from flask import make_response
from flask import Flask

app = Flask(__name__)

@app.route('/statefun', methods=['POST'])
def handle():
    response_data = handler(request.data)
    response = make_response(response_data)
    response.headers.set('Content-Type', 'application/octet-stream')
    return response


if __name__ == "__main__":
    app.run()
```

## 配置运行

有状态函数运行时通过对Flask服务器进行http调用向greeter函数发出请求。要做到这一点，它需要知道可以使用什么端点来访问服务器。这也是配置到输入和输出Kafka主题的连接的好时机。配置位于module.yaml的文件中。

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
            endpoint: http://python-worker:8000/statefun
            states:
              - seen_count
            maxNumBatchRequests: 500
            timeout: 2min
    ingresses:
      - ingress:
          meta:
            type: statefun.kafka.io/routable-protobuf-ingress
            id: example/names
          spec:
            address: kafka-broker:9092
            consumerGroupId: my-group-id
            topics:
              - topic: names
                typeUrl: com.googleapis/example.GreetRequest
                targets:
                  - example/greeter
    egresses:
      - egress:
          meta:
            type: statefun.kafka.io/generic-egress
            id: example/greets
          spec:
            address: kafka-broker:9092
            deliverySemantic:
              type: exactly-once
              transactionTimeoutMillis: 100000
```

此配置做了一些有趣的事情。

首先是声明我们的函数`example/greeter`。它包括可访问的端点以及该功能可以访问的状态。

入口是输入Kafka主题，它将`GreetRequest`消息路由到该函数。除基本属性（如经纪人地址和消费者组）外，它还包含目标列表。这些是每条消息将发送到的功能。

出口是输出Kafka群集。它包含特定于代理的配置，但允许每条消息路由到任何主题。

## 部署

既然已经构建了greeter应用程序，现在就该进行部署了。部署有状态函数应用程序的最简单方法是使用社区提供的基本镜像并加载模块。 基本镜像提供了有状态功能运行时，它将使用提供`module.yaml`的配置来为此特定作业进行配置。可以`Dockerfile`在根目录中找到。

```text
FROM flink-statefun:2.0.0

RUN mkdir -p /opt/statefun/modules/greeter
ADD module.yaml /opt/statefun/modules/greeter
```

现在，您可以使用提供的Docker设置在本地运行此应用程序。

```text
$ docker-compose up -d
```

然后，要查看操作中的示例，请参阅本主题的内容`greetings`：

```text
docker-compose logs -f event-generator 
```

## 想走得更远吗？

这个Greeter从不忘记用户。尝试修改该函数，对于花费超过60秒而没有与系统进行交互的用户重置其 `seen_count` 

## 完整应用

```java
from messages_pb2 import SeenCount, GreetRequest, GreetResponse

from statefun import StatefulFunctions
from statefun import RequestReplyHandler
from statefun import kafka_egress_record

functions = StatefulFunctions()

@functions.bind("example/greeter")
def greet(context, greet_request: GreetRequest):
    state = context.state('seen_count').unpack(SeenCount)
    if not state:
        state = SeenCount()
        state.seen = 1
    else:
        state.seen += 1
    context.state('seen_count').pack(state)

    response = compute_greeting(greet_request.name, state.seen)

    egress_message = kafka_egress_record(topic="greetings", key=greet_request.name, value=response)
    context.pack_and_send_egress("example/greets", egress_message)


def compute_greeting(name, seen):
    """
    Compute a personalized greeting, based on the number of times this @name had been seen before.
    """
    templates = ["", "Welcome %s", "Nice to see you again %s", "Third time is a charm %s"]
    if seen < len(templates):
        greeting = templates[seen] % name
    else:
        greeting = "Nice to see you at the %d-nth time %s!" % (seen, name)

    response = GreetResponse()
    response.name = name
    response.greeting = greeting

    return response


handler = RequestReplyHandler(functions)

#
# Serve the endpoint
#

from flask import request
from flask import make_response
from flask import Flask

app = Flask(__name__)


@app.route('/statefun', methods=['POST'])
def handle():
    response_data = handler(request.data)
    response = make_response(response_data)
    response.headers.set('Content-Type', 'application/octet-stream')
    return response


if __name__ == "__main__":
    app.run()
```



