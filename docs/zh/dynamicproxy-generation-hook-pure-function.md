# 使代理生成钩子功能纯粹

纯函数是对于给定的一组输入始终返回相同输出的函数。 在代理生成钩子的情况下，这意味着对于给定类型的两个相等的代理生成钩子（由重写的Equals / GetHashCode方法指定），代理将从其方法返回相同的值，并在再次询问相同的值时，类型将再次返回相同的值/引发相同的异常。

这是DynamicProxy做出的主要假设，这就是使缓存机制起作用的原因。 如果代理生成钩子等于已用于生成代理类型的钩子，则DynamicProxy将假定它返回与另一个代理钩子相同的值，这将导致相同的代理类型，因此它会缩短生成过程并返回现有的 代理类型。

## See also

* [在代理生成钩子上重写Equals / GetHashCode](dynamicproxy-generation-hook-override-equals-gethashcode.md)