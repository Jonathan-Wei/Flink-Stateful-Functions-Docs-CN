# Python演练

有状态功能提供了一个用于构建健壮的，有状态事件驱动的应用程序的平台。它提供了对状态和时间的细粒度控制，从而可以构建高级系统。在本分步指南中，您将学习如何使用Stateful Functions API构建有状态的应用程序。

你在构建什么？

先决条件

救命，我被卡住了！

如何遵循

从事件开始

我们的首要职能

发松回应

有状态的Hello



完整应用

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



