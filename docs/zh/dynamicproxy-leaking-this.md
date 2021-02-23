# 泄漏 `this`

## 问题

思考这个简单的接口和实现:

```csharp
public interface IFoo
{
    IFoo Bar();
}

public class Foo : IFoo
{
    public IFoo Bar()
    {
        return this;
    }
}
```

现在，假设我们使用`IFoo`创建了一个代理，并像这样使用它：

```csharp
var foo = GetFoo(); // returns proxy
var bar = foo.Bar();
bar.Bar();
```

你能看出这里有什么bug吗? 第二次执行的方法不是在代理对象上调用的，而是在目标对象上! 我们的代理泄漏了目标对象。

该问题显然不会影响代理类（因为在这种情况下，代理和目标是同一对象）。 为什么动态代理不自行处理这种情况？ 因为没有通用的简便方法来处理此问题。 我展示的示例是最琐碎的示例，但是代理对象可能以多种不同的方式泄漏该示例。 它可以将其泄漏为返回对象的属性，可以将其泄漏为引发事件的发送方参数，可以将其分配给某些全局变量，也可以将其自身传递给其自身参数之一的方法，等等。动态代理可以不能预测其中任何一个，也不应该预测。

## 解决方案

最好知道这样的问题存在并了解其后果，虽然在某些情况下，您通常对此无能为力。 但是，在其他情况下，解决此问题确实非常简单。

```csharp
public class LeakingThisInterceptor:IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        invocation.Proceed();
        if (invocation.ReturnValue == invocation.InvocationTarget)
        {
            invocation.ReturnValue = invocation.Proxy;
        }
    }
}
```

添加了一个拦截器（将其作为拦截器管道中的最后一个），它将泄漏的目标切换回代理实例。 就这么简单。 请注意，此拦截器专门针对上述示例中的场景（目标对象通过返回值泄漏）。 对于每种情况，您都需要一个专用的拦截器。