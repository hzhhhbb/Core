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


So, how would we communicate a return value back to the calling code when the intercepted method's return type is not just `Task`, but say, `Task<int>`? The following example shows how you can control the result of the task received by calling code:


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


Unfortunately, it usually won't be *that* easy in real-world scenarios, where you cannot assume that every method that your interceptor deals with will have the exact same return type of `Task<int>`. So, instead of being able to just comfortably `return someInt;` from a `async Task<int>` method, you'll have to resort to a non-generic `Task`, a `TaskCompletionSource`, and some reflection:


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


Phew! And things get even more complex once we want to do an `invocation.Proceed()` to a succeeding interceptor or the proxy's target method. Let's look at that next!


## 将`invocation.Proceed()`与`async`/`await`结合使用

Here's a quick recap about `invocation.Proceed()`: This method gets used to proceed to the next interceptor in line, or, if there is no other interceptor but a proxy target object, to that. Remember this image from the [introduction to DynamicProxy](dynamicproxy-introduction.md#interception-pipeline):

![](images/proxy-pipeline.png)

Let's get straight to the point: `Proceed()` will not do what you might expect in an async scenario! Remember that the very first `await` inside your interceptor completes interception, i.e. causes an early return to the calling code? This means that interception starts to "bubble back up" towards the calling code (i.e. along the green arrows in the above picture).

Therefore, after having `await`-ed in your interceptor, interception has completed at the position of the last green arrow. Calling `invocation.Proceed` in the continuation (i.e. after the `await`) will then simply advance to the very first interceptor again... that's likely not what you want, and can cause infinite loops and other unexpected malfunctions.


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


In order for you to be able to work around this problem, DynamicProxy offers another method, `invocation.CaptureProceedInfo()`, which allows you to capture the invocation's current position in the interception pipeline (i.e. where along the yellow arrows it is currently located). This method returns an object to you which you can use to continue interception from this very same location onward.

The solution to the problem demonstrated above now becomes very simple:


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


## Closing remarks

 * As you have seen, async interception is only trivial in very simple scenarios, but can get quite complex very quickly. If you don't feel comfortable writing such code yourself, look for third-party libraries that help you with async interception, for example:

   * [Castle.Core.AsyncInterceptor](https://www.nuget.org/packages/Castle.Core.AsyncInterceptor) (third-party, despite the Castle Project's package namespace)

   * [stakx.DynamicProxy.AsyncInterceptor](https://www.nuget.org/packages/stakx.DynamicProxy.AsyncInterceptor)

   If you are the author of a generally useful async interception helper library, and would like to add your library to the above list, feel free to submit a PR.

 * The above examples have shown cases where a "stale" invocation object has its `ReturnValue` repeatedly set, even though only the first value might be observed by calling code. It is however possible that invocation objects are recorded somewhere, and could be inspected later on. Other parties might then observe a `ReturnValue` that does not reflect what the original caller got.

   Because of that, it might be good practice for your async interceptors to restore, at the end of interception, the `ReturnValue` to the value that actually made it back to the calling code.

 * The same recommendation&mdash;restoration to the value(s) observed by the calling code&mdash;applies to by-ref arguments that have been repeatedly overwritten using `invocation.SetArgumentValue`.
