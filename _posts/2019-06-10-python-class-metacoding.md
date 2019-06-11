---
layout:     post
title:      "python 元编程之类元编程"
subtitle:   "使用元编程技术动态创建类"
date:       2019-06-10 23:30:00
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

# 前言

所谓类元编程，与属性元编程对应，**其实就是在运行时动态创建/修改类的技术。**

如果你熟悉 python 标准库，那么你一定知道 collections.namedtuple ，实际上 **namedtuple 就能够动态创建类。**


```python
>>> from collections import namedtuple
>>> Fruits = namedtuple("Fruits","description weight price")
>>> apple = Fruits("apple", 10, 2)
>>> apple
Fruits(description='apple', weight=10, price=2)
>>> _, weight, price = apple
10 2
```

如上代码所示，namedtuple 第**一个参数是类名字符串**、**第二个参数是以空格分割的类属性**，**返回值是类对象。**

确实，Fruits 变量承载了一个类对象，因为其能够创建 apple 实例，并且 apple 实例**还是一个可迭代的对象**，能够进行拆包操作（第 7 行）。

接下来，我们会自己创建一个功能与 nametuple 类似的类工厂函数**说明动态创建类的方法**、然后，我们还会说明如何在**创建类后动态修改类**，即使用类装饰器。最后我会说明**元类的一些性质。**

# 生产类的工厂

因为我们创建的类工厂的功能要与 nametuple 类似、即**生产的类能够迭代**、并且**能够以友好的方式输出描述字符串**，所以，生产出的类需要实现 `__repr__` 和 `__iter__` magic 方法。

当然，因为需要接受以空格分割的类属性字符串，所以需要实现 `__init__` magic 方法，**用以初始化实例。**实现的类工厂函数 —— class_factory 如下：

```python
>>> def class_factory(cls_name, field_names):

        field_names = field_names.replace(',', ' ').split()  # 1
        field_names = tuple(field_names)  # 2
    
        def __init__(self, *args, **kwargs):  # 3
            attrs = dict(zip(self.__slots__, args))
            for name, value in attrs.items():
                setattr(self, name, value)
    
        def __iter__(self):  # 4
            for name in self.__slots__:
                yield getattr(self, name)
    
        def __repr__(self):  # 5
            values = ', '.join('{}={!r}'.format(*i) for i
                               in zip(self.__slots__, self))
            return '{}({})'.format(self.__class__.__name__, values)
    
        cls_attrs = dict(__slots__ = field_names,  # 6
                         __init__  = __init__,
                         __iter__  = __iter__,
                         __repr__  = __repr__)
    
        return type(cls_name, (object,), cls_attrs)  # 7

Fruits = class_factory("Fruits","description weight price") # 8
>>> apple
Fruits(description='apple', weight=10, price=2)
>>> _, weight, price = apple
10 2
```

解释：

1. 根据传入的**以空格分割的类属性进行拆分**，拆分后放在 field_names 列表中。
2. 将之转化为 tuple 类型，这里在为 `__slots__` 属性赋值做准备。
3. 这个函数即是新类的 `__init__` 函数，这里只设置了若干个定位参数，需要注意的是，若果传入的参数的**数量超过 `__slots__`中的参数数量**，那么此函数只会按**照传入参数的顺序**为实例设置属性。
4. **实现了可迭代对象的接口**，这里返回生成器，用于按顺序遍历实例属性。
5. 实现 `__repr__` 接口，使得实例能够以友好的方式输出描述字符串。
6. **创建类属性字典。**
7. 使用 type 元类构造方法，**构建新类，并返回类对象。**
8. 使用 class_factory 创建类对象并赋给 Fruits ，**下面的行为和使用 nametuple 一样。** 

通常，我们把 type 当做函数使用，例如 `type(obj)` 可以获取该 obj 所属的类。然而 **type 是一个元类，当我们传入三个参数时可以构建一个类对象。**

这三个参数分别是：

1. **name**: **新类的名字**。
2. **bases**: 一个元组，是多重继承的**父类集合**。
3. **dict**: 一个字典，呈放着指定的**新类的属性和值**。

>类函数可以视为特殊的类属性，所以上例中的 `__init__`、`__iter__`、`__repr__` 也算作属性。

所以，**使用标准库提供的 type 元类可以很容易的实现运行时动态创建属性**，实际上大部分标准库类、用户自己实现的类都是 type 类的实例，包括 object 类。

