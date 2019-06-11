---
layout:     post
title:      "python 元编程之动态属性"
subtitle:   "使用元编程技术动态创建属性"
date:       2019-06-09 11:30:00
author:     "Btz"
header-img: "img/bg-python-metacoding.jpg"
catalog: true
tags:
    - python
---

# 元编程相关博文的目录及链接
这篇博文是元编程系列博文中的其中一篇、这个系列中其他博文的目录和连接见下：

1. [使用 python 特性管理实例属性](https://halfclock.github.io/2019/06/01/python-property/)
2. [浅析 python 属性描述符（上）](https://halfclock.github.io/2019/06/03/python-descriptor_01/)
3. [浅析 python 属性描述符（下）](https://halfclock.github.io/2019/06/04/python-descriptor_02/)
4. [python 导入时与运行时](https://halfclock.github.io/2019/06/07/python-import-and-running/)
5. [python 元编程之动态属性](https://halfclock.github.io/2019/06/09/python-metacoding/)
6. [python 元编程之类元编程](https://halfclock.github.io/2019/06/10/python-class-metacoding/)

#  前言

元编程是一门程序运行时动态创建属性/类的技术，本篇文章主要讲述创建动态属性，也可以说是属性元编程技术。

元编程这个词汇看起来很高大上，但是在生产环境中用的也不少，而所谓的属性元编程实际上就是**在程序运行时为实例动态增加属性。**

这样一来，**使用特性(property)管理实例属性其实也算是一类属性元编程**，当用户访问实例本不存在的属性时，python 解释器从类中搜索有无特性（类属性承载），若有则会调用特性的相关接口并动态**创建/赋给**真实的实例属性。

>同理，属性描述符也是一类属性元编程技术。

然而、**在解释器从类和超类搜索不到相应的特性/属性描述符时**，python 也提供了相应的接口来处理找不到的属性，这就为我们提供了实现属性元编程的方法。

# 特性能够动态创建实例属性
如果你读过系列文章第一篇，那么一定了解了 python 特性的概念以及如何使用，如果你读过系列文章的第二篇，那么一定了解 python 特性的本质，以及其在何时使用。

接下来我会用一个例子来说明特性如何动态创建实例属性。

>还是 Fruits 类的例子，只不过这次为了更好的展示特性如何动态创建实例属性，有一些逻辑上的错误。

```python
>>> class Fruits:
    
        def __init__(self):
            pass
    
        @property
        def weight(self):
            return self.__weight
    
        @weight.setter
        def weight(self, value):
            if value >= 0:
                self.__weight = value
            else:
                raise ValueError("想干嘛呢？")

>>> apple = Fruits()
>>> vars(apple) # 1
{}
>>> apple.weight = 2 # 2
>>> vars(apple) # 3
{'_Fruits__weight': 2}
>>> apple.weight = -1
Traceback (most recent call last):
...
ValueError: 想干嘛呢？
```

**解释：**

1. 通过 vars() 查看实例的属性、因为我让 `__init__` 方法为空，所以 apple 实例没有任何属性。
2. 直接为实例的 weight 属性赋值，这里实际上会调用 Fruit 类 weight 特性的 “set” 方法。
3. 赋值后再次查看 apple 实例的变量，这一次 apple 实例却有属性了还是私有属性 `_Fruits__weight`。

>在上例中，直接 pass 掉 `__init__` 是逻辑错误的，但是却能够帮助我们看清特性是如何动态创建实例属性的。

重点是、在运行时**解释器为 apple 实例动态创建了一个私有属性** `_Fruits__weight` **并能对其的设值进行动态验证。**

如果你读过[使用 python 特性管理实例属性](https://halfclock.github.io/2019/06/01/python-property/)，你应该会知道在使用 `obj.attr = value` 语句时解释器会去类中搜索有无特性/覆盖型属性描述符，有特性/覆盖型属性描述符时会调用它们的接口、若都没有，才会去搜索实例的属性字典。

实际上、这个过程是由 python 提供的一对 magic 方法 —— `__getattribute__` 和 `__setattr__` 实现的，也就是说语句 `obj.attr` 或 `getattr(obj,attr)` 无论如何也会调用 obj 的 `__getattribute__(self,name)`，语句 `obj.attr = value` 无论如何也会调用 obj 的 `__setattr__(self,name,value)`，并由它们来进行寻找特性/属性描述符的工作。

而当实例没有该属性，所属的类也没有对应的特性/属性描述符时，`__getattribute__(self,name)` 方法不会直接抛出异常，而是会**调用另一个 magic 方法** `__getattr__(self,name)`。

# 使用 `__getattr__` 创建属性

现在，水果店老板给了你一个水果列表，包括**水果的名字和该水果对应的描述**，这个列表有数百种水果。麻烦的是，产品经理小姐姐提出了一个**奇怪的需求**：要求你**必须使用 "obj.某水果的名字" 的形式来获取该水果的描述**，而不能使用字典的形式 dict["某水果的名字"] 获取。

在小姐姐向你抛了一个媚眼后，你答应了她，并写下了如下代码。(๑´ㅂ`๑)


```python
>>> class FruitsList:

        def __init__(self, fdict):
            self.fruit_dict = fdict
    
        def __getattr__(self, item):
            return self.fruit_dict[item]

>>> fdict = {"apple": "apple description", "pear" : "pear description", "banana" : "banana description"} # 1
>>> fruitlist = FruitsList(fdict) # 2
>>> vars(fruitlist) # 3
{'fruit_dict': {'apple': 'apple description', 'pear': 'pear description', 'banana': 'banana description'}}
>>> fruitlist.apple # 4
apple description
>>> fruitlist.pear # 5
pear description
>>> vars(fruitlist) # 6
{'fruit_dict': {'apple': 'apple description', 'pear': 'pear description', 'banana': 'banana description'}}
```

解释：

1. fdict 是老板给的水果数据的**模拟测试数据**，只包含三种水果。
2. 使用 fdict 构建 FruitsList 实例。
3. 查看 fruitslist 实例的属性、发现其**只有一个属性** —— fruit_dict 。
4. 访问实例的 apple 属性，并**成功拿到其值。**
5. 访问实例的 pear 属性，并**成功拿到其值。**
6. 此时 fruitslist 实例**还是只有一个属性。**

上面这个例子表明了解释器在实例没有对应属性、实例所属的类没有对应特性/属性描述符时，python 解释器会如何处理实例属性的访问 —— 调用 `__getattr__(self,name)` 特殊方法。

这个例子中，在访问 fruitlist 实例不存在的属性时，解释器动态返回了值，**看上去就好像运行时依据 fruit_dict 动态为 fruitlist 实例创建了属性一样**。没错，使用 `__getattr__(self,name)` 很容易就实现了属性元编程。

当有数百种水果都需要用像**访问实例属性的方式**来访问时，我们不可能为 FruitList 类手动创建数百种水果属性或者特性，显然使用 `__getattr__(self,name)` 相对合理。

-------

**产品经理小姐姐提出的需求并不合理**、但是，日常生产中的确有**类似的应用场景**，例如，当 python 加载如下的 json 格式的数据时，我们想要拿到最里层 "d1" 的值，通常使用字典访问：`data["a"]["d"][0]["d1"]`，这很麻烦。
```python
{
    "a":{
        "b":[{"b1":"bv"}],
        "c":[{"c1":"cv"}],
        "d":[{"d1":"dv"}]
    }
}
```

但是，当我们合理编写 `__getattr__(self,name)` magic 方法，我们就能实现 `data.a.b.d[0].d1` 这样的访问方式，少了一堆中括号和双引号看上去会舒服得多，这就是动态属性访问的魅力。

> 使用动态属性访问 JSON 类数据的代码有很多（网上有大量实现），不过《流畅的 python》 第 19 章有一个实现相对简单，这里是源码 -> [FrozenJSON](https://github.com/fluentpython/example-code/blob/master/19-dyn-attr-prop/oscon/explore0.py)

# 使用 `__dict__` 快速创建属性

对于上面这个例子来讲，还有一个简单的方法能够在运行时的动态为 fruitlist 创建属性 —— 使用 `__dict__` 特殊属性。

将上述代码重构：


```python
>>> class FruitsList:

        def __init__(self, fdict):
            self.__dict__.update(fdict)  # 1
   

>>> fdict = {"apple": "apple description", "pear" : "pear description", "banana" : "banana description"}
>>> fruitlist = FruitsList(fdict) # 2
>>> vars(fruitlist) # 3
{'apple': 'apple description', 'pear': 'pear description', 'banana': 'banana description'}
>>> fruitlist.apple # 4
apple description
>>> fruitlist.pear
pear description
```
解释：

1. FruitsList 类的构造方法将传进来的参数都放进实例的 `__dict__` 中。
2. 使用 fdict 构造实例。
3. 查看 fruitlist 实例的属性，发现此时实例属性已经更新，其拥有了 fdict 里的所有数据。
4. 访问实例的 apple 属性，发现可以拿到正确的数据。


在 python 中，`__dict__` 是存储着对象或者类的可写属性。有 `__dict__` 属性的对象任何时候都能随意设置新属性值。如果类有 `__slots__` 属性，那么它的实例可能就没有 `__dict__` 属性。

>类可以定义 `__slots__` 属性，限制实例能够有哪些属性。`__slots__` 属性的值是一个字符串组成的元组，指明允许有的属性。

# End

总的来说、python 里动态创建属性难度不大、除了使用特性和属性描述符在解释器调用 `__getattribute__(self,name)` 阶段动态创建属性、我们还可以在解释器调用 `__getattr__(self,name)` 阶段动态创建属性，甚至，我们能够**直接使用 `__dict__` 属性直接批量动态创建实例属性**。

但是使用 `__dict__` 不太安全，因为你永远不知道用户传递的数据是怎样的，规范与否，字典嵌套了几层等都不是我们能控制的，所以我更推荐使用像 [FrozenJSON](https://github.com/fluentpython/example-code/blob/master/19-dyn-attr-prop/oscon/explore0.py) 这样的处理方式，**自己在 `__getattr__(self,name)` 里编写控制代码总是要安心的多。**

除了动态创建属性，python 甚至可以**动态创建/修改类**，这就**涉及到类的类 —— 元类**。
