---
layout:     post
title:      "使用 python 特性管理实例属性"
subtitle:   "@porperty 装饰器的正确用法和意义"
date:       2019-06-01 22:30:00
author:     "Btz"
header-img: "img/bg-python-property.png"
catalog: true
tags:
    - python
---

# 说在前面
你可能听说过 **python 元编程**的大名，使用元编程技术可以在程序运行中**动态的创建属性甚至动态创建类**。本博文**暂时不会讲述元编程的关键知识，而是讲述进行元编程所要知道的的基础知识**，用以更好的理解 python 元编程的原理和性质。

本篇博文，包括接下来的几篇博文，会讲述**相对零散的元编程基础知识**、本篇讲述如何使用 python 特性来管理实例属性；

接下来、我会说明 python 特性的本质——属性描述符、及**使用属性描述符来更好的管理类的实例属性。**

再下一篇博文则会**系统的说明 python 中属性描述符的分类与性质**，甚至会说明 python 如何使用非覆盖型属性描述符实现类方法；

再下一篇博文则会说明 **python 在导入时和运行时分别会做出的动作**。

最后的两篇博文则会讲述 python 中的**元编程技术，包括动态创建属性及动态创建类**

# 元编程相关博文的目录及链接
1. [使用 python 特性管理实例属性](https://halfclock.github.io/2019/06/02/python-property-and-descriptor/)
2. [浅析 python 属性描述符（上）]()
3. [浅析 python 属性描述符（下）]()
4. [python 导入时与运行时]()

# 管理属性的古老方法 -> set( ) / get( )

在实际的面向对象编程中，很多场景下必须使用一定的**存储逻辑来管理实例属性**，使应用程序能够正常使用。

例如，在水果零售系统中，客户在购买水果时可以指定购买几斤该水果、可以看到关于该水果的描述、单价等信息，当然还需要根据单价和重量来决定最后的售价，那么水果的类可以这样表示:

```python
class Fruits:

    def __init__(self, price, weight, description):
        # 水果的描述
        self.description = description
        # 水果的价格
        self.price = price
        # 水果的重量
        self.weight = weight

    def subtotal(self):
        # 小记
        return self.price * self.weight
```

但是这似乎不太合理，如果水果店老板不小心把水果的 price 设置成负数，或者顾客恶意的把 weight 设置成负数，那么水果店可能过两天就倒闭了。

为了防止这样的情况出现、**我们必须设置 price 和 weight 的存储规则、使其被设置的时候不能设置小于零的数。**

**那么传统的方法是：使用 set() 和 get() 来管理，顺序是：**  
1. 为了**不能直接使用 “ . ” 直接访问**，我们必须将相关的属性都设置成私有（python 里的做法是在属性前加双下划线）。
2. **为了能够使用存储规则存储数据**，我们必须提供一系列的 set 和 get 方法。

这样做之后的代码如下：


```python
class Fruits:

    def __init__(self, price, weight, description):
        # 水果的描述
        self.__description = description
        # 水果的价格
        self.__price = price
        # 水果的重量
        self.__weight = weight
        
    def set_description(self, value):
        self.__description = value
    
    def get_description(self):
        return self.__description
    
    def set_price(self, value):
        if value >= 0:
            self.__price = value
    
    def get_price(self):
        return self.__price
    
    def set_weight(self, value):
        if value >= 0:
            self.__weight = value
    
    def get_weight(self):
        return self.__weight

    def subtotal(self):
        # 小记
        return self.__price * self.__weight
```

上面的代码中，我为 price 、weight、description 都设置的 set() 和 get() 方法，其中 price  、 weight 还设置了小于零的值无法存储的规则。

这是传统的管理属性的方法，毫无疑问也是最简单的管理方法。

然而、这样做的弊端也很明显：

1. 首先、**不能够方便的使用 “ . ” 来访问属性了**，编写程序时，还必须多写几个字母。
2. 一堆 set() 和 get() 方法不仅写起来费劲，而且**容易掩盖住真正重要的 subtotal 方法。**
3. 如果水果店的老板需要给水果增加一些有同样存储逻辑属性，你**依然得重新编写 set() 和 get() 方法。**
4. **想要修改某一个 set/get 时，寻找某个属性的 set/get 方法的过程就需要花费大量的时间。**

尽管，像 idea 这种 java IDE 已经有了方便的自动生成 get/set 的功能。但是这依旧没法有效的解决上面几个弊端。

简单的使用 python 的 property 装饰器能够解决第一、二个弊端，如下一节所示。

# 使用 property 管理属性
使用 property 装饰器管理属性，不仅能够使用 “ . ” 方便的访问属性，还能在存取值时加上自己需要的规则。

下面这段代码是在最初的 Fruits 类改善而来的：


```python
>>> class Fruits:

    def __init__(self, price, weight, description):
        # 水果的描述
        self.description = description
        # 水果的价格
        self.price = price
        # 水果的重量
        self.weight = weight

    @property  # ①
    def weight(self):
        print("get:苹果的重量")
        return self.__weight

    @weight.setter # ②
    def weight(self, value):
        print("set:苹果的重量")
        if value >= 0:
            self.__weight = value
        else:
            raise ValueError("想干嘛呢？") # ③

    def subtotal(self):
        # 小记
        return self.price * self.weight

>>> apple = Fruits(10, 2, "apple") # ④
set:苹果的重量
>>> apple.weight # ⑤
get:苹果的重量
2
>>> apple.subtotal()
20
>>> apple.weight = -1 # ⑥
set:苹果的重量
Traceback (most recent call last):
...
ValueError: 想干嘛呢？ # ⑦
```

在上面的代码中：

①: 使用 `@property` 装饰器装饰 weight 函数，注意:**该函数返回的不是 self.weight 而是 self.__weight。**

②: **函数在使用`@property` 装饰器装饰后，会变成类属性（特性），而且会有一个 setter 方法，该方法也是一个装饰器，作用是装饰同属性(特性)的 set 函数。**被装饰的函数必须与属性（被`@property` 装饰器装饰的函数）同名。

③: 当 value 小于零时，不能设值，应当抛出异常。

④: 创建一个 Fruits 对象 apple 用于测试特性的行为是否符合预期。注意：**在构造 apple 实例时 init 特殊方法会调用由 `@weight.setter` 装饰的方法，输出“set:苹果的重量”字符串。**

⑤: 可以使用 “ . ” 访问 weight，并且**访问是通过 `@property` 装饰的函数访问的。**

⑥: 尝试为 apple 实例设置 weight 为 -1。

⑦: **apple.weight = -1 语句访问的是由 `@weight.setter` 装饰的方法**，并且因为 value 不满足预期，程序抛出异常。

---
> **首先我们必须知道、python 特性都是类属性、但是特性管理的其实是实例属性的存取。**

由上面这个例子，可以得到使用 `@property` 的方法：

1. 使用 `@property` 装饰器**装饰的函数会变成该类的特性，特性名就是函数名。**
2. 之后使用 `@特性名.setter` 装饰器装饰该特性的 “set 方法”，**此方法名必须与特性名一致。**

当类函数被 `@property` 装饰时，实际上，这个函数已经成为了该类的特性，**也就是该类的类属性了**，这个过程在解释器导入该模块时就已经确定了。这可以通过观察上例中的 ④ 得到。

因为我们注意到，在实例初始化时，self.weight = weight 会调用 weight 特性的 set 方法。

**实际上，在实例初始化前，weight 就已经是 Fruits 类的特性了。**使用 `self.weight` ，解释器会调用 weight 特性的 get 方法。使用 `self.weight = 某个值` ，解释器会调用 weight 特性的 set 方法。

**那么 python 要怎么知道 weight 是否是 Fruits 类的特性呢？**换句话说，python 怎么知道 `self.weight = 2` 语句该访问 weight 特性的 set 方法，还是该新建实例属性 weight 呢？

通常，python 按照以下流程图来执行 `self.weight = 2` 语句：
![](/img/in-post/post-python-property/15594841902205.jpg)

对应的文字描述是：
1. 先去 Fruits 类中搜索，看有没 weight 特性，若有，那么直接调用该特性的 set 方法。
2. 若在 Fruits 类中搜索不到特性，那么去实例的属性中寻找，是否有 weight 这个属性，若有为这个属性重新设值。
3. 若实例中的属性也没有 weight ，那么创建实例属性，并赋值。

**实际上，执行 `self.weight ` 语句也是按照此流程，只不过，set 方法替换成了 get 方法，赋值变成了取值，最后如果实例属性也没有 weight ，那么会抛出异常。**

> 这个流程图也使用于覆盖类型的属性描述符，什么是覆盖类型的属性描述符，我将在下一篇博文中介绍

# 特性是用于管理实例属性的
再强调一遍、**所有的特性都是类属性、但特性管理的是实例属性的存取。**

必须要注意到，上例中、无论是 get 方法还是 set 方法，最终操作的对象都是实例属性`__weight`

再将代码列出：
```python
    @property
    def weight(self):
        print("get:苹果的重量")
        return self.__weight

    @weight.setter
    def weight(self, value):
        print("set:苹果的重量")
        if value >= 0:
            self.__weight = value
        else:
            raise ValueError("想干嘛呢？")
```
**被 @property 装饰的 weight 方法为什么不能返回 self.weight 呢？**

首先、特性应该管理实例属性，而现在的 weight 已经是类属性了。

其次，如果写成 `return self. weight` 那么调用方 get 到的是 weight 特性、**程序会调用 weight 特性的 "get" 方法，如此一来，程序陷入无限递归。**

>在理解这句话时，请配合上一小节的流程图理解。

若在初始化实例后，查看该实例的实例属性，那么**会看到 weight 特性的 set 方法为 apple 的私有属性设了值。**

```python
>>> apple = Fruits(10, 2, "apple")
set:苹果的重量
>>> vars(apple)
{'description': 'apple', 'price': 10, '_Fruits__weight': 2}
```

同样、使用 weight 特性的 get 方法时，其实是取上面代码中的 `_Fruits__weight` 属性。


# 结语
可以看出，我们已经使用 python 特性解决了古老的 set()/get() 方法所不能解决的两个弊端，**最直观的好处是、我们可以用 “.” 来方便的访问属性了。**

其次，在使用了 `@property` 和 `@特性名.setter` 装饰器以后，**我们可以很清楚的看出哪些方法是用于处理存储逻辑的，哪些是处理业务逻辑的。**

---

但是、在上述代码中，我只是重写了 weight 属性的特性，若我再将 price 属性相关的特性也写出，**那么代码依旧会变得冗长**。

并且、**weight 属性与 price 属性的存储逻辑是一致的**，即不能存小于零的数，如果只使用特性，那么我们就不得不为 price 再写一遍几乎同样的代码，这让人心烦。 

---

好在，python 的 `@property` 装饰器，是由类来实现的，**该类实现了全部的属性描述符的接口**，也可以说，**property 装饰器本身就是一种属性描述符**。

那么，如果我们自己编写属性描述符类，再将 weight 和 price 设置为 Fruits 的类属性，并且赋予他们属性描述符实例，就可以减少代码的重复了。

---

什么是属性描述符？如何使用属性描述符来解决古老的 set()/get() 方法无法顾及的弊端呢？
请看下一篇博文。

















