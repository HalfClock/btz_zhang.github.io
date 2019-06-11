---
layout:     post
title:      "浅析 python 属性描述符（下）"
subtitle:   "解析覆盖型属性描述符与非覆盖性属性描述符"
date:       2019-06-04 14:30:00
author:     "Btz"
header-img: "img/bg-python-descriptor.jpg"
catalog: true
tags:
    - python
---

# 元编程相关博文的目录及链接
这篇博文是元编程系列博文中的其中一篇、这个系列中其他博文的目录和连接见下：

1. [使用 python 特性管理实例属性](https://halfclock.github.io/2019/06/01/python-property/)
2. [浅析 python 属性描述符（上）](https://halfclock.github.io/2019/06/03/python-descriptor_01/)
3. [浅析 python 属性描述符（下）](https://halfclock.github.io/2019/06/04/python-descriptor_02/)
4. [python 导入时与运行时](https://halfclock.github.io/2019/06/07/python-import-and-running/``)
5. [python 元编程之动态属性](https://halfclock.github.io/2019/06/09/python-metacoding/)
6. [python 元编程之类元编程](https://halfclock.github.io/2019/06/10/python-class-metacoding/)

# Review
上一篇博文，我们论述了 python 属性描述符相比于 property 的优势。

使用属性描述符不仅能够保持 property 的优点，比如，在用通过 " . " 访问属性的同时使用一定逻辑进行验证，还具有 property 不具备的优势。

首先、建立属性描述符类能够将其实例赋给多个需要**同样验证逻辑**的托管类属性。

其次、**属性描述符能够使用继承来扩展**，进一步的减少代码冗余。

实际上、property 也是一类属性描述符，即覆盖型描述符。接下来，我们来认识属性描述符的具体分类。

# 描述符类型简述

在上一篇博文中，介绍描述符类时，我提到过描述符的定义，即**实现了描述符协议的类**。

**描述符协议**有三个: `__get__`、`__set__`、`__delete__`、只要类实现了其中的某一个方法，那么这个类就成为了描述符类。

实际上、**根据类是否实现 `__set__` 方法、描述符分为覆盖型描述符和费覆盖型描述符**，他们有着不同的性质和应用。

覆盖型描述符**可以不实现 `__get__` 方法**。

下面这几个类分别展示了各类描述符：

```python
class Overriding:  # 覆盖型描述符
    """数据描述符或强制描述符"""
    def __get__(self, instance, owner):
        print("get——>"+"Overriding")

    def __set__(self, instance, value):
        print("set——>" + "Overriding")

class OverridingNoGet:  # 没实现 get 的覆盖型描述符
    """没有``__get__``方法的覆盖型描述符"""
    def __set__(self, instance, value):
        print("set——>" + "OverridingNoGet")

class NonOverriding:  # 非覆盖型描述符
    """也称非数据描述符或遮盖型描述符"""
    def __get__(self, instance, owner):
        print("get——>" + "NonOverriding")

class Managed:  # 托管类实例，用于测试各类描述符的行为
    over = Overriding()
    over_no_get = OverridingNoGet()
    non_over = NonOverriding()
```

上面的代码中、Overriding 是覆盖型描述符与上一篇博文中的 Quantity 和 property 是同类型。OverridingNoGet 是没有实现 `__get__` 方法的覆盖型描述符。NonOverriding 是非覆盖型描述符，其没有 `__set__` 方法。

**Managed 是托管类实例，用于接下类测试各类描述符的行为。**

# 覆盖型描述符

接下来通过代码测试来查看覆盖类描述符的性质：

```python
>>> obj = Managed() # 1
>>> obj.over # 2
get——>Overriding
>>> Managed.over # 3
get——>Overriding
>>> obj.over = 7 # 4
set——>Overriding
>>> obj.over # 5
get——>Overriding
>>> obj.__dict__['over'] = 8 # 6
>>> vars(obj) # 7
{'over': 8}
>>> obj.over # 8
get——>Overriding
>>> Managed.over = 7 # 9
>>> obj.over # 10
8
```

**解释：**
1. 新建托管类 Managed 实例 obj **用于测试**。
2. 实例 **obj.over 会触发描述符实例的 `__get__` 方法**。
3. 托管类直接访问 over 属性也会触发描述符的`__get__` 方法。
4. 为 obj.over 赋值，会触发描述符实例的 `__set__` 方法。
5. 赋值后，使用实例访问 over 依旧会访问描述符实例的 `__get__` 方法。
6. 使用实例的 `__dict__` 属性直接创建实例属性 over ，并为其赋值
7. 通过 `vars()` 查看实例属性，发现实例属性中有 `over`，且其值为刚才赋予的 8.
8. obj.over 依旧访问描述符实例的 `__get__` 方法。
9. 尝试使用 7 将类属性 over 覆盖
10. 现在 obj.over 返回 8，而不是访问描述符实例的 `__get__` 方法了。

> 在理解上例时，请结合上一篇博文的描述符工作流程图一同理解。

#### 覆盖型描述符性质总结

首先、**不管使用 obj.over 还是 Managed.over 都会访问到描述符实例的 `__get__` 方法。**

其次、obj.over = 7 会调用描述符实例的 `__set__` 方法。

**就算通过 `__dict__` 直接给实例的 over 属性赋值，其实例的 over 属性也会被描述符实例(Managed 类属性)覆盖**，再次调用 obj.over 依旧会访问描述符实例的 `__get__` 方法。

Managed.over = 7 与 obj.over = 7 不同，其**不会调用**描述符实例的 `__set__` 方法，而是会将 Managed 类属性（描述符实例）覆盖掉，再次调用 obj.over 访问的就是实例属性了。

# 无 `__get__` 的覆盖型描述符

```python
>>> obj.over_no_get # 1
<__main__.OverridingNoGet object at 0x1100be8d0>
>>> Managed.over_no_get # 2
<__main__.OverridingNoGet object at 0x1100be8d0>
>>> obj.over_no_get = 7 # 3
set——>OverridingNoGet
>>> obj.over_no_get # 4
<__main__.OverridingNoGet object at 0x1100be8d0>
>>> obj.__dict__['over_no_get'] = 9 # 5
>>> vars(obj) # 6
{'over': 9}
>>> obj.over_no_get # 7
9
>>> Managed.over_no_get
>>> <__main__.OverridingNoGet object at 0x1100be8d0>
>>> obj.over_no_get = 7 # 8
set——>OverridingNoGet
>>> obj.over_no_get # 9
9
>>> Managed.over_no_get = 7 # 10
>>> obj.over_no_get = 10 # 11
>>> obj.over_no_get # 12
10
```

**解释：**

1. 使用 `obj.over_no_get` , 会**返回描述符实例对象。**
2. 使用 `Managed.over_no_get` ，也会返回描述符实例对象。
3. `obj.over_no_get = 7` 会调用描述符实例的 `__set__` 方法。
4. 再次调用 `obj.over_no_get` 依旧返回描述符实例对象，**说明 `obj.over_no_get = 7` 不会覆盖描述符实例。**
5. `obj.__dict__['over'] = 9` 使用实例的 `__dict__` 属性**直接创建实例属性 over_no_get ，并为其赋值为 9**。
6. 通过 `vars()` 查看实例属性、发现实例属性中有 `over_no_get`，且其值为刚才赋予的 9。
7. **再次通过 `obj.over_no_get` , 发现现在不返回描述符实例对象了，其返回的是实例属性的值 —— 9。**
8. 再次调用 `obj.over_no_get = 7` ,**依旧调用描述符实例的 `__set__` 方法。**
9. 使用 `obj.over_no_get` 查看，发现其没有受到刚才调用 `obj.over_no_get = 7` 的影响。
10. 尝试使用 7 将类属性 over_no_get 覆盖
11. 调用 `obj.over_no_get = 10` 发现**已经不调用描述符实例的 `__set__` 方法了。**
12. 发现上面的 `obj.over_no_get = 10` 对实例属性赋值了。现在的 `obj.over_no_get` 的值是10。

#### 无 `__get__` 覆盖型描述符总结 
首先、因为无 `__get__` 覆盖型描述符没有 `__get__` 方法、当其托管类实例没有对应的实例属性时，我们使用 `obj.over_no_get` 或者 `Managed.over_no_get` 拿到的是描述符实例。

其次，当托管类实例有对应的实例属性时，使用 `obj.over_no_get` 拿到的就是实例属性了，而使用 `Managed.over_no_get` 拿到还是描述符实例。

`obj.over_no_get = 7` 与有 `__get__` 覆盖型描述符行为一致。

调用 `Managed.over_no_get = 7` 也会将 Managed 类属性（描述符实例）覆盖掉。
再次调用 `obj.over_no_get = 7` 就不会调用 `__set__`了，而是给实例属性赋值。

# 覆盖型描述符工作流程

#### `obj.attr = value` 的工作流

因为覆盖型描述符都实现了 `__set__` magic 方法，**所以`obj.attr = value` 的工作流很简单、就是直接访问描述符实例的 `__set__` magic 方法。**

> 上面的例子中，当调用了 `Managed.over_no_get = 7` 后，再调用`obj.over_no_get = 7` 就不会调用描述符实例的 `__set__`。
> 
> 其原因是，此时 Managed 类的 over_no_get 类属性承载的已经不是描述符实例了，而是 7。

#### `obj.attr` 的工作流

`obj.attr` 的工作流，根据描述符有无 `__get__` 而不同，具体看下面的流程图：
![未命名文件 -1-](/img/in-post/post-python-descriptor/Overiding-set-workflow.png)
# 非覆盖型描述符
非覆盖型描述符的性质与覆盖型描述符有着很大的区别，它的具体行为可以通过观察下面的测试代码得到：


```python
>>> obj.non_over # 1
get——>NonOverriding
>>> Managed.non_over # 2
get——>NonOverriding
>>> obj.non_over = 7 # 3
>>> obj.non_over # 4
7
>>> vars(obj) # 5
{'non_over': 7}
>>> obj.__dict__['non_over'] = 9 # 6
>>> vars(obj) # 7
{'non_over': 9}
>>> Managed.non_over # 8
get——>NonOverriding
>>> Managed.non_over = 7 # 9
>>> Managed.non_over # 10
7
>>> obj.non_over # 11
9
```

1. 当实例没有 non_over 实例属性，直接调用描述符实例的 `__get__` 方法。
2. 直接访问类的 non_over 属性，也会直接调用描述符实例的 `__get__` 方法。
3. 现在尝试调用 `obj.non_over = 7` ，看在描述符实例没有 `__set__` 时，解释器如何执行。
4. 从访问结果来看、调用 `obj.non_over` 访问的是实例属性。
5. 通过 vars 查看实例属性，发现的确如此。
6. 尝试使用 `__dict__` 直接修改实例属性
7. 发现第三步的 `obj.non_over = 7` 与使用 `__dict__` 直接修改没有差别。
8. 现在直接访问类的 non_over 属性，依然访问描述符实例的 `__get__` 方法。
9. 尝试将类的 non_over 属性覆盖掉。
10. 覆盖类属性成功。
11. 对类属性赋的值对实例属性没有影响。(普通的类也有这种行为，即实例属性会覆盖类属性)

#### 非覆盖型描述符总结

非覆盖型描述符与覆盖型描述符区别很大，最大的区别在于其**因为没有实现 `__set__` 方法**，所以在调用类似 `obj.non_over = 7` 时会直接对实例属性赋值。

还有一个区别是，当实例有 `non_over` 属性时，就算描述符实现了 `__get__` 方法，`obj.non_over` **还是会直接访问实例属性。**

这与覆盖型描述符完全不同，**覆盖型描述符**在实现了 `__get__` 方法的情况下，`obj.non_over` **无论如何都会执行 `__get__` 方法**。

> 非覆盖型描述符从实例属性搜索，而覆盖型描述符从类属性搜索。

可能会有点绕，可以直接通过下面的非覆盖型描述符的 `obj.attr` 流程图来感受区别：

![未命名文件 -2-](/img/in-post/post-python-descriptor/NonOverriding-workflow.png)
看到了吗？**非覆盖型描述符首先搜索的是实例中有无 attr 实例属性，没有采取搜索托管类的属性描述符；** 而覆盖型描述符首先搜索的是托管类。

>非覆盖的意思也就显而易见了，即属性描述符的 `__get__` 方法无法覆盖实例属性。

对于非覆盖型描述符 `obj.attr = value` 的工作流不言而喻，因为其没有实现 `__set__` 方法，**所以无论如何都会给实例属性 attr 赋 value 值。**

# 各类描述符的使用场景

#### 全覆盖型描述符
>这里指实现了 `__set__` 和 `__get__` 协议的描述符。

能够使用全覆盖型描述符的场景，通常**还需要考虑是否使用特性。**这部分可以参照上一篇博文最后的总结。

此类描述符还有另一个使用的场景，即**只读属性**，只读属性的 `__set__` 只需要抛出指定的异常即可。

必须设置 `__get__` 的原因是，防止用户使用 `__dict__` 直接对实例属性进行修改，因为覆盖型描述符不管是否有实例属性，在读值时都会访问 `__get__` 方法。

#### 半覆盖型描述符
>这里指没有 `__get__` 方法的覆盖型描述符。

**此类描述符通常用于验证属性。**

即检查用户给的 value 是否符合系统定义的规则，如果符合规则，才将之存储至实例属性中，当需要拿到实例属性时，不用通过 `__get__` ，直接访问实例属性即可能快速的拿到需要的值。


#### 非覆盖型描述符
>这里指没有 `__set__` 方法的覆盖型描述符。

**此类描述符适合使用在第一次访问需要加载数据（花费时间长）的场景。———— 高效缓存**

因为第一次访问实例属性时，调用描述符实例的 `__get__` 方法，在该方法中加载数据，然后将加载完成的数据(value)，使用 obj. attr = value 赋给实例属性。

之后再访问实例属性就无需加载数据，不再访问描述符实例的 `__get__` 方法了，直接访问实例属性即可。

# 总结
本篇博文与上一篇博文总结了属性描述符是什么、怎么使用、以及何时使用的问题。

指出了属性描述符是**实现了描述符协议的类**、其**实例通常被托管类类属性所承载**、并且根据是否实现 `__set__` 方法，**分为覆盖型描述符和非覆盖型描述符，他们分别应用于只读属性、属性验证和高效缓存中。**

本文也进一步的揭示了，属性描述符相对于特性( property )的优势，**无 `__get__` 的覆盖型描述符**和**非覆盖型描述符**能在特性无法应用的场景中如鱼得水。

下一篇博文，将解释 python 语言在导入时和运行时解释器所做的事情，理解它们的区别对于观察元编程的性质而言有着很大的用处。