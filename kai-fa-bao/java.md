# Java

有状态功能是应用程序的基础。它们是隔离，分布和持久性的原子单位。作为对象，它们封装单个实体（例如，特定的用户，设备或会话）的状态并编码其行为。有状态的功能可以通过消息传递相互之间以及与外部系统进行交互。支持Java SDK作为[Embedded\_module](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/modules.html#embedded-module)。

首先，需要在你的应用程序中添加以下依赖项

```markup
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>statefun-sdk</artifactId>
	<version>2.0.0</version>
</dependency>
```

## 定义有状态函数

 有状态函数是实现`StatefulFunction`接口的任何类。以下是一个简单的hello world函数的示例。

```text
package org.apache.flink.statefun.docs;

import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.StatefulFunction;

public class FnHelloWorld implements StatefulFunction {

	@Override
	public void invoke(Context context, Object input) {
		System.out.println("Hello " + input.toString());
	}
}
```

函数通过其invoke方法处理每个传入消息。输入是非类型化的，并以java.lang.Object的形式在系统中传递，因此一个函数可以潜在地处理多种类型的消息。

.上下文提供关于当前消息和函数的元数据，以及如何调用其他函数或外部系统。函数是基于函数类型和唯一标识符来调用的。

### 有状态匹配函数

 有状态功能为处理事件和状态提供了强大的抽象，允许开发人员构建可以对任何类型的消息做出反应的组件。通常，函数仅需要处理一组已知的消息类型，并且`StatefulMatchFunction`接口提供了针对该问题的有效解决方案。

#### 匹配简单函数

有状态匹配函数是针对这种模式的有状态函数的一种自定义变体。开发人员概述了预期的类型，可选的谓词和类型明确的业务逻辑，然后让系统将每个输入分配给正确的操作。 变量绑定在一个configure方法内部，该方法在首次加载实例时执行。

```java
package org.apache.flink.statefun.docs.match;

import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.match.MatchBinder;
import org.apache.flink.statefun.sdk.match.StatefulMatchFunction;

public class FnMatchGreeter extends StatefulMatchFunction {

	@Override
	public void configure(MatchBinder binder) {
		binder
			.predicate(Customer.class, this::greetCustomer)
			.predicate(Employee.class, Employee::isManager, this::greetManager)
			.predicate(Employee.class, this::greetEmployee);
	}

	private void greetManager(Context context, Employee message) {
		System.out.println("Hello manager " + message.getEmployeeId());
	}

	private void greetEmployee(Context context, Employee message) {
		System.out.println("Hello employee " + message.getEmployeeId());
	}

	private void greetCustomer(Context context, Customer message) {
		System.out.println("Hello customer " + message.getName());
	}
}
```

#### 使你的函数完整

 与第一个示例类似，默认情况下，match函数是部分函数，​​它将在与任何分支都不匹配的任何输入上抛出`IllegalStateException`。可以通过提供一个`otherwise`子句来完成所有操作，以完成不匹配的输入，并将其视为Java switch语句中的默认子句，从而使它们变得完整。`otherwise`操作将其消息视为非类型化的`java.lang.Object`消息，允许你处理任何意外消息。

```java
package org.apache.flink.statefun.docs.match;

import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.match.MatchBinder;
import org.apache.flink.statefun.sdk.match.StatefulMatchFunction;

public class FnMatchGreeterWithCatchAll extends StatefulMatchFunction {

	@Override
	public void configure(MatchBinder binder) {
		binder
			.predicate(Customer.class, this::greetCustomer)
			.predicate(Employee.class, Employee::isManager, this::greetManager)
			.predicate(Employee.class, this::greetEmployee)
			.otherwise(this::catchAll);
	}

	private void catchAll(Context context, Object message) {
		System.out.println("Hello unexpected message");
	}

	private void greetManager(Context context, Employee message) {
		System.out.println("Hello manager");
	}

	private void greetEmployee(Context context, Employee message) {
		System.out.println("Hello employee");
	}

	private void greetCustomer(Context context, Customer message) {
		System.out.println("Hello customer");
	}
}
```

#### **Action Resolution Order**

匹配函数将始终使用以下解析规则匹配从最特定到最不特定的操作。

首先，找到一个匹配类型和谓词的操作。如果两个谓词对特定输入返回true，则首先在绑定器中注册的谓词将获胜。接下来，搜索与类型匹配但没有关联谓词的操作。最后，如果一个catch-all存在，它将被执行或者抛出一个IllegalStateException。

## 函数类型和消息传递

 在Java中，函数类型定义为包含名称空间和名称的_字符串_类型引用。该类型绑定到[模块](https://ci.apache.org/projects/flink/flink-statefun-docs-release-2.0/sdk/modules.html#embedded-module)定义中的实现类。下面是hello world函数的示例函数类型。

```text
package org.apache.flink.statefun.docs;

import org.apache.flink.statefun.sdk.FunctionType;

/** A function type that will be bound to {@link FnHelloWorld}. */
public class Identifiers {

  public static final FunctionType HELLO_TYPE = new FunctionType("apache/flink", "hello");
}
```

然后可以从其他函数中引用此类型，以创建地址并向特定实例发送消息。

```text
package org.apache.flink.statefun.docs;

import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.StatefulFunction;

/** A simple stateful function that sends a message to the user with id "user1" */
public class FnCaller implements StatefulFunction {

  @Override
  public void invoke(Context context, Object input) {
    context.send(Identifiers.HELLO_TYPE, "user1", new MyUserMessage());
  }
}
```

## 发送延迟的消息

函数能够延迟发送消息，以便它们会在一段时间后到达。函数甚至可以向自己发送延迟的消息，这些消息可以用作回调。延迟的消息是非阻塞的，因此功能将在发送和接收延迟的消息之间继续处理记录。

```text
package org.apache.flink.statefun.docs.delay;

import java.time.Duration;
import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.StatefulFunction;

public class FnDelayedMessage implements StatefulFunction {

	@Override
	public void invoke(Context context, Object input) {
		if (input instanceof Message) {
			System.out.println("Hello");
			context.sendAfter(Duration.ofMinutes(1), context.self(), new DelayedMessage());
		}

		if (input instanceof DelayedMessage) {
			System.out.println("Welcome to the future!");
		}
	}
}
```

## 完成异步请求

在与外部系统\(如数据库或API\)进行交互时，需要注意与外部系统的通信延迟不会主导应用程序的全部工作。有状态函数允许注册一个java CompletableFuture，它将在将来的某个时候解析为一个值。Future和一个元数据对象一起注册，该元数据对象提供关于调用者的附加上下文。

当将来完成时，无论是成功完成还是异常完成，调用方函数类型和id都将使用AsyncOperationResult调用。异步结果可以在以下三种状态之一完成:

### 成功

 异步操作已成功完成，可以通过`AsyncOperationResult#value`获得结果。

### 失败

 异步操作失败，可以通过`AsyncOperationResult#throwable`找到原因。

### 未知

 有状态功能在`CompletableFuture`完成之前已重新启动（可能在其他计算机上），因此未知异步操作的状态是什么。

```java
package org.apache.flink.statefun.docs.async;

import java.util.concurrent.CompletableFuture;
import org.apache.flink.statefun.sdk.AsyncOperationResult;
import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.StatefulFunction;

@SuppressWarnings("unchecked")
public class EnrichmentFunction implements StatefulFunction {

	private final QueryService client;

	public EnrichmentFunction(QueryService client) {
		this.client = client;
	}

	@Override
	public void invoke(Context context, Object input) {
		if (input instanceof User) {
			onUser(context, (User) input);
		} else if (input instanceof AsyncOperationResult) {
			onAsyncResult((AsyncOperationResult) input);
		}
	}

	private void onUser(Context context, User user) {
		CompletableFuture<UserEnrichment> future = client.getDataAsync(user.getUserId());
		context.registerAsyncOperation(user, future);
	}

	private void onAsyncResult(AsyncOperationResult<User, UserEnrichment> result) {
		if (result.successful()) {
			User metadata = result.metadata();
			UserEnrichment value = result.value();
			System.out.println(
				String.format("Successfully completed future: %s %s", metadata, value));
		} else if (result.failure()) {
			System.out.println(
				String.format("Something has gone terribly wrong %s", result.throwable()));
		} else {
			System.out.println("Not sure what happened, maybe retry");
		}
	}
}
```

## 持久化

有状态函数将状态视为头等公民，因此所有有状态函数都可以轻松定义状态，并在运行时自动将其设置为容错状态。通过仅定义一个或多个持久字段，所有有状态函数都可以包含状态。

最简单的入门方法是使用`PersistedValue`，它由其名称和所存储类型的类定义。数据始终限于特定的函数类型和标识符。以下是一个有状态函数，可根据用户被访问的次数向其打招呼。

{% hint style="info" %}
 **注意：**必须将所有**PersistedValue**，**PersistedTable**和**PersistedAppendingBuffer**字段标记为**@Persisted**批注，否则运行时将不使它们成为容错的。
{% endhint %}

```java
package org.apache.flink.statefun.docs;

import org.apache.flink.statefun.sdk.Context;
import org.apache.flink.statefun.sdk.FunctionType;
import org.apache.flink.statefun.sdk.StatefulFunction;
import org.apache.flink.statefun.sdk.annotations.Persisted;
import org.apache.flink.statefun.sdk.state.PersistedValue;

public class FnUserGreeter implements StatefulFunction {

	public static FunctionType TYPE = new FunctionType("example", "greeter");

	@Persisted
	private final PersistedValue<Integer> count = PersistedValue.of("count", Integer.class);

	public void invoke(Context context, Object input) {
		String userId = context.self().id();
		int seen = count.getOrDefault(0);

		switch (seen) {
			case 0:
				System.out.println(String.format("Hello %s!", userId));
				break;
			case 1:
				System.out.println("Hello Again!");
				break;
			case 2:
				System.out.println("Third time is the charm :)");
				break;
			default:
				System.out.println(String.format("Hello for the %d-th time", seen + 1));
		}

		count.set(seen + 1);
	}
}
```

持久化值附带了正确的基本方法来构建强大的有状态应用程序。调用`PersistedValue#get`将返回存储在state中的对象的当前值，如果未设置任何值，则返回null。相反，`PersistedValue#set`将更新state中的值，`PersistedValue#clear`将从state中删除该值。

### 集合类型

除了`PersistedValue`之外，Java SDK还支持两种持久化的集合类型。`PersistedTable`是键和值的集合，`PersistedAppendingBuffer`是一个只追加的缓冲区。

这些类型在功能上分别等同于`PersistedValue`和`PersistedValue`，但在某些情况下可能提供更好的性能。

```java
@Persisted
PersistedTable<String, Integer> table = PersistedTable.of("my-table", String.class, Integer.class);

@Persisted
PersistedAppendingBuffer<Integer> buffer = PersistedAppendingBuffer.of("my-buffer", Integer.class);
```

## 函数提供者和依赖注入

 有状态功能是在节点的分布式群集中创建的。 `StatefulFunctionProvider`是工厂类，用于在首次激活时创建有状态函数的新实例。

```java
package org.apache.flink.statefun.docs;

import org.apache.flink.statefun.docs.dependency.ProductionDependency;
import org.apache.flink.statefun.docs.dependency.RuntimeDependency;
import org.apache.flink.statefun.sdk.FunctionType;
import org.apache.flink.statefun.sdk.StatefulFunction;
import org.apache.flink.statefun.sdk.StatefulFunctionProvider;

public class CustomProvider implements StatefulFunctionProvider {

	public StatefulFunction functionOfType(FunctionType type) {
		RuntimeDependency dependency = new ProductionDependency();
		return new FnWithDependency(dependency);
	}
}
```

在每个并行工作程序中，每种类型的提供程序都调用一次，而不是每个id调用一次。如果有状态函数需要自定义配置，则可以在提供程序内部定义它们并将其传递给函数的构造函数。这也是创建共享物理资源（如数据库连接）的地方，这些资源可由任意数量的虚拟函数使用。现在，测试可以快速提供模拟或测试依赖项，而不需要复杂的依赖项注入框架。

```text
package org.apache.flink.statefun.docs;

import org.apache.flink.statefun.docs.dependency.RuntimeDependency;
import org.apache.flink.statefun.docs.dependency.TestDependency;
import org.junit.Assert;
import org.junit.Test;

public class FunctionTest {

	@Test
	public void testFunctionWithCustomDependency() {
		RuntimeDependency dependency = new TestDependency();
		FnWithDependency function = new FnWithDependency(dependency);

		Assert.assertEquals("It appears math is broken", 1 + 1, 2);
	}
}
```

