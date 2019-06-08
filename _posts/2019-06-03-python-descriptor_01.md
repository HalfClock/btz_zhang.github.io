---
layout:     post
title:      "浅析 python 属性描述符（上）"
subtitle:   "python 使用属性描述符高效的管理实例属性"
date:       2019-06-03 12:30:00
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
4. [python 导入时与运行时](https://halfclock.github.io/2019/06/07/python-import-and-running/)

# Review

在上一篇博文中、我们使用 python 特性（property）管理了实例属性，最大的好处是：在使用 property 装饰器后，我们**能够在通过使用 " . " 这种方便的方式（obj.attr）来访问实例属性的同时，为其设置存储规则。**

并且，因为处理存储的函数都有 `@property` 或者 `@特性名.setter` 这种**明显的装饰器标志**，我们**可以很容易的找到处理业务逻辑的核心函数。**

然而、**当大量的属性都需要相同的存取逻辑做控制时**，例如：水果类的 weight 和 price 的值均不能小于零。**单纯的使用 property 依然无法避免代码的重复。**

当重复为 weight 和 price 编写几乎相同的代码时，**重构代码的时机到了！！！**

运用和 property 同宗**的属性描述符可以很好的避免代码段的重复。**

# 博文的编写思路
首先、我会直接亮出使用属性描述符重构的代码(基于上一篇博文的 Fruits 类)，**用以给你属性描述符，以及其如何减少重复代码的直观印象。**

然后、我会仔细的说明**实现属性描述符需要注意的细节**、**属性描述符的性质**、**及其如何与托管类实例交互。**

接着、我会说明为什么 python 特性也是一种属性描述符

最后、我会举例说明属性描述符比之 python 特性和优势在哪里

# 使用属性描述符进行高效管理

下面这段代码是基于上一篇博文的 Fruit 类进行的重构：


```python
class Quantity:
    def __init__(self, attrname):
        self.attrname = attrname

    def __get__(self, instance, owner):
        print("get:"+str(instance.description)+"的" + str(self.attrname))
        return instance.__dict__[self.attrname]

    def __set__(self, instance, value):
        print("set:"+str(instance.description)+"的" + str(self.attrname))
        if value >0:
            instance.__dict__[self.attrname] = value
        else:
            raise ValueError("想干嘛呢？")
class Fruits:

    weight = Quantity("weight")
    price = Quantity("price")

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
        
>>> apple = Fruits(10, 2, "apple") # 1
set:apple的price
set:apple的weight
>>> pear = Fruits(11, 3, "pear")
set:pear的price
set:pear的weight
>>> vars(apple) # 2
{'description': 'apple', 'price': 10, 'weight': 2}
>>> vars(pear)
{'description': 'pear', 'price': 11, 'weight': 3}
>>> apple.weight # 3
get:apple的weight
2
>>> apple.price 
get:apple的price
10
>>> apple.weight = 10 # 4
set:apple的weight
>>> apple.weight # 5
get:apple的weight
10
>>> vars(pear) # 6
{'description': 'pear', 'price': 11, 'weight': 3}
>>> apple.price = -1 # 7
Traceback (most recent call last):
...
ValueError: 想干嘛呢？
```

上例使用属性描述符类 Quantity 类对 Fruits 类进行了重构，使用了 Quantity 实例作为 Fruits 类的 weight 和 price 的类属性值。

可以看到、**重构后的 Fruits 类实例有如下行为**：

1. 在**初始化** Fruits 类实例时，**会调用描述符实例的 `__set__` magic 方法。**
2. 查看实例 apple 的全部属性、发现其**有 price 和 weight 实例属性**。这是由于 `__set__` magic 方法使用 `instance.__dict__[self.attrname] = value` 语句为实例属性设了值
3. **可以通过 apple.weight 访问实例属性**，但是其是通过描述符实例的 `__get__` magic 方法来访问的。
4. **可以通过 apple.weight = 10 这种方式为实例属性设置值**，但是其实通过描述符实例的 `__set__` magic 方法来访问的。
5. 能够访问到刚刚为 apple 实例设置的值。
6. 查看 pear 实例的全部属性、发现**刚才对 apple 实例的所有操作对 pear 实例毫无影响。**
7. 若设置 value 为负值，那么会抛出异常。

> 如果你不明白这些行为的原理，没关系，我会在下一小节解释属性描述符的原理，现在你只需要知道，重构后的代码有着这样的行为。

#### 重构后的好处

好了，在使用了 Quantity 属性描述重构了 Fruits 类之后、**我们依然可以使用 " . " 方便的访问实例属性，并同时做存储逻辑的验证。**

而且、**我仅用了 30 行代码就实现了 weight 和 price 两个实例属性的管理。**要知道、上一篇博文中，仅实现了 weight 属性的管理就写了 27 行代码。

不仅是代码量减少了，如果水果店老板想为所有水果都增加一个折扣属性(discount)、其也不能为负值。那么我们只需要在 Fruit 类中增加一行代码 `discount = Quantity("discount")` 这大大减少了重复代码，提高了代码的可重用性。

# 属性描述符原理
为了说明白属性描述符的原理，我将先说明一些专有名词。
#### 专有名词

**描述符类**

* **实现了描述符协议的类**、比如上例中的 Quantity 类、它实现了描述符类的一些协议(`__get__`、`__set__`)。

> 实现了`__get__`、`__set__`、`__delete__` 方法的类是描述符，只要实现了其中一个就是。

**托管类**

* **将描述符实例作为类属性的类**，比如上例中的 Fruits 类，他有 weight、price 两个类属性，且都被赋予了描述符类的实例。

**描述符实例**

* **描述符类的实例**、比如上例中 Fruits 类中就用 `Quantity("weight")` 创建了一个描述符实例，**通常来讲，描述符类的实例会被赋给托管类的类属性。**

**托管实例**

* **托管类的实例**、比如上例中的 apple 、 pear。

**托管属性**

* **托管类中由描述符实例处理的公开属性**、比如上例中 Fruits 类的**类属性** weight、price

**存储属性**
* **可以粗略的理解为、托管实例的属性**、在上例中使用 `vars(apple)` 得到的结果中 price 和 weight **实例属性**就是存储属性，它们实际**存储着***_实例的_***属性值**

> 你可能不理解存储属性和托管属性的区别、因为在上例中**托管属性与存储属性同名**。
> 那么，你只需要记住：**托管属性是类(Fruits)属性、存储属性是实例(apple)的属性。**

#### 描述符类与托管类的关系

首先，描述符类与托管类都是类，可以将他们想象成类的工厂、下面两张图很好的展示了他们之间的关系:
> 以下两张图修改自《流畅的 python》 第 20 章

![](/img/in-post/post-python-descriptor/7311559545526_.pic.png)


如上图、Quantity 作为描述符实例的工厂、**产出了两个实例并绑定至 Fruits 类的类属性 weight、price**。

Fruit 作为托管类实例的工厂、可以产出多个实例、**每一个实例都有两个存储属性 weight、price**

![](/img/in-post/post-python-descriptor/Xnip2019-06-03_15-15-50.jpg)

上图将描述符的两个实例抽象成了两个小机器人、手上拿着一个放大镜和一个手抓，**放大镜用于获取托管类实例的值(`__get__`)、手抓用于设置托管类实例的值(`__set__`)**。

值得注意的是、Fruits 工厂不管生产多少实例，**都只能拥有两个描述符小机器人**，因为它们是类属性。

#### 描述符实例如何工作
如果你看过上一篇博文、你可能还记得上一篇博文中的特性的工作流程图。实际上特性就是一种属性描述符、所以在这里 **Quantity 属性描述符的工作流程与特性几乎一致**，例如:`apple.weight = 10` 语句的执行过程如下面的流程图：

![](/img/in-post/post-python-descriptor/descriptor_workflow.png)
>以上流程、对于 Quantity 这个类型的描述符而言，`apple.weight` 这样的代码的执行流程与上图几乎没有差别，无非是在搜索到有 weight 描述符实例时，调用 `__get__` magic 方法；在搜索到有 weight 实例属性时获取该属性的值；都搜索不到则抛出异常。

# property 是一种属性描述符

为什么说 python 特性也是一种属性描述符呢？让我们看 python 2.2 之前是如何使用 property的：

>property 类实现了完整的描述符协议


```python
def get_weight(instance):
    print("get weight")
    return instance.__dict__["weight"]

def set_weight(instance, value):
    print("set weight")
    if value > 0:
        instance.__dict__["weight"] = value
    else:
        raise ValueError("想干嘛呢？")

class Fruits:

    weight = property(get_weight, set_weight)
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

>>> apple = Fruits(10, 2, "apple")
set weight
>>> vars(apple)
{'description': 'apple', 'price': 10, 'weight': 2}
>>> apple.weight
get weight
2
>>> apple.weight = 10
set weight
>>> apple.weight
get weight
10
>>> apple.weight = -1
Traceback (most recent call last):
...
ValueError: 想干嘛呢？
```

你看出来了吗？上例中的我们写了一对 set/get 方法，并用他们生成了一个 property 对象，赋予了 Fruits 类的 weight 类属性。这和属性描述符类 Quantity 有太多的相似之处。

**实际上 property 类的构造方法返回一个描述符实例，该实例的 `__get__` magic方法即是get_weight、`__set__` magic 方法既是 set_weight。**

甚至这样做以后、**也能很好的管理 Fruits 实例的 weight 属性**，行为与使用 Quantity 一致。 

肯定有人会说、既然这样，**那我完全没有必要使用 Quantity 了，直接写一个特性工厂函数即可！！！**
比如：

> 代码来自《流畅的 python》第19章

```python
def quantity(storage_name): 

    def qty_getter(instance):  
        return instance.__dict__[storage_name]  

    def qty_setter(instance, value):  
        if value > 0:
            instance.__dict__[storage_name] = value  
        else:
            raise ValueError('value must be > 0')

    return property(qty_getter, qty_setter)  
```

这样一来、`weight = Quantity("weight")` 就可以转变为 `weight = quantity('weight')`。甚至比创建 Quantity 类的代码还要短。

但是，对于描述符类来说、依旧有着得天独厚的优势，**即面向对象的方式**

# 描述符类能够继承
现在，项目的产品经理小姐姐提出了一个合理的需求 —— **Fruits 的描述不能为空！！**这很合理，因为描述为空时，顾客在系统中根本看不到自己买的是哪种水果。

轮到程序猿头疼了，难道再增加一个描述符类，或者特性工厂函数吗？如果这样做了，那**小姐姐以后又提出一种新属性的存取逻辑怎么办？**

我们注意到、不管是值不能小于零、还是描述不能为空，**二者的存取逻辑都在 set 方法上，对于 get 方法几乎没有逻辑验证。**

那么我们为什么不写一个**描述符类**、其实现了通用的 `__get__` 方法，再设置一个抽象的验证方法、新来的描述符类只需要继承该抽象类，再覆盖该验证方法即可。这样能够极大的节省代码冗余。

> 这种思想通常被称为模板方法设计模式

实现代码如下：

```python

import abc


class AutoStorage:

    def __init__(self, attrname):
        self.attrname = attrname

    def __get__(self, instance, owner): # __get__ 方法除了必要的判断 instance 是否真实存在以外，操作与之前几乎一致
        if instance is None:
            return self
        else:
            return instance.__dict__[self.attrname]

    def __set__(self, instance, value): # 1
        instance.__dict__[self.attrname] = value


class Validated(abc.ABC, AutoStorage): # 2

    def __set__(self, instance, value):
        value = self.validate(instance, value) # 3
        super().__set__(instance, value) # 4

    @abc.abstractmethod  
    def validate(self, instance, value): # 5
        """返回经过验证的值或抛出异常"""


class Quantity(Validated):  
    """验证值是否大于等于零"""

    def validate(self, instance, value): # 6
        if value < 0:
            raise ValueError('value must be > 0')
        return value


class NonBlank(Validated):
    """验证字符串是否不为空"""

    def validate(self, instance, value): # 7
        value = value.strip()
        if len(value) == 0:
            raise ValueError('value cannot be empty or blank')
        return value 
```

在上述代码中：

1. **AutoStorage 描述符类的 set 方法不做任何验证。**
2. Validated 不但**继承了 AutoStorage 描述符类**、**而且还是一个抽象类。**
3. **重写了 `__set__` magic 方法**，并将 value 设为经过 validate 方法验证过的值。
4. 经过**验证后的 value 可以直接委托给父类 AutoStorage 描述符类直接存储了**。
5. 设置 validate 方法为抽象方法、**其由子类来覆盖它**。
6. Quantity 描述符类重写了 validate 方法，**验证值是否大于零。**
7. NonBlank 描述分类重写了 validate 方法，**验证值是否为空。**

如此一来 Fruits 类的方法体中，只需要增加一行代码即可：

```python
class Fruits:
    description = NonBlank("description")
    weight = Quantity("weight")
    price = Quantity("price")
    
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

上述的诸多描述符类通常放在单独的 model 模块中，以供多个模块共同使用。

例如，产品经理小姐姐有一天和你说，甲方也想卖酸奶，需要给酸奶写一个类；**那么此时 model 模块中的 NonBlank 和 Quantity 也能够提供给酸奶类使用了。**

> 这就是设计模式的魔力，其能够减少大量的代码冗余

若我们将诸多描述符类放在单独的 model 模块中、那么 Fruits 代码看起来会是这样：

```python
import de_model as model
class Fruits:
    description = model.NonBlank("description")
    weight = model.Quantity("weight")
    price = model.Quantity("price")
... 以下省略 ...
```

如果你学过 Django，那么你会意识到这和 Django ORM 中的 models.TextField() 用法一致。其实 Django 的 models.TextField() 就是通过属性描述符来实现的。

> models.TextField() 不需要传入托管属性名。**其原理是类装饰器**，我会在后几篇博文中提到。

# 总结
使用 python 特性能够很好的管理需要特殊存储逻辑的实例属性。

但是、当大量的属性都需要同样的存储逻辑时、单纯的使用 property 依旧会引起代码冗余。
**此时、应该考虑是使用属性描述符还是实现特性工厂函数**来解决这个问题。

我给出的建议是：在这种情况下，**尽量使用属性描述符、因为你不知道后续会不会有类似但又不同的属性存取逻辑。**（例如本博文中的 description 和 weight）

使用属性描述符比之特性工厂有着很大的优势、因为其是类，可以实现众多的面向对象的设计模式。

你可能已经注意到，在本博文中，对于 property 的描述，从来都是**一种**属性描述符。那么，**除了本博文描述的属性描述符，还有其他类型的属性描述符吗？它们又拥有怎样的特性？**下一篇博文将会解答此问题。

