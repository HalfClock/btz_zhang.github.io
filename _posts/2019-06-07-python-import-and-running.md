---
layout:     post
title:      "python 导入时与运行时"
subtitle:   "解密 python import 的特殊之处"
date:       2019-06-07 11:59:59
author:     "Btz"
header-img: "img/bg-python-import-and-running.jpg"
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

# Preview

Python 导入时和运行时的概念不像 Java 语言一样易于区分。

Java 的 import 只是告诉 Java 编译器需要特定的包，**而 Python 则不同**，Python 的首次 import 会 “执行所有的顶层代码” **甚至可以做所有运行时能够做到的事。**

这篇文章是为了说明白 Python 导入时与运行时的行为差异，并总结出一些经验准则。

将这篇文章纳入元编程系列文章的原因是：**了解了这一部分概念以后能够帮助我们理解元编程的相关概念。**

# 导入时与运行时的概念

Python 有两种**执行代码**的方式：

1. 通过命令行方式执行 Python 脚本。**(运行时)**
2. 将代码从一个文件首次导入到另一个文件/解释器中。**(导入时/import)**

**这两种方式分别对应着导入时与运行时。** 也就是说，我们接下来要弄明白的是：

假设我们有一个编写好的 python 脚本(test.py)，那么我们在使用命令行使用命令 `python  test.py` 和 `import test.py` **分别会发生什么**，并尝试总结出有用的经验准则。

# 测试代码

为了能够看清首次导入时究竟会发生什么，我们需要提供一个有着**类、函数、普通顶层代码**的相对复杂的 `evaltime.py` 文件，还需要在这个文件中再导入另一个 `evalsupport.py` 文件，**用以观察在各种不同条件下的行为**，`evaltime.py`代码如下：

>以下代码都是《流畅的 python》第二十一章第三节的源码，作者 Luciano Ramalho 也是通过这两个类来总结导入时与运行时的行为差异，我觉得设计的很好。

```python
from evalsupport import deco_alpha

print('<[1]> evaltime module start')

class ClassOne():
    print('<[2]> ClassOne body')

    def __init__(self):
        print('<[3]> ClassOne.__init__')

    def __del__(self):
        print('<[4]> ClassOne.__del__')

    def method_x(self):
        print('<[5]> ClassOne.method_x')

    class ClassTwo(object):
        print('<[6]> ClassTwo body')

@deco_alpha
class ClassThree():
    print('<[7]> ClassThree body')

    def method_y(self):
        print('<[8]> ClassThree.method_y')

class ClassFour(ClassThree):
    print('<[9]> ClassFour body')

    def method_y(self):
        print('<[10]> ClassFour.method_y')

if __name__ == '__main__':
    print('<[11]> ClassOne tests', 30 * '.')
    one = ClassOne()
    one.method_x()
    print('<[12]> ClassThree tests', 30 * '.')
    three = ClassThree()
    three.method_y()
    print('<[13]> ClassFour tests', 30 * '.')
    four = ClassFour()
    four.method_y()

print('<[14]> evaltime module end')
```

`evalsupport.py`代码如下：


```python
print('<[100]> evalsupport module start')

def deco_alpha(cls):
    print('<[200]> deco_alpha')

    def inner_1(self):
        print('<[300]> deco_alpha:inner_1')

    cls.method_y = inner_1
    return cls


class MetaAleph(type):
    print('<[400]> MetaAleph body')

    def __init__(cls, name, bases, dic):
        print('<[500]> MetaAleph.__init__')

        def inner_2(self):
            print('<[600]> MetaAleph.__init__:inner_2')

        cls.method_z = inner_2


print('<[700]> evalsupport module end')
```

在 `evaltime.py` 中有着：
1. 像 `print('<[1]> evaltime module start')` 一样的**顶层代码**。
2. 像 ClassOne、ClassTwo、ClassThree、ClassFour 的**类定义体**；**ClassOne、ClassTwo 还是嵌套关系**。
3. 在类中有诸多的类方法，甚至 ClassOne 类还定义了 **`__init__`，`__del__` magic 方法。**
4. **ClassFour 继承了 ClassThree**
5. ClassThree 有一个**类装饰器** `deco_alpha`，来自于另一个模块 — `evalsupport.py`。

在 `evalsupport.py` 中有着：
1. 像 `print('<[100]> evalsupport module start')` 一样的**顶层代码**。
2. 一个函数定义 `deco_alpha`，其嵌套这一个函数 `inner_1`。(此函数是类装饰器函数)
3. 有一个类定义 `MetaAleph`，其中 `__init__` 有一个嵌套函数 `inner_2`

这两个函数基本已经**囊括了 python 代码的所有应用场景**、包括顶层代码、类、函数、magic 方法、嵌套类、嵌套函数、类继承、装饰器、导入模块。

