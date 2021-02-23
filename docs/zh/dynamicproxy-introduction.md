# Castle 动态代理 - 简介

Castle 动态代理 是一个用于在运行时动态生成轻量级.NET代理的库。 代理对象允许在不修改类代码的情况下截取对对象成员的调用。

DynamicProxy与内置在CLR中的代理实现不同，后者要求代理类继承MarshalByRefObject。 继承`MashalByRefObject`来代理一个对象可能太麻烦了，因为它不允许类继承另一个类，也不允许透明的类代理。 此外，Castle 动态代理还提供了标准CLR代理无法提供的功能，例如，它允许您混合多个对象。

## 要求

要使用Castle 动态代理，您需要以下环境:

* 已安装以下运行时之一
  * .NET Framework 4.5+
  * .NET Core 2.1+
  * 任何其他支持.NET Standard 2.0+和使用System.Reflection.Emit生成运行时类型的.NET平台
* `Castle.Core.dll` (动态代理所在的程序集)

:information_source: **DynamicProxy 程序集:** 在以前的版本（v2.2之前的版本）中，DynamicProxy存在于其自己的程序集“ Castle.DynamicProxy.dll”中。 后来将其移至“ Castle.Core.dll”，现在不需要其他程序集即可使用它。

## 代理

首先，DynamicProxy的用途是什么？为什么要关心它？ DynamicProxy（简称DP），顾名思义，是一个可帮助您实现代理对象设计模式的框架。 动态意味着实际创建代理类型是在运行时进行的，您可以动态地组成代理对象。

根据维基百科：

> 在最常见的情况下，代理是一个类，它充当另一件事的接口。 另一件事可能是任何东西：网络连接，内存中的大对象，文件或其他昂贵或无法复制的其他资源。

通过《黑客帝国》，来帮助理解代理。

![](images/matrix.jpg)

矩阵可作为代理的比喻（图片很酷）

我认为地球人都看过这部电影，并且喜欢它的情节。无论如何，矩阵中的人不是真实的人（记得经典台词“There is no spoon”吗？）。它们是真实的人的代理，这些人可能在任何地方。 它们相貌和行为都像，但是它们实际上并不是它们。 另一个含义是不同的规则适用于代理。 代理可以是被代理对象，但是代理对象可以更多（飞行，逃离子弹或类似东西）。 希望你能明白这一点，然后再将这个比喻扩展开来。 更重要的一件事是，代理最终将行为委托给它们后面的实际对象（有点像-“如果您在矩阵中被杀，您也会在现实生活中死亡”）。

WCF代理是程序员日常工作中透明代理的一个很好的例子。 从使用它们的代码的角度来看，它们只是实现接口的一些对象。 他们可以像其他任何对象一样通过接口使用它，即使他们正在使用的接口的实际实现可能在另一台机器上。

这是使用代理的一种方式，它向用户隐藏实际对象的位置。

## 拦截管道

另一种动态代理的最常见用法，是向代理对象添加行为。您可以使用实现了`IInterceptor`接口的拦截器将行为注入代理。

![](images/proxy-pipeline.png)

动态代理如何工作的示意图

上面的图片显示了它是如何工作的。

* The blue rectangle is the proxy. Someone calls a method on the proxy (denoted by yellow arrow). Before the method reaches the target object it goes through a pipeline of interceptors.
* Each interceptor gets an `IInvocation` object (which is another important interface from DynamicProxy) that holds all the information about the current request, such as the `MethodInfo` of the intercepted method, along with its parameters and preliminary return value; references to the proxy and the proxied object; and a few other bits. Each invoked interceptor gets a chance to inspect and change those values before the actual method on the target object is called. For example, an interceptor may log debug information about the arguments passed to the method, or validate them.
* Each interceptor can call `invocation.Proceed()` to pass control further down the pipeline. Interceptors usually call this method just once, but multiple calls are allowed, e.g. when implementing a "retry". (It is also permissible to cut short the interception pipeline and omit the call to `Proceed` altogether.)
* When the last interceptor calls `Proceed`, the actual method on the proxied object is invoked, and then the call travels back, up the pipeline (green arrow) giving each interceptor another chance to inspect and act on the returned value or thrown exceptions.
* Finally the proxy returns the value held by `invocation.ReturnValue` as the return value of called method.

### Interceptor example

If this was not clear enough, here's a sample interceptor, that shows how it works:

```csharp
[Serializable]
public class Interceptor : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        Console.WriteLine("Before target call");
        try
        {
           invocation.Proceed();
        }
        catch(Exception)
        {
           Console.WriteLine("Target threw an exception!");
           throw;
        }
        finally
        {
           Console.WriteLine("After target call");
        }
    }
}
```

Hopefully, at this stage you have a pretty good idea about what DynamicProxy is, how it works, and what it's good for. In the next chapter we'll dive into some more advanced capabilities, plugging into, and influencing the process of generating proxy class.

## See also

[Kinds of proxy objects](dynamicproxy-kinds-of-proxy-objects.md)
