# 异步拦截

本文讨论了几种与异步有关的拦截方案。 我们将按实现难度递增的顺序查看这些方案：

 * 在拦截器代码中使用或不使用`async`/`await`来拦截不会产生返回值的可等待方法（例如`Task`）

 * 在拦截器代码中使用`async`/`await`来拦截产生返回值的可等待方法（例如`Task<TResult>`）

 * 将`invocation.Proceed()`与`async`/`await`结合使用

本文包含使用`async`/`await`关键字的C＃代码示例。 在继续之前，请确保您对这些工作原理有很好的了解。 它们只是“语法糖”，将使.NET编译器把异步方法重写为状态机。文章 [Async/Await FAQ by Stephen Toub](https://devblogs.microsoft.com/pfxteam/asyncawait-faq/) 更加详细地解释了这些关键字。

方便起见，本文中显示的示例将重点介绍拦截器的`Intercept`方法。 假定代码示例由以下代码为基础：

```csharp
var serviceProxy = proxyGenerator.CreateInterfaceProxyWithoutTarget<IService>(new AsyncInterceptor());

// 示例将显示如何触发拦截：
int result = await serviceProxy.GetAsync();

public interface IService
{
	Task DoAsync();
	Task<int> GetAsync(int n);
}

class AsyncInterceptor : IInterceptor
{
	// 示例将主要集中于此方法的实现：
	public void Intercept(IInvocation invocation)
	{
		...
	}
}
```

## 拦截不会产生返回值的可等待方法

拦截可等待的方法（例如，返回类型为`任务`的方法）非常容易，与拦截一般的方法没有太大区别。 您只需要将拦截的调用的返回值设置为任何有效的`Task`对象即可。

```csharp
// calling code:
await serviceProxy.DoAsync();

// interception:
public void Intercept(IInvocation invocation)
{
	invocation.ReturnValue = httpClient.PostAsync(...);
}
```

那不是很有趣，在现实世界中，你会很快就会想要使用`async`/`await`。然而，这也相当微不足道：


```csharp
// calling code:
await serviceProxy.DoAsync();

// interception:
public void Intercept(IInvocation invocation)
{
	invocation.ReturnValue = InterceptAsync(invocation);
}

private async Task InterceptAsync(IInvocation invocation)
{
	// In this method, you have the comfort of using `await`:
	var response = await httpClient.PostAsync(...);
	...
}
```

一旦你想向调用者返回一个值，事情就变得更复杂了。让我们再看看这个！

## 拦截产生返回值的可等待方法

当拦截产生返回值的可等待方法（例如返回类型为`Task<TResult>`的方法）时，重要的是要记住，被执行的第一个`await`立即返回给调用者。（C#编译器将把`await`后面的语句移到一个continuation中，该continuation将在`await完成`后执行。）

换句话说，拦截器中的第一个`await`就完成了代理方法的拦截！

DynamicProxy要求拦截器（或代理目标对象）为拦截的`非void`方法提供返回值。当然，在异步场景中也有同样的要求。因为第一个`await`会提前返回到调用者，所以必须确保在任何`await`之前设置拦截调用的返回值。

我们已经在上一节结束时做过了；让我们再快速并仔细查看一下：


```csharp
// calling code:
var result = await serviceProxy.DoAsync();

// interception:
public void Intercept(IInvocation invocation)
{
	invocation.ReturnValue = InterceptAsync(invocation);
	//         ^^^^^^^^^^^^^
}

private async Task InterceptAsync(IInvocation invocation)
{
	// At this point, interception is still going on
	// as we haven't yet hit upon an `await`.

	var response = await httpClient.PostAsync(...);

	// At this point, interception has already completed!

	invocation.ReturnValue = ...;
	// 在此设置的返回值将对调用者不可见
	// ^ Any assignments to `invocation.ReturnValue` are no longer
	//   observable by the calling code, which already received its
	//   return value earlier (specifically, the `Task` produced by
	//   the C# compiler representing this asynchronous method).
	//   `invocation` is essentially a "cold", "detached", "stale",
	//   or "dead" object (pick your favourite term).
}
```

那么，当截获方法的返回类型不仅仅是`Task`，而是`Task<int>`时，我们如何将返回值传递回调用代码呢？以下示例显示如何通过调用代码来控制接收到的任务的结果：

```csharp
// calling code:
var result = await serviceProxy.GetAsync(41);
Assert.Equal(42, result);

// interception:
public void Intercept(IInvocation invocation)
{
	invocation.ReturnValue = InterceptAsync(invocation);
}

private async Task<int> InterceptAsync(IInvocation invocation)
{
	// We can still use `await`:
	await ...;

	// And we can simply `return` values, and the C# compiler
	// will do the rest for us:
	return (int)invocation.Arguments[0] + 1;
}
```

不幸的是，在现实世界中，情况通常并非那么简单，在这种情况下，您不能假设拦截器处理的每个方法都具有与`Task<int>`完全相同的返回类型。 因此，除了只能从`async Task<int>`方法中`return someInt`;之外，你还必须求助于非通用的`Task`，`TaskCompletionSource`和一些反射：

```csharp
// calling code--as before:
var result = await serviceProxy.GetAsync(41);
Assert.Equal(42, result);

// interception:
public void Intercept(IInvocation invocation)
{
	var returnType = invocation.Method.ReturnType;

	// For this example, we'll just assume that we're dealing with
	// a `returnType` of `Task<TResult>`. In practice, you'd have
	// to have logic for non-`Task`, `Task`, `Task<TResult>`, and
	// any other awaitable types that you care about:
	Debug.Assert(typeof(Task).IsAssignableFrom(returnType) && returnType.IsGenericType);

	// Instantiate a `TaskCompletionSource<TResult>`, whose `Task`
	// we will return to the calling code, so that we can control
	// the result:
	var tcsType = typeof(TaskCompletionSource<>)
	              .MakeGenericType(returnType.GetGenericArguments()[0]);
	var tcs = Activator.CreateInstance(tcsType);
	invocation.ReturnValue = tcsType.GetProperty("Task").GetValue(tcs, null);

	// Because we're not in an `async` method, we cannot use `await`
	// and have the compiler generate a continuation for the code
	// following it. Let's therefore set up the continuation manually:
	InterceptAsync(invocation).ContinueWith(_ =>
	{
		// This sets the result of the task that we have previously
		// returned to the calling code, based on `invocation.ReturnValue`
		// which has been set in the (by now completed) `InterceptAsync`
		// method (see below):
		tcsType.GetMethod("SetResult").Invoke(tcs, new object[] { invocation.ReturnValue });
	});
}

private async Task InterceptAsync(IInvocation invocation)
{
	// In this method, we now have the comfort of `await`:
	var response = await httpClient.GetStringAsync(...);

	// ... and we can still set the final return value! Note that
	// the return type of this method is now `Task`, not `Task<TResult>`,
	// so we can no longer `return` a value. Instead, we use the
	// "stale" `invocation` to hold the real return value for us.
	// It will get processed in the continuation (above) when this
	// async method completes:
	invocation.ReturnValue = (int)invocation.Arguments[0] + 1;
}
```

一旦我们想对后续的拦截器或代理的目标方法执行`invocation.Proceed()`，事情就会变得更加复杂。 让我们接下来看看！

## 将`invocation.Proceed()`与`async`/`await`结合使用

这是关于`invocation.Proceed()`的快速回顾：该方法用于继续执行下一个拦截器，如果没有其他拦截器，而是代理目标对象，则使用该方法。 图片来源：[Castle 动态代理 - 简介](dynamicproxy-introduction.md#interception-pipeline)

![](images/proxy-pipeline.png)

让我们直截了当地指出：在异步情况下，`Proceed()`并不会达到您的期望！ 还记得拦截器中的第一个`await`完成了拦截，即导致提早返回调用代码吗？ 这意味着拦截开始朝调用代码`冒泡`（即沿着上图中的绿色箭头）。

因此，在拦截器中`await`后，拦截已在最后一个绿色箭头的位置完成。 然后在连续中（即在`await`之后）调用`invocation.Proceed`会简单地再次进入第一个拦截器...这可能不是您想要的，并且可能导致无限循环和其他意外故障。

```csharp
// calling code:
await serviceProxy.DoAsync();

// interception:
public void Intercept(IInvocation invocation)
{
	invocation.ReturnValue = InterceptAsync(invocation);
}

private async Task InterceptAsync(IInvocation invocation)
{
	await ...;
	invocation.Proceed(); // will not proceed, but reenter this interceptor
	                      // or--if this isn't the first--an earlier one!
}
```

为了使您能够解决此问题，DynamicProxy提供了另一种方法`invocation.CaptureProceedInfo()`，该方法可让您捕获调用在侦听管道中的当前位置（即，它沿黄色箭头的当前位置）。 此方法向您返回一个对象，您可以使用该对象继续从此相同位置继续进行拦截。

上面演示的问题的解决方案现在变得非常简单：

```csharp
// calling code--as before:
await serviceProxy.DoAsync();

// interception:
public void Intercept(IInvocation invocation)
{
	invocation.ReturnValue = InterceptAsync(invocation);
}

private async Task InterceptAsync(IInvocation invocation)
{
	// If we want to `Proceed` at any point in this method,
	// we need to capture how far in the invocation pipeline
	// we're currently located *before* we `await`:
	var proceed = invocation.CaptureProceedInfo();

	await ...;

	// At this point, interception is completed and we have
	// a "stale" invocation that has been reset to the very
	// beginning of the interception pipeline. However,
	// that doesn't mean that we cannot send it onward to the
	// remaining interceptors or to the proxy target object:
	proceed.Invoke();

	// At this point, a later interceptor might have over-
	// written `invocation.ReturnValue`. As explained earlier,
	// while the calling code will no longer observe this
	// value, we could inspect it to set our own task's
	// result (if we returned a `Task<TResult>`).
}
```

## 结语

 * 正如您所看到的，异步截获在非常简单的场景中只是微不足道的，但是很快就会变得非常复杂。如果您不喜欢自己编写这样的代码，请寻找有助于异步拦截的第三方库，例如：
   * [Castle.Core.AsyncInterceptor](https://www.nuget.org/packages/Castle.Core.AsyncInterceptor) (第三方，尽管是程序包的命名空间是Castle Project的)

   * [stakx.DynamicProxy.AsyncInterceptor](https://www.nuget.org/packages/stakx.DynamicProxy.AsyncInterceptor)

   如果您是一个对异步拦截有帮助的程序库的作者，并且想将您的库添加到上面的列表中，请随时提交PR。

 * 上面的示例显示了`过时`调用对象重复设置其`ReturnValue`的情况，即使调用代码可能只观察到第一个值。然而，调用对象可能被记录在某个地方，并且可以在以后进行检查。其他方可能会观察到一个`ReturnValue`，它不能反映原始调用者得到了什么。
   因此，异步拦截器在拦截结束时将`ReturnValue`还原为实际返回到调用代码的值可能是一种好的做法。

 * 相同的建议（恢复调用代码所观察到的值），适用于使用invocation.SetArgumentValue反复覆写的引用参数。