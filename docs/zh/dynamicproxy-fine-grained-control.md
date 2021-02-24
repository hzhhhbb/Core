# 使用*代理生成钩子*和*拦截器选择器*进行精细化控制

你的拦截器看起来像这样吗？

```csharp
public void Intercept(IInvocation invocation)
{
    if (invocation.TargetType != typeof(Foo))
    {
        invocation.Proceed();
        return;
    }

    if (invocation.Method.Name != "Bar")
    {
        invocation.Proceed();
        return;
    }

    if (invocation.Method.GetParameters().Length != 3)
    {
        invocation.Proceed();
        return;
    }

    DoSomeActualWork(invocation);
}
```

## 解决方案

如果是这样，通常意味着你做错了。 将决策移至`IProxyGenerationHook`和`IInterceptorSelector`。

* 我是否想拦截这种方法？ 如果答案是否定的，请使用代理生成钩子将其从代理方法中筛选掉。

> 请注意，由于DynamicProxy 2.1中的错误，如果您选择不使用接口代理上的代理方法，则会出现异常。 解决方法是拦截该方法，然后使用拦截器选择器不为该方法返回任何拦截器。 此错误已在DynamicProxy 2.2中修复

* 如果确实要拦截此方法，我要使用哪些拦截器？ 我需要所有这些吗？ 我只需要一个吗？ 使用拦截器选择器进行控制。
## 当你不这样做的时候

另一方面，请记住，每个功能都是双刃剑。 过度使用代理生成钩子和拦截器选择器可能会大大降低代理类型缓存的效率，这可能会损害性能。 与往常一样，请考虑需要多少控制以及对缓存的影响。 有时，如果仅在顶部放置一个拦截器，则比增加代理数量好。 与往常一样–在尽可能类似于您的生产方案的方案中尝试，以检查哪个选项最适合您。

## See also

* [拦截器的单一职责原则](dynamicproxy-srp-applies-to-interceptors.md)