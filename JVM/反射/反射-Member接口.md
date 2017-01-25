# 反射-Member接口

Member接口定义了单个成员（可以是属性、构造函数或者方法）的信息，它提供了以下几个方法：

1. getDeclaringClass: 返回声明这个成员的类信息

2. getModifier: 返回修饰符信息

3. getName: 返回成员的名称信息

4. isSynthetic：标识当前成员是否为人造的，即是否是由编译器引入的

在反射框架中，Field, Method以及Constructor都实现了这个接口