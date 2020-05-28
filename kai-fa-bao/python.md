# Python

有状态功能是应用程序的基础。它们是隔离，分布和持久性的原子单位。作为对象，它们封装单个实体（例如，特定的用户，设备或会话）的状态并编码其行为。有状态的功能可以通过消息传递相互之间以及与外部系统进行交互。Python SDK支持作为[远程模块](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/modules.html#remote-module)。

首先，将Python SDK添加为应用程序的依赖项。

```text
apache-flink-statefun==2.0.0
```

## 定义有状态功能

有状态函数是具有两个参数（上下文和消息）的任何函数。 该函数通过有状态函数修饰符绑定到运行时。 以下是一个简单的hello world函数的示例。

```python
from statefun import StatefulFunctions

functions = StatefulFunctions()

@functions.bind("example/hello")
def hello_function(context, message):
    """A simple hello world function"""
    user = User()
    message.Unpack(user)

    print("Hello " + user.name)
```

这段代码在名称空间示例中声明了一个类型为`hello`的函数，并将其绑定到`hello_function` Python实例。

消息是非类型化的，并以`google.protobuf.Any`的形式通过系统传递。任何一个函数都可以处理多种类型的消息。

上下文提供有关当前消息和函数的元数据，以及如何调用其他函数或外部系统。此页底部列出了上下文对象支持的所有方法的完整引用。

## 类型提示

如果函数有一组已知的受支持类型，可以将它们指定为类型提示。这包括支持多个输入消息类型的函数的联合类型。

```python
import typing
from statefun import StatefulFunctions

functions = StatefulFunctions()

@functions.bind("example/hello")
def hello_function(context, message: User):
    """A simple hello world function with typing"""

    print("Hello " + message.name)

@function.bind("example/goodbye")
def goodbye_function(context, message: typing.Union[User, Admin]):
    """A function that dispatches on types"""

    if isinstance(message, User):
        print("Goodbye user")
    elif isinstance(message, Admin):
        print("Goodbye Admin")
```

## 函数类型和消息传递

修饰符bind在运行时注册每一个函数到函数类型下。 函数类型必须采用`<namespace>/<name>`形式。然后可以从其他函数引用函数类型以创建地址并向特定实例发送消息。

```python
from google.protobuf.any_pb2 import Any
from statefun import StatefulFunctions

functions = StatefulFunctions()

@functions.bind("example/caller")
def caller_function(context, message):
    """A simple stateful function that sends a message to the user with id `user1`"""

    user = User()
    user.user_id = "user1"
    user.name = "Seth"

    envelope = Any()
    envelope.Pack(user)

    context.send("example/hello", user.user_id, envelope)
```

或者，可以将功能手动绑定到运行时。

```text
functions.register("example/caller", caller_function)
```

## 发送延迟的消息

函数能够延迟发送消息，以便它们会在一段时间后到达。函数甚至可以向自己发送延迟的消息，这些消息可以用作回调。延迟的消息是非阻塞的，因此功能将在发送和接收延迟的消息之间继续处理记录。 延迟是通过[Python timedelta](https://docs.python.org/3/library/datetime.html#datetime.timedelta)指定的。

```python
from google.protobuf.any_pb2 import Any
from statefun import StatefulFunctions

functions = StatefulFunctions()

@functions.bind("example/caller")
def caller_function(context, message):
    """A simple stateful function that sends a message to the user with id `user1`"""

    user = User()
    user.user_id = "user1"
    user.name = "Seth"

    envelope = Any()
    envelope.Pack(user)

    context.send("example/hello", user.user_id, envelope)
```

## 持久化

有状态函数将状态视为一等公民，因此所有有状态函数都可以轻松地定义状态，运行时可以自动地容错。所有有状态函数都可以仅通过在上下文对象中存储值来包含状态。数据总是作用于特定的函数类型和标识符。状态值可以是absent、None或google.protobu . any。

{% hint style="info" %}
**注意**：\[远程模块\]\([//ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/modules.html\#remote-module](//ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/modules.html#remote-module)）要求所有状态值都在module.yaml中注册.
{% endhint %}

以下是一个有状态功能，可根据用户被访问的次数向其打招呼。

```python
from google.protobuf.any_pb2 import Any
from statefun import StatefulFunctions

functions = StatefulFunctions()

@functions.bind("example/count")
def count_greeter(context, message):
    """Function that greets a user based on
    the number of times it has been called"""
    user = User()
    message.Unpack(user)


    state = context["count"]
    if state is None:
        state = Any()
        state.Pack(Count(1))
        output = generate_message(1, user)
    else:
        counter = Count()
        state.Unpack(counter)
        counter.value += 1
        output = generate_message(counter.value, user)
        state.Pack(counter)

    context["count"] = state
    print(output)

def generate_message(count, user):
    if count == 1:
        return "Hello " + user.name
    elif count == 2:
        return "Hello again!"
    elif count == 3:
        return "Third time's the charm"
    else:
        return "Hello for the " + count + "th time"
```

此外，可以通过删除持久值来清除持久值。

```text
del context["count"]
```

## 公开函数

Python SDK附带了一个`RequestReplyHandler`，它根据RESTful HTTP POSTS自动分派函数调用。`RequestReplyHandler`可以使用任何公开的HTTP框架。

### 使用Flask服务函数

[Flask](https://palletsprojects.com/p/flask/)是一种流行的Python Web框架。它可用于快速轻松地展示`RequestResponseHandler`。

```python
@app.route('/statefun', methods=['POST'])
def handle():
    response_data = handler(request.data)
    response = make_response(response_data)
    response.headers.set('Content-Type', 'application/octet-stream')
    return response


if __name__ == "__main__":
	app.run()
```

## 上下文参考

`context`传递给每个函数的对象具有以下属性/方法。

* **send**\(self, typename: str, id: str, message: Any\)
  * 向函数类型为/且消息类型为google.protobuf.Any的任何函数发送消息
* **pack\_and\_send**\(self, typename: str, id: str, message\)
  * 与上述相同，但会将`protobuf`消息打包到 `Any`
* **reply**\(self, message: Any\)
  * 发送消息到调用函数
* **pack\_and\_reply**\(self, message\)
  * 与上述相同，但会将`protobuf`消息打包到 `Any`
* **send\_after**\(self, delay: timedelta, typename: str, id: str, message: Any\)
  * 延迟后发送消息
* **pack\_and\_send\_after**\(self, delay: timedelta, typename: str, id: str, message\)
  * 与上述相同，但会将`protobuf`消息打包到 `Any`
* **send\_egress**\(self, typename, message: Any\)
  * 以`<namespace>/<name>`格式向出口发送消息
* **pack\_and\_send\_egress**\(self, typename, message\)
  * 与上述相同，但会将protobuf消息打包到 `Any`
* **getitem**\(self, name\)
  * 如果未设置值，则检索在该名称下注册为Any或None的状态
* **delitem**\(self, name\)
  * 删除在该名称下注册的状态
* **setitem**\(self, name, value: Any\)
  * 将给定名称下的值存储在state中。