接下来我们将观察在**首次导入时**与**运行时** python 分别会做出什么动作。

>Luciano Ramalho 建议你在看下去前先自己拿纸和笔将自己认为的程序输出写在纸上，我也是这么做的，第一次这么做让我明白了自己在认知上的差错。

# 导入时解析
在 **python 命令行**中运行 `import evaltime.py` 会得到以下输出：

```python
<[100]> evalsupport module start
<[400]> MetaAleph body 
<[700]> evalsupport module end 
<[1]> evaltime module start 
<[2]> ClassOne body 
<[6]> ClassTwo body 
<[7]> ClassThree body 
<[200]> deco_alpha 
<[9]> ClassFour body 
<[14]> evaltime module end
```

下面对上面的 10 行代码做出解释：

1. 导入时，python **会执行 `evaltime.py` 所导入的 `evalsupport.py` 模块中的所有顶层代码。**
2. `evalsupport.py` 中的 deco_alpha 方法体没有执行，**实际上 python 编译了 deco_alpha 函数，但是不会执行定义体**。**MetaAleph 类的定义体运行了**，但是其中的 `__init__` magic 方法没有执行。
3. 原因见 1
4. python 会**执行 `evaltime.py` 的所有顶层代码。**
5. ClassOne 的类定义体也执行了、但是类方法没有执行。
6. **嵌套于 ClassOne 的 ClassTwo 类的定义体也执行了。** 
7. ClassThree 的**类定义体先于它的类装饰器方法体执行了。**
8. **类装饰器 deco_alpha 在被其装饰的 ClassThree 后执行了方法体**，但是嵌套其中的函数 `inner_1` 没有执行。
9. ClassFour 类执行了类方法体，但是貌似与其父类 ClassThree 没有关联。
10. 原因见 4

**值得注意的是：这些输出只会在首次 import 时看见**，之后再次 import 就不会有任何输出了。

**原因是：**在首次导入时，解释器会从上到下一次性解析完 .py 模块的源码，然后生成用于执行的字节码，并将其存入 `__pycache__` 的 .pyc 文件中。如果有句法错误，则会在此时报告。

当本地的 `__pycache__` 文件中有最新的 .pyc 文件，那么解释器则会跳过上述步骤，因为已经有运行所需的字节码了。

#### 导入时总结:
在 python 命令行**首次输入** `import A` 时会按照**代码顺序**执行以下几条：

1. **会执行模块 A 中的所有顶层代码。**
2. **会执行模块 A 导入的所有模块的顶层代码(上例中的 evalsupport.py)。**
3. **执行的顶层代码中**：
    1. **若是类的定义体，那么会执行类的定义体、但是只会编译类方法，不会执行类方法定义体。**
    2. **若是函数，则解释器只会编译此函数，不会执行函数定义体。**
4. **在执行的类的定义体中**：
    1. **若是嵌套类**，那么会执行嵌套类的定义体，规则与第三条规则中的类定义体情况一致。
    2. 若该类被装饰、**在执行完类定义体后执行装饰器函数定义体。**
    3. 与该类是否是某个类的子类无关（正常执行）。
5. **执行装饰器函数定义体中**：
    1. **若是嵌套函数，那么只会编译此函数，不会执行该函数定义体。**

总结来说、首次导入时，所有的顶层代码都会执行，包括在此模块中导入的其他模块的顶层代码；此外，类的定义体也会被执行；除了装饰器函数的定义体会在被装饰的类/函数执行过后执行，**其他的所有函数只会被解释器编译，不会执行函数定义体。**

>`if __name__ == ‘__main__’` 属于顶层代码，只是此时这个语句的结果为 False，原因后面会提到。

>被编译器编译的意思是：将其解析成可执行的字节码，换言之**就是做了全局名称绑定，以便以后要使用时找到它。**

>对于类而言，被编译器编译后，定义了类的属性和方法，并构建了类对象。

# 运行时解析
我们直接来看在**系统命令行**运行 `python evaltim.py` 会发生什么：


```python
<[100]> evalsupport module start
<[400]> MetaAleph body
<[700]> evalsupport module end
<[1]> evaltime module start
<[2]> ClassOne body
<[6]> ClassTwo body
<[7]> ClassThree body
<[200]> deco_alpha
<[9]> ClassFour body
<[11]> ClassOne tests ..............................
<[3]> ClassOne.__init__
<[5]> ClassOne.method_x
<[12]> ClassThree tests ..............................
<[300]> deco_alpha:inner_1
<[13]> ClassFour tests ..............................
<[10]> ClassFour.method_y
<[14]> evaltime module end
<[4]> ClassOne.__del__
```
下面对上面的代码作出解释：

