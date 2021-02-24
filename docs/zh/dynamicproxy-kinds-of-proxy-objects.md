# 代理类型

开箱即用的DynamicProxy提供了几种可以使用的代理对象。 它们分为两大类：

## 基于继承

通过继承代理类来创建基于继承的代理。 代理拦截对类的虚拟成员的调用，并将其转发给基本实现。 在这种情况下，代理和代理对象是一个。 这也意味着您无法为现有对象创建基于继承的代理。 DynamicProxy中有一种基于继承的代理。

* 类代理 - 为类创建基于继承的代理. 只有类中的virtual（虚拟的）成员能被拦截。

## 基于组合

基于组合的代理是一个新对象，它从代理的类/实现的代理接口继承，并且（可选）将拦截的调用转发到目标对象。 DynamicProxy公开以下基于组合的代理类型：

* 有目标对象的类代理 - 针对类的代理. 它不是一个完美的代理。如果类具有非虚拟方法或公共字段，则无法拦截它们，从而给代理用户带来对象状态不一致的视图。 因此，应谨慎使用。
* 没有目标对象的接口代理 - 针对接口的代理. 这种代理不需要提供目标对象。 取而代之的是，拦截器应该为代理的所有成员提供实现。
* 有目标对象的接口代理 - 顾名思义，这种代理包装了目标对象，并将对这些接口的调用转发给目标对象。
* 具有目标接口的接口代理 - 这种代理是其他两种接口代理的混合体。它允许但不要求提供目标对象。 它还允许在代理的生存期内交换目标。它不与代理目标的一种类型绑定，因此一种代理类型可以重用于不同的目标类型，只要它们实现目标接口即可。

## 外部资源

* [Tutorial on DynamicProxy discussing with examples all kinds of proxies](http://kozmic.pl/dynamic-proxy-tutorial/)
* [Castle Dynamic Proxy not intercepting method calls when invoked from within the class](http://stackoverflow.com/questions/6633914/castle-dynamic-proxy-not-intercepting-method-calls-when-invoked-from-within-the/)
* [Why won't DynamicProxy's interceptor get called for *each* virtual method call?](http://stackoverflow.com/questions/2153019/why-wont-dynamicproxys-interceptor-get-called-for-each-virtual-method-call)