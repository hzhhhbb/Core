# 拦截期间引用参数的行为

DynamicProxy支持引用参数（即，在C＃中具有“ ref”或“ out”修饰符，在Visual Basic中具有“ ByRef”修饰符的参数）。 但是，这些参数在被拦截的调用中的工作方式与常规方法（非拦截）调用中的工作方式“稍有不同”。 在大多数使用情况下，您可能根本不会注意到这种差异。 但是当您这样做时，了解其存在可能会有所帮助。

**在常规方法调用期间**, 一个引用参数是调用者传递变量的完美别名：例如，更改`ref`参数的值会立即更改别名变量的值。

**在DynamicProxy方法拦截期间**, 另一方面，对ref参数的更改（你将使用IInvocation实例的Arguments属性或SetArgumentValue方法执行）不会立即反映在别名变量中。 别名变量中的更改仅在拦截完成后才可见。 这是因为DynamicProxy实际上在方法拦截的整个过程中将所有调用参数“缓冲”在单独的存储位置（`IInvocation.Arguments`）中。

* 当拦截开始时，DynamicProxy“按值”复制所有调用者传递过来的参数到`IInvocation`对象的`Arguments`数组中。那些表示引用参数的`Arguments`不索引为调用者别量变量。

* 拦截结束后，DynamicProxy会将最终值从Arguments复制回原始的ref和out参数。调用者在这时就可以看到别名变量的更改。

简而言之，在拦截期间修改引用参数将*最终反映*在参数变量中，当不是*立即反映*。

## 示范

让我们看一下这种行为上的差异。 以下代码示例将基于此接口:

```csharp
public interface IService
{
    void Execute(ref int n);
}
```

### 引用参数如何在普通C＃（或Visual Basic）中工作

如上所述，让形参成为实参的别名变量,对形参执行的任何操作都是对实参执行的。

我们可以使用以下IService的实现来看到这种行为：

```csharp
sealed class Service : IService
{
    private ExecuteDelegate execute;

    public Service(ExecuteDelegate execute) => this.execute = execute;

    public void Execute(ref int n) => this.execute?.Invoke(ref n);
}

delegate void ExecuteDelegate(ref int n);
```

然后按如下方式使用和调用`IService.Execute`：

```csharp
int n = 0;

var service = new Service(
    execute: (ref int alias) =>
    {
        Console.WriteLine(n);  // => 0
        alias = 42;
        Console.WriteLine(n);  // => 42
    });

Console.WriteLine(n);          // => 0
service.Execute(ref n);
Console.WriteLine(n);          // => 42
```

请注意，在lambda中，我们更改了`alias`参数，可以立即观察到对别名`n`的更改。

### 引用参数在DynamicProxy拦截期间如何工作

如上所述，对`ref`参数的更改将仅在拦截之后（而不是在拦截期间）在别名变量中可见。 让我们来看一个例子。 除了让我们自己实现`IService`之外，我们将让DynamicProxy创建一个实例。 因此，我们需要一个拦截器：

```csharp
sealed class Interceptor : IInterceptor
{
    private Action<IInvocation> intercept;

    public Interceptor(Action<IInvocation> intercept) => this.intercept = intercept;

    public void Intercept(IInvocation invocation) => this.intercept?.Invoke(invocation);
}
```

注意与上述示例的`IService`实现的相似性。 这里的基本思想是相同的：我们希望能够定义在可以访问别名变量的lambda内，方法调用期间应该发生的情况:

```csharp
int n = 0;

var service = new ProxyGenerator().CreateInterfaceProxyWithoutTarget<IService>(new Interceptor(
    intercept: invocation =>
    {
        Console.WriteLine(n);  // => 0
        invocation.SetArgumentValue(0, 42);
        Console.WriteLine(n);  // => 0 !!!
    }));

Console.WriteLine(n);          // => 0
service.Execute(ref n);
Console.WriteLine(n);          // => 42
```

请注意，这次，在调用期间更改参数值不会立即影响别名变量`n`，即使最终会更新。
