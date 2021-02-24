# Castle 动态代理

Castle 动态代理为您的对象生成代理，你可以使用它们来透明地为其添加或更改行为，提供预处理/后期处理以及许多其他功能。 以下是一些较知名和流行的用法：

* [Castle Windsor](http://www.castleproject.org/projects/windsor/) 使用代理来启用其拦截功能并用于类型化工厂
* [Moq](https://github.com/moq/moq4) 使用它来提供“ .NET最受欢迎和友好的模拟框架”
* [NSubstitute](http://nsubstitute.github.io/) 使用它来提供“ .NET模拟框架的友好替代品”
* [FakeItEasy](http://fakeiteasy.github.io/) 使用它来提供“ .NET的简单模拟库”
* [Rhino Mocks](https://www.hibernatingrhinos.com/oss/rhino-mocks) 使用它提供“用于.NET平台的动态模拟对象框架”
* [NHibernate](http://nhibernate.info/) 使用它提供延迟加载功能（v4.0之前的版本）
* [Entity Framework Core](https://github.com/aspnet/EntityFrameworkCore) 在包[Microsoft.EntityFrameworkCore.Proxies](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Proxies)中使用它来提供延迟加载代理

如果您不熟悉DynamicProxy，则可以阅读[快速介绍](dynamicproxy-introduction.md)，浏览该库中核心类型的描述,或者进行更高级的详细讨论：
* [代理类型](dynamicproxy-kinds-of-proxy-objects.md)
* [泄漏代理的对象](dynamicproxy-leaking-this.md)
* [使代理生成钩子功能纯粹](dynamicproxy-generation-hook-pure-function.md)
* [在代理生成钩子上重写Equals / GetHashCode](dynamicproxy-generation-hook-override-equals-gethashcode.md)
* [使你的类可序列化](dynamicproxy-serializable-types.md)
* [使用代理生成钩子和拦截器选择器进行精细控制](dynamicproxy-fine-grained-control.md)
* [使你的拦截器符合单一职责原则](dynamicproxy-srp-applies-to-interceptors.md)
* [拦截期间引用参数的行为](dynamicproxy-by-ref-parameters.md)
* [可选参数的限制](dynamicproxy-optional-parameter-value-limitations.md)
* [异步拦截](dynamicproxy-async-interception.md)

:information_source: **`Castle.DynamicProxy.dll` 去哪了?:** 在2.5版本之前，DynamicProxy 存在于同名的程序集中。在2.5版本的修改中，它被合并到`Castle.Core.dll`。

:warning: **使用ProxyGenerator的同一实例:** 如果您的运行时间较长（网站，Windows服务等），并且必须创建许多动态代理，则应确保重用同一个ProxyGenerator实例。 如果不是使用同一实例，您将绕过缓存机制，这会导致CPU使用率高和内存消耗不断增加。

## See also

* [Castle DictionaryAdapter](dictionaryadapter.md)
