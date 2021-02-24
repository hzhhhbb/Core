# 可选参数的限制

DynamicProxy使用`System.Reflection`和`System.Reflection.Emit`创建代理类型。 由于这些工具（在.NET和Mono上）存在一些错误和局限性，因此不可能在每种情况下都如实地重现动态生成的代理类型中可选参数的默认值。

在实践中，这通常无关紧要，但是对于反射生成代理类型的框架或库（例如Dependency Injection容器）而言，这可能会成为问题。


## 当你的代码在Mono上运行时

在Mono（至少5.16版本）上，DynamicProxy可能无法正确再现代理类型中的默认参数值...

* **类型为`DateTime?`或`decimal?`的可选参数。** 无论实际的默认参数值是什么，反射可能会（通过`ParameterInfo.[Raw]DefaultValue`）报告此类参数的默认值是`null`（直到Mono 5.10）或`Missing.Value`（在5.16上）。

   你只能通过在代理类型的原始方法中二次检查默认值来找到正确的默认值。

   根本原因已在[mono/mono#8504](https://github.com/mono/mono/issues/8504)和[mono/mono#8597](https://github.com/mono/mono/issues/8597)（针对Mono 5.10）, [mono/mono＃11303](https://github.com/mono/mono/issues/11303)（针对Mono 5.16）中记录。

## 当你的代码在.NET Framework或.NET Core上运行时

.NET Framework（至少4.7.1版本）和.NET Core（至少2.1版本）受有关默认参数值的几个错误或限制的影响。DynamicProxy可能无法在代理类型中正确地再现默认参数值...

* **一些默认值为default(SomeStruct)的`Struct`类型。** 如果反射（通过`ParameterInfo.[Raw]DefaultValue`）报告此类参数的默认值`Missing.Value`，则您可以放心地假设*正确*的默认值为`default（SomeStruct）`。

   请注意，如果在这种情况下反射报告默认值为`null`，则这不是错误，而是正常的`System.Reflection`行为。 在这种情况下，您也可以安全地假定`default（SomeStruct）`为正确的默认值。

   对于.NET Core，此问题的根本原因已在[dotnet/corefx#26164](https://github.com/dotnet/corefx/issues/26164)中记录.

* **一些具有null缺省值的可空枚举类型`SomeSnum?`的可选参数。** 如果反射（通过`ParameterInfo.[Raw]DefaultValue`）报告此类参数的默认值`Missing.Value`，那么您可以放心地假设实际的默认值不是`null`。

   你只能通过在代理类型的原始方法中二次检查默认值来找到正确的默认值。

   对于.NET Core，此问题的根本原因已在[dotnet/coreclr#17893](https://github.com/dotnet/coreclr/issues/17893)中记录。

* **泛型参数为`Struct`类型的可选参数。** 例如，给定泛型类型`C<T>`和方法`void M(T arg = default(T))`，如果代理封闭的泛型类型为`C<SomeEnum>`，则可能会报告反射（通过`ParameterInfo.[Raw]DefaultValue`）代理参数`arg`的默认值`Missing.Value`。 如果是这样，您可以放心地假设实际默认值为`default（SomeEnum）`。

   请注意，如果在这种情况下反射报告默认值为`null`，则这不是错误，而是正常的`System.Reflection`行为。 在这种情况下，你也可以安全地假设`default（SomeEnum）`是正确的默认值。

   对于.NET Core，此问题的根本原因已在[dotnet/coreclr#29570](https://github.com/dotnet/corefx/issues/29570)中记录。
