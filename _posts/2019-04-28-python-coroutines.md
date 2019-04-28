---
layout:     post
title:      "理解“狭义”的 python 协程"
subtitle:   "从 yield 到 yield from"
date:       2019-04-28 18:30:00
author:     "Btz"
header-img: "img/bg-python-coro.jpg"
catalog: true
tags:
    - python
---

> 这篇博文讲述的 python 协程是**不正式的、宽泛的协程**，即通过客户调用 .send(...) 方法发送数据或使用 yield from 结构驱动的生成器函数，**而不是 asyncio 库采用的定义更为严格的协程。**

# 前言
在**事件驱动型编程**中，协程常用于离散事件的仿真（在单个线程中使用一个主循环驱动协程执行并发活动）。

协程通过显式**自主地把控制权让步给中央调度程序**从而实现了*协作式多任务*。

所以，**协程是 python 事件驱动型框架和协作式多任务的基础。**

那么，弄明白协程的**进化过程**、基本行为和**高效的使用方式**是很有必要的。

本博文想要解释清楚 python 协程的基本行为以及如何高效的使用协程。

> 在阅读本文之前，你必须要了解 python 中 yield 关键字、和生成器的基本概念。如果你还不知道这两个概念是啥，你可以看我的上一篇博文：[浅析 python 迭代器与生成器](https://halfclock.github.io/2019/04/20/python-iterator-and-generator/) 或者通过[ CSDN 上冯爽朗的博文](https://blog.csdn.net/mieleizhi0522/article/details/82142856) 简单了解 yield 关键字的使用方法。

# 从生成器到协程
> 协程是指一个过程，这个过程与调用方协作，即**根据调用方提供的值**产出相应的值**给调用方。**

从协程的定义来看，协程的部分行为和带有 yield 关键字生成器的行为类似，因为调用方可以使用 .next() 方法**让生产器产出值给调用方。例如，这个斐波那契生成器函数：**


```python
>>> def fibonacci():
        a, b = 0, 1
        while True:
            yield a
            a, b = b, a + b
```
调用方调用 next（）函数可以**获取它的产出值**：
```python
>>> f = fibonacci()
>>> print(next(f))
0
>>> print(next(f))
1
```
这么看来，**生成器的行为离协程的行为就差一步，即接收调用方提供的值。**

在 python 2.5 后 yield 关键字就可以在表达式中使用了，而且生成器 API 中增加了 .send(value)方法。**生成器的调用方可以使用 .send(...) 方法给生成器发送数据。**

这样一来生成器就可以接收调用方提供的值了，**其接收的数据会成为 yield 表达式的值。**

例一是一个简单的例子，来说明调用方如何发送数据及生成器如何接受数据。

```python
>>> def coroutine():
        print('-- 协程开始 --')
        x = yield 'Nothing'
        print('-- 协程接收到了数据: {!r} -- '.format(x))

>>> coro = coroutine()
<generator object coroutine at 0x10bbb2408>
>>> next(coro)
-- 协程开始 --
Nothing
>>> coro.send(77)
-- 协程接收到了数据: 77 --
Traceback (most recent call last):
...
StopIteration
```
上面的例子表明：
1. 在协程中，**yield 通常出现在表达式的右边。**
2. 调用方先使用一次 .next() 执行 yield ‘Nothing’ 让协程产出字符串 “Nothing” 并**悬停至至 yield 表达式这一行**
3. 调用方使用 .send() 发送数据给协程。
4. 发送的**数据代替 yield 表达式**，并赋给变量 x。
5. **协程结束时与生成器一致，都会抛出 StopIteration 异常。**

需要特别注意的地方有：
**首先、调用方只有在协程停在了 yield 表达式时，才能调用 .send() 发送数据**，否则，协程会抛出 TypeError 异常，如例二：

```python
>>> coro = coroutine()
>>> coro.send(77)
Traceback (most recent call last):
 ... in coro.send(77)
TypeError: can't send non-None value to a just-started generator
```
>悬停在 yield 表达式的协程状态是 `GEN_SUSPENDED` ，你可以使用`inspect.getgeneratorstate(...) `函数确定协程的状态。

**其次、调用方使用 .send(y) 发送的数据会代替协程中的 yield 表达式**，在上例中,发送的数据 y 是 77 ,77 代替了 yield 表达式，并赋给了变量 x。

**最后、当赋值完毕后、协程会继续前进至下一个 yield 关键字并悬停**，直至结束从而抛出 StopIteration 异常。

你可以把 .send( y ) 看做两个部分的结合，即:
1. yield 表达式 = y
2. .next()

这样一来，拥有 .send（）方法的生成器，完全符合了协程的定义，它可以通**过 .send() 接受调用方传递的值，并且可以通过 yield 产出值给调用方。**

不过，此时我们没有办法在一创建协程时，立马使用它。

你必须要先使用一次 .next() 让协程悬停在 yield 表达式那一行，从而使协程转变至 `GEN_SUSPENDED 状态`。这样的行为被称作**预激协程。**

# 预激协程
>毫无疑问，预激协程是一个很容易被遗忘的步骤。
>需要使用 .send() 发送数据之前还必须使用一次 .next()，这让人感到厌烦。

我们有什么办法能够自动预激协程呢？

有一种方法是使用能够提前调用一次  .next() 的装饰器，如下面这个 coroutine 装饰器：

```python
# BEGIN CORO_DECO
>>> from functools import wraps

>>>def coroutine_deco(func):
        """Decorator: primes `func` by advancing to first `yield`"""
        @wraps(func)  #使用 functools.wraps 装饰器获得源 func 的所有参数 "*args,**kwargs"
        def primer(*args,**kwargs): 
            gen = func(*args,**kwargs) #使用源生成器函数获取生成器
            next(gen) #调用 .next 方法
            return gen #返回调用 .next 方法后的生成器
        return primer
    # END CORO_DECO

```
>网上有多个类似的装饰器。这个改自 ActiveState 中的一个诀窍——[Pipeline made of coroutines](http://code.activestate.com/recipes/578265-pipeline-made-of-coroutines/)，作者是 Chaobin Tang，而他是受到了 David Beazley 的启发。—— 《流畅的 python 》

使用这个装饰器后，现在我们再运行例二的代码就不会报 TypeError 异常，而是会正常运行了，如下：

```python
@coroutine_deco
>>> def coroutine():
        print('-- 协程开始 --')
        x = yield 'Nothing'
        print('协程接收到了数据: {!r}'.format(x))

>>> coro = coroutine()
-- 协程开始 --
>>> import inspect
>>> inspect.getgeneratorstate(coro)
GEN_SUSPENDED
>>> cro.send(77)
协程接收到了数据: 77
Traceback (most recent call last):
...
StopIteration
>>>inspect.getgeneratorstate(coro)
GEN_CLOSED
```
该例子有如下行为需要注意：
* 在创建协程 coro 对象后，直接输出了 “-- 协程开始 --” 字符串，这表明，**在创建协程对象后，其自动调用了一次 next() 方法。**
* 使用 `inspect.getgeneratorstate` 查看协程的状态，发现其已经是 `GEN_SUSPENDED` 状态，**说明协程内部已经悬停在 yield 关键字处。**
* 能够直接调用 .send() 方法而不用事先使用 .next() 了。
* 协程结束时的状态是 `GEN_CLOSED`

>协程还有一个很常用的方法 —— .close() 用于提前关闭协程。使用该方法后，协程会在 yield 表达式那一行抛出 GeneratorExit 异常。

有时，我们需要协程在结束了所有工作时，返回一个值，**这在 python 3.3 之前是不可能的，因为在协程的方法体中写 return 关键字会报句法错误。**

# 让协程在终止时返回值

我们可以在 python 3.3 及之后的版本中**让终止的协程返回想要的值**，只是获取返回值的方法比较曲折。

下面的例三，定义了一个动态计算平均值的协程，并让其在结束工作（接受到 None 值）后**返回一个元组**，该元组保存着目前为止收到的数据个数以及最终的平均值。

```python
>>> from collections import namedtuple

>>> Result = namedtuple('Result', 'count average')

>>> def averager():
        total = 0.0
        count = 0
        average = None
        while True:
            term = yield average
            if term is None:
                break  
            total += term
            count += 1
            average = total/count
        return Result(count, average)  
        
```
该函数有以下行为：

```python
>>> coro_avg = averager()
>>> next(coro_avg) # <1>
>>> coro_avg.send(10)  # <2>
10.0
>>> coro_avg.send(30)
20.0
>>> coro_avg.send(6.5)
15.5
>>> coro_avg.send(None)  # <3>
Traceback (most recent call last):
...
StopIteration: Result(count=3, average=15.5)
```
注释：  
① : 手动预激协程。  
② : 调用 .send(10) 返回目前传入所有数的平均值10、之后每传入一个数都能实时计算所有数的平均值。  
③ : 传入 None ，手动结束该协程。

注意到，和往常一样，结束后**协程抛出了 StopIteration 异常**。不一样的是，**该异常保存着返回的值**，即 Result 对象。

>return 表达式的值会偷偷传给调用方，赋值给 StopIteration 异常的一个属性。这样做有点不合常理，但是能**保留生成器对象的常规行为**——耗尽时抛出 StopIteration 异常。

改造上面的代码，手动捕获异常，获取返回值，可以这样写：
```python
>>> coro_avg = averager()
>>> next(coro_avg) 
>>> coro_avg.send(10)  
10.0
>>> coro_avg.send(30)
20.0
>>> coro_avg.send(6.5)
15.5
>>> try:
        coro_avg.send(None) 
    except StopIteration as exc: 
        result = exc.value
>>> result 
Result(count=3, average=15.5)
```

目前，我们说明了如何让**生成器接收调用方提供的值从而进化成协程**、如何**使用装饰器自动预激协程**、以及**如何从协程获取看起来很有用的返回值。**

**使用协程似乎太麻烦了点 ！**

不是吗？ 为了避免麻烦，我们必须自己定义一个自动预激协程的装饰器，为了获取协程的返回值，我们还必须捕捉异常，并获取异常的 value 属性。

有什么办法能够消除这些麻烦呢?（不用自定义预激装饰器也不用捕获异常以获得返回值）

**在 python 3.3 以后，有一个新的句法能够帮助我们解决这些麻烦，即 yield from**

# yield from 及其工作原理

使用 yield from 关键字**不仅能自动预激协程**、**自动提取异常的 value 属性返回值作为 yield from 表达式的值**，还能够**作为调用方和协程之间的通道**。

如果将例三中的 averager() 改编成使用 yield from 关键字来实现，会是例四的代码：

```python
>>> from collections import namedtuple

>>> Result = namedtuple('Result', 'count average')

>>> def averager():
        total = 0.0
        count = 0
        average = None
        while True:
            term = yield average
            if term is None:
                break  
            total += term
            count += 1
            average = total/count
        return Result(count, average)  

>>> result = set() # <1>

>>> def yf_averager(result): # <2>
        while True: # <3>
            r = yield from averager() # <4>
            result.add(r)
            
>>> yfa = yf_averager(result) # <5>
>>> next(yfa)  # <6>
>>> yfa.send(10) # <7>
10.0
>>> yfa.send(30)
20.0
>>> yfa.send(6.5)
15.5
>>> yfa.send(None) # <8>
>>> result  # <9>
{Result(count=3, average=15.5)}
```
在例四中，averager() 方法并没有做任何改变  
解释：  
①：创建 result 集合以在调用方收集结果。  
②：yield from 关键字的**载体函数**，有时也叫“委派生成器” ，设立这一函数是因为**在函数外部使用 yield from（以及 yield）会导致句法错误。**  
③：使用循环以保证传入 None 时 **yf_averager 生成器不抛出 StopIteration 异常**从而直接结束整个程序，若是如此，我们便观察不到 result 了。  
④：使用 yield from 关键字后面是**协程**、前面是接收协程最终返回值的变量 r，这个 r 我们最终会放在全局变量 result 集合中。还有一点需要注意、**当函数体重含有 yield from 那么它本身就是协程了**。  
⑤：新建 yf_averager 协程，以**建立调用方与 averager 协程的通道**  
⑥：预激 yf_averager 协程  
⑦：使用 .send（）发送数据  
⑧：发送 None 以结束 averager 协程  
⑨：展示 result 集合中的值，确认接收到了最终的结果  

上面如果上面这个例子你不怎么看得懂，没关系，我会在后面解释。  
你现在**只需要知道 yield from 有这些行为：**

1. 在例四中，我们没有预激 averager 协程，但是它能够正常工作。**这说明 yield from 关键字会自动预激协程。**
2. 调用方使用委派生成器 yf_averager 传入的值会送到 averager 里，并且调用方可以接收到 averager 协程处理后返回的值。**这说明了使用 yield from 的委派生成器 yf_averager 可以在调用方和协程之间建立通道，传输数据。**
3. 在获取 averager 结果时，我们没有捕获异常，而是在第 22 行代码中将返回值直接赋给了变量 r。**这说明了协程的最终返回值会成为 yield from 表达式的值。**

-------
#### yield from 关键字的原理

接下来这段伪码等效于 **RESULT = yield from EXPR 语句**。它能够帮助你理解例四中 yield from 的行为
>这并不是完整的伪代码，它去除了 .throw（）和 .close（）方法，只处理 StopIteration 异常。完整的伪码在这里 -> [yield_from_expansion](https://github.com/HalfClock/example-code/blob/master/16-coroutine/yield_from_expansion.py)，不过在理解其功能的方面上，这足够了。

```python
_i = iter(EXPR)  # <1>
try:
    _y = next(_i)  # <2>
except StopIteration as _e:
    _r = _e.value  # <3>
else:
    while 1:  # <4>
        _s = yield _y  # <5>
        try:
            _y = _i.send(_s)  # <6>
        except StopIteration as _e:  # <7>
            _r = _e.value
            break

RESULT = _r  # <8>
```

解释：    
① ：EXPR 可以是任何可迭代的对象，因为获取迭代器 _i（这是子生成器，例子中的 averager 协程）使用的是 iter() 函数。  
② ：**预激子生成器（averager 协程）；结果保存在 _y 中，作为产出的第一个值。**  
③ ：如果抛出 StopIteration 异常，**获取异常对象的 value 属性，赋值给 _r**——这是最简单情况下的返回值（RESULT）。  
④ ：运行这个循环时，委派生成器（yf_averager 生成器）会阻塞，**只作为调用方和子生成器之间的通道**。  
⑤ ：**产出子生成器当前产出的元素；等待调用方发送 _s 中保存的值。**因为这一个 yield 表达式和 ⑥ 中的send()，**委派生成器也变成了协程。**  
⑥ ：尝试让子生成器向前执行，**转发调用方发送的 _s**。  
⑦ ：如果子生成器抛出 StopIteration 异常，**获取 value 属性的值，赋值给 _r**，然后退出循环，让委派生成器恢复运行。  
⑧ ：**返回的结果（RESULT）是 _r**，即整个 yield from 表达式的值。

> 以上的伪代码和注释，几户原封不动的搬了《流程的 python 》里的解释，我只是增加了一些注释。因为我想不出如何更好的总结 yield from 关键字的原理。

注意，因为 yf_averager 是带 yield 关键字的生成器，所以在 ⑧ 结束后，**若找不到下一个 yield 关键字，那么 yf_averager 生成器会抛出 StopIteration 异常**，这是我在例四中设立 while 循环 ③ 的直接原因。

>我建议你在看懂这段伪代码的基础上再去**回顾例四**，这下你**应该豁然开朗**了。如果还看不懂的话，我建议你多花些时间去看《流程的 python 》的第十六章，该章用了60多页的篇幅把 python 协程讲得很通透。

# 结语
本篇博文中，我用了四个小节叙述了我理解中的协程、及其使用技巧。在一开始，我讲述了**协程是什么**，及**如何在 python 2.2 及以后的版本中用生成器构建协程**；然后我讲述了**协程的必要操作（预激）的自动化方法**和**如何在 python 3.3 及以后的版本中获取协程的返回值**；最后，我讲述了方便的 **yield from 关键字的用法、行为**以及**它的主要原理**。

如果你想要知道**协程的具体用处**，《流程的 python 》的第十六章中举了一个离散事件仿真的例子——**出租车队运营仿真**。该仿真程序会创建几辆出租车，并模拟他们并行运作（离开仓库、寻找乘客、乘客下次、四处徘徊、回家）。对于说明如何使用协程做离散事件仿真是一个很好的例子。

>这是那个出租车队运营仿真例子的源码 -> [taxi_sim](https://github.com/HalfClock/example-code/blob/master/16-coroutine/taxi_sim.py)

我希望你看完这篇博文后能够有所收获、如果你看到了一些错误，请在评论中指出。

