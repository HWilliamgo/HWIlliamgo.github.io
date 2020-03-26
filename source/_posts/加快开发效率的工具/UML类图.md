箭头方向如何记忆：
箭头方向总结为：若A依赖B，则A->B。
此处的依赖也包括泛华关系，即实现接口，例如类`Student`实现了`Person`接口，则`Student`类其实依赖了`Person`类（借助于接口定义自身行为，也算是依赖）。因此`Student`->`Person`
