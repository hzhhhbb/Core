# 在代理生成钩子上重写Equals / GetHashCode

## Equals/GetHashCode

使用DynamicProxy的最常见错误之一是没有在代理生成钩子上重写`Equals` /`GetHashCode`方法，这意味着你正在放弃缓存，再加上BCL中的错误，就意味着性能下降（增加了内存消耗）。

## 解决方案

解决方案非常简单，该规则也没有例外,就是始终在实现`IProxyGenerationHook`的所有类上重写`Equals` / `GetHashCode`方法。

## See also

* [使代理生成钩子功能纯粹](dynamicproxy-generation-hook-pure-function.md)