>值得注意的是，nametuple 的实现要比这个复杂的多，它并不是用元类 type 来实现的，而是使用内置的 `exec` 处理字符串形式的源码模板实现的。这里是 nametuple 的源码 ——> [nametuple](https://hg.python.org/cpython/file/3.4/Lib/collections/__init__.py#l236)

除了动态创建类以外，python 还可以**使用类装饰器动态定制类**。

# 使用类装饰器动态定制类

你可能还记得在 [浅析 python 属性描述符（上）](https://halfclock.github.io/2019/06/03/python-descriptor_01/) 一文中我使用了属性描述符管理托管实例属性，下面是那篇文章的代码片段：


```python
import de_model as model
class Fruits:
    description = model.NonBlank("description")
    weight = model.Quantity("weight")
    price = model.Quantity("price")
    ... 省略 ...
```

你是否注意到了，我们在创建属性描述符实例时必须要将托管类对应的属性名以字符串的形式传递进去，因为在描述符实例在调用 `__set__` 方法时会用到这个参数(attrname)。

**这显然很麻烦，因为我们要写两遍同样的变量**，有什么办法能够简化他呢？

因为 `model.Quantity()` 在右边，**解释器无法在执行这句代码时知道等式左边的变量的名字是什么！**

所以，我们必须要在 Fruits 类构建完了之后，再**定制每一个属性描述符实例的 attrname。**

碰巧的是，类装饰器正是在 Fruits 类构建完之后起作用的，**它是一个接受类对象作为参数，返回原来的类/修改后的类的函数。**

>与函数装饰器类似，不过函数装饰器接收的是函数对象。

这样，我们可以**写一个类装饰器来达成目的**：


```python
def entity(cls):  # <1>
    for key, attr in cls.__dict__.items():  # <2>
        if isinstance(attr, Validated):  # <3>
            attr.attrname = key  
    return cls  # <4>
```

解释：

1. 类装饰器函数的参数是一个类对象。
2. 迭代该类的属性字典，拿到各个属性及其值，对于 Fruits 类来说，这个字典有三个键：description、weight、price，值分别是对应的属性描述符实例。
3. 如果该属性的值是 `Validatad` 的实例，那么**将该描述符实例的 attrname 属性重新赋值为字符串形式的键。**
4. 返回修改后的类。

将这个类描述符放在 models 模块后，Fruits 类这样使用它：


```python

import de_model as model
@model.entity
class Fruits:
    description = model.NonBlank()
    weight = model.Quantity()
    price = model.Quantity()
    ... 省略 ...
```

可以看出，重构以后的 Fruits 仅多了一行 `@model.entity` ，等式右边烦人的参数可以不写了，**就算以后为它添加其他属性也一样。**

**类装饰器能够以比较简单的方式做到以前需要使用元类去做的事情 - 创建类时定制类。**

然而装饰器有一个缺点，无论是类装饰器还是函数装饰器，**其只对直接依附的类有效，其子类无法继承这个装饰器所作出的改动。**

然而，**使用元类能够弥补这个 “缺点”。**

# 理解元类

为了理解元类在发挥着类装饰器的作用时，又**弥补了类装饰器只对依附的类有效的 “缺点”**。我们必须先了解元类的一些性质，以及其如何作为类装饰器使用。

首先、根据本文的第一部分的 type 元类可以看出：**元类是制造类的工厂。**

《流畅的 python》的作者 Luciano 画的一副涂鸦能够很好的表明元类与类的关系：

![](/img/in-post/post-python-class-metacoding/15601821873491.jpg)

上图中**绿色的工厂就是元类、而蓝色的工厂是元类生产的类对象。**

-------

所以，**元类的实例是类**。

其次、**元类的子类还是元类**，例如，在 [python 导入时与运行时](https://halfclock.github.io/2019/06/07/python-import-and-running/) 一文里，evalsupport 模块的 MetaAleph 类，其继承了元类 type，所以它也是元类。


```python
class MetaAleph(type):
    print('<[400]> MetaAleph body')

    def __init__(cls, name, bases, dic):
        print('<[500]> MetaAleph.__init__')

        def inner_2(self):
            print('<[600]> MetaAleph.__init__:inner_2')

        cls.method_z = inner_2
```

元类 MetaAleph 的**构造方法接收和 type 类的构造方法几乎一样的参数**，不同的是，这里还多了一个 cls 参数，这里的 cls 是指类对象，也就是说， **MetaAleph 不仅能构造类对象，还能够对已经构造好的类对象进行修改。**

从这个角度看，元类可以当做类装饰器使用，因为，除了使用元类的构造方法直接构造类对象以外、**还可以使用如下方法声明某类是某元类的实例**：

```python
class A(metaclass = MetaAleph):
    ... 省略 ...
```

上面的代码中，A 类使用了 `metaclass  = MetaAleph` 声明了自己是元类 MetaAleph 的**实例**，注意，是实例！

这样一来，解释器在执行完 A 类的定义体以后，得到 A 类的类对象、类名、父类集合、参数字典数据，并将之**传递给 MetaAleph 类的构造方法** `__init__` ，**真实的 A 类对象其实是经过  `__init__` 修改过的类**。

这样的行为和类装饰器没什么不同，只不过，使用元类有一个好处 —— **A 类的子类也会是元类 MetaAleph 的实例。**

即、**元类的实例的子类也是该元类的实例**

>也许你会感觉很绕，开始怀疑人生，没关系，后面会有实例用于说明。

例如下面的类 A 也将是 MetaAleph 类的实例，在构建完 B 类对象之后，依旧要传递给
MetaAleph 类的构造方法进行修改，实际上的 B 类对象也是修改后的类对象。

```python
class A(metaclass = MetaAleph):
    ... 省略 ...

class B(A)
    ... 省略 ...
```


这样一来，我们就了解了，元类有**四个非常重要的性质**：

1. **元类的实例是类。**
2. **元类的子类还是元类。**
3. **元类可以当做类装饰器来使用。**
4. **元类的实例的子类也是该元类的实例。**

#### 通过测试理解元类

接下来，我会通过**类似于** [python 导入时与运行时](https://halfclock.github.io/2019/06/07/python-import-and-running/) 一文中的 evaltime 模块来说明元类的四个重要性质。

>注意，下面代码的 evalsupport 模块与 [python 导入时与运行时](https://halfclock.github.io/2019/06/07/python-import-and-running/) 一文一模一样，如果你没有看过这篇文章，我建议你先去看，因为对理解下面代码的行为帮助很大。

```python

from evalsupport import deco_alpha
from evalsupport import MetaAleph

print('<[1]> evaltime_meta module start')


@deco_alpha
class ClassThree(): # 1
    print('<[2]> ClassThree body')

    def method_y(self):
        print('<[3]> ClassThree.method_y')


class ClassFour(ClassThree): 
    print('<[4]> ClassFour body')

    def method_y(self):
        print('<[5]> ClassFour.method_y')


class ClassFive(metaclass=MetaAleph): # 2
    print('<[6]> ClassFive body')

    def __init__(self):
        print('<[7]> ClassFive.__init__')

    def method_z(self):
        print('<[8]> ClassFive.method_y')


class ClassSix(ClassFive): # 3
    print('<[9]> ClassSix body')

    def method_z(self):
        print('<[10]> ClassSix.method_y')


if __name__ == '__main__':
    print('<[11]> ClassThree tests', 30 * '.')
    three = ClassThree()
    three.method_y()
    print('<[12]> ClassFour tests', 30 * '.')
    four = ClassFour()
    four.method_y()
    print('<[13]> ClassFive tests', 30 * '.')
    five = ClassFive()
    five.method_z()
    print('<[14]> ClassSix tests', 30 * '.')
    six = ClassSix()
    six.method_z()

print('<[15]> evaltime_meta module end')
```

解释：

1. 删除了 ClassOne、ClassTwo，ClassThree 与 ClassFour 与之前的版本一致。
2. 新增了 ClassFive ，它是 MetaAleph 的实例。
3. 新增了 ClassSix，他是 ClassFive 的子类。

和 [python 导入时与运行时](https://halfclock.github.io/2019/06/07/python-import-and-running/) 一文一样我们将分别测试导入时与运行时的输出。

>我建议你先将自己的答案写出来，再看下去

在 python 命令行中敲入 `import evaltime` 会出现如下输出：

```python
<[100]> evalsupport module start
<[400]> MetaAleph body
<[700]> evalsupport module end
<[1]> evaltime module start
<[2]> ClassThree body
<[200]> deco_alpha 
<[4]> ClassFour body # 1
<[6]> ClassFive body 
<[500]> MetaAleph.__init__ # 2
<[9]> ClassSix body
<[500]> MetaAleph.__init__ # 3
<[15]> evaltime module end
```

解释：

1. 与之前一样，虽然 ClassFour 继承了 ClassThree ，但是**装饰 ClassThree 的装饰器 deco_alpha 并不会装饰 ClassFour。**
2. ClassFive 是 MetaAleph 类的实例，所以在**构建完 ClassFive 类对象之后会调用 MetaAleph 的 `__init__` 方法**。
3. ClassSix **继承了 ClassFive**，**尽管它没有显式声明**它是 MetaAleph 类的实例，但是**解释器依旧在构建完 ClassSix 类对象之后调用了** MetaAleph 的 `__init__` 方法。

在系统命令行中敲入 `python evaltime.py` 会出现如下输出：


```python
<[100]> evalsupport module start
<[400]> MetaAleph body
<[700]> evalsupport module end
<[1]> evaltime_meta module start
<[2]> ClassThree body
<[200]> deco_alpha
<[4]> ClassFour body
<[6]> ClassFive body
<[500]> MetaAleph.__init__
<[9]> ClassSix body
<[500]> MetaAleph.__init__ # 1
<[11]> ClassThree tests .............................. 
<[300]> deco_alpha:inner_1 
<[12]> ClassFour tests ..............................
<[5]> ClassFour.method_y # 2
<[13]> ClassFive tests ..............................
<[7]> ClassFive.__init__ 
<[600]> MetaAleph.__init__:inner_2 # 3
<[14]> ClassSix tests ..............................
<[7]> ClassFive.__init__
<[600]> MetaAleph.__init__:inner_2 # 4
<[15]> evaltime_meta module end
```

解释：

1. 之前的输出与导入时一致
2. ClassFour **没有被 deco_alpha 装饰。**
3. ClassFive 的 method_z() 被元类的 inner_2 替换掉了。
4. ClassFive 的子类 ClassSix 的 method_z() **也被元类的 inner_2 替换掉了**。

这些测试不仅说明了**类装饰器只对直接依附的类有效，其子类无法继承这个装饰器所作出的改动**。还说明了，**元类（MetaAleph）的实例（ClassFive）的子类（ClassSix）也是该元类的实例**。

即，类装饰器可能对子类没有影响，但是元类对实例的子类依旧有影响。

# Django 的 models 模块

**Django 的 models 是利用元编程技术的典型案例**，下面是一段 Django modesls 的经典代码：

```python
from django.db import models # 3

class Item(models.Model): # 2

    text = models.TextField(default='') # 1
```

1. 这里的 models.TextField() 明显与[浅析 python 属性描述符（上）](https://halfclock.github.io/2019/06/03/python-descriptor_01/) 一文中的 model.Quantity() 一样都是属性描述符，用于管理 Item 类的实例属性。
2. models.Model 类是某元类的实例，Item 继承了 models.Model **只是为了声明自己是 那个元类的实例**，该元类的**作用可以稍稍参考本文的 entity 类装饰器**。
3. models 模块与 [浅析 python 属性描述符（上）](https://halfclock.github.io/2019/06/03/python-descriptor_01/) 一文中的 model 模块类似，存放着各类属性描述符及其父类，并且还存放着像 models.Model 一样的元类。

这三行代码，几乎将我这六篇元编程相关博文的知识都囊括进去了，不仅**使用了 models.TextField() 属性描述符管理托管类 Item 的实例属性**。而且还**继承了 models.Model 这个元类的实例，实现了动态修改类的效果。**

可以说，**动态创建实例属性、动态修改/定制类**在这三行代码中均有体现。

# END

这六篇系列文章**并不能囊括所以的元编程知识，相反这仅仅是元编程最基础的知识。**幸运的是，在日常编程中，我们并不需要/几乎不使用元编程的相关知识，元编程技术通常是那些框架开发者在频繁使用。

但是，如你所见，这六篇博文中**多多少少都会有对日常生产有所帮助的思想/场景**。比如：正确使用 python 特性和属性描述符就能帮助我们很好的管理实例属性，减少冗余代码。

希望这个系列的博文对你有所帮助，如果发现一些错误，请在评论中指出，我会尽快改正。