1. **第 9 行及之前的输出与首次导入时的输出一致。**
2. 第 10 行开始运行 `if __name__ == '__main__':` 之后的代码
3. 11 行是类初始化的标准行为 —— 调用 `__init__`
4. 第 14 行值得注意，其 method_y 没有输出类定义体中的 method_y ,而是执行了**被类装饰器**使用猴子补丁替**换掉的方法 -> inner_1 的。**
5. 第 16 行也值得注意，装饰器对 ClassThree 子类 ClassFour 好像没有影响。
6. 第 18 行，只有在程序结束以后，ClassOne 类的实例才会被垃圾回收程序垃圾回收。

**值得注意的是，不管调用多少次 `python evaltim.py` 输出的结果都一致。**

#### 运行时总结

运行时的执行规则与导入时基本一致：

1. 在系统命令行调用 `python evaltim.py` 与在 python 命令行调用 `import evaltime` 的**执行顺序和执行规则一致**。
2. 不同的是，运行时会执行 `if __name__ == '__main__':` 代码块的代码。

值得一提的是，如果你将 `if __name__ == '__main__':` 代码块的代码**全部都变成顶层代码**，也就是去掉 `if __name__ == '__main__':` 这一行，**那么运行时的输出与首次导入时的输出几乎一致，只是最后一行销毁 ClassOne 实例的代码不会输出。**

这是因为导入时发生在 python 命令行中，在这里 python 程序在 import 完 evaltime 后还没有结束，不会触发垃圾回收机制。

那么实际上，**python 导入时与运行时的重大差别在于** `if __name__ == '__main__':` 代码块的代码是否执行。

接下来我们来探究导入时与运行时特殊变量 `__name__` 的情况。

# 运行时与导入时的差别

现在，我们将 evaltime.py 中的所有代码替换成一句：

```python
print("此时 __name__ 变量的值是：", repr(__name__))
```

> repr() 只是将 `__name__` 变量的值以字符串的形式返回，不必在意。

在 python 命令行中执行 `import evaltime`，得到如下结果：


```python
此时 __name__ 变量的值是： 'evaltime'
```

在系统命令行中执行 `python evaltime.py` 得到如下结果：


```python
此时 __name__ 变量的值是： '__main__'
```

那么由此可见、运行时与导入时最大的不同在于 `__name__` 变量的值，在**导入时** python 解释器会为 `__name__` 赋值为**模块名（evaltime）**。在运行时 python 解释器会为 `__name__` 赋值为 `__main__`。

其次、在系统命令行运行 `python evaltime.py` 不管多少次输出结果都是一致、而在 python 命令行中运行 `import evaltime` 只会在第一次输出结果、后面的重复调用只会检查 `__pycache__` 是否有最新的 .pyc 文件，如果有，则什么也不做。

也就是说、python 导入时与运行时的动作几乎一致，都会按照**相同的规则执行模块文件中的所有顶层代码。**但是由于在导入时与运行时 python 解释器为 `__name__` 变量赋值不同，**所以我们能够为运行时添加一些导入时所不能做到的操作**，比如定义 Python 的 "Main 函数"。

# 定义 Python 的 “Main 函数”

>如果你学过 python ，你一定见过 `if __name__ == '__main__':` 这句代码，看完上面的文字，你一定已经明白了其原理。

**C 语言有一个特殊函数的 main 函数**、当操作系统运行 C 程序时会自动执行该函数。

像 C 语言一样有 main 函数的语言不在少数、但是 python 不在其列，python 解释只会想上面的小节描述的一样从头到尾开始执行顶层代码，没有自动执行的特殊函数。

**所以、根据执行方式的不同，是否有着不同行为就相当重要了。**好在，python 解释器在导入时与运行时为特殊变量 `__name__` 赋值不同，这提供了 python 拥有 Main 函数的方法。

首先，我们必须意识到，`if __name__ == '__main__':` 下的代码块**只会在运行时执行**。
其次，我们需要像其他语言使用 Main 函数一样使用它，即：

1. **将大部分的代码放入模块的函数或者类中。**
2. **创建名为 main() 的函数来包含运行时想要执行的代码**，包括调用函数和生成类对象。
3. 在 `if __name__ == '__main__':` 代码块下调用 main()。

使用上面的准则来定义 Python 的 Main 函数能够**极大程度的解决 python 导入时和运行时动作基本一致的弊端。**（例如:导入时可能执行耗时操作、扰乱终端信息等）

# End
本文中，我总结了 python 导入时与运行时的区别、**指出了导入时会按照一定规则执行所有顶层代码**，紧接着我们实验出了 **python 运行时的行为与导入时相当一致**。

接下来、我分析了导入时与运行时的重点差别 —— **解释器会为 `__name__` 属性赋不同的值**。

根据这个的性质，**我引出了 Python Main 函数的使用准则**、**使用这些准则能够很大程度的避免python 导入时和运行时动作基本一致导致的弊端。**




