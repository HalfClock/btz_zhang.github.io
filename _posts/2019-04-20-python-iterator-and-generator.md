---
layout:     post
title:      "浅析 python 迭代器与生成器"
subtitle:   "深入理解 python 迭代相关流程"
date:       2019-04-20 11:30:00
author:     "Btz"
header-img: "img/bg-python-iandg.jpg"
catalog: true
tags:
    - python
---

# 要这篇博文有何用？
这篇博文是用于帮助理解 **python 可迭代对象、迭代器与生成器的**，你在阅读后应该能够比较清晰的理解 python 中迭代相关的概念与流程。

这篇博文能够解答： 
1. 在 python 中究竟什么是**迭代**？
2. 什么是**可迭代的对象**，为什么 python 的序列类型的对象均可迭代？
3. **迭代器**是啥？它和可迭代对象有什么关联？
4. **生成器**又是啥？
5. **生成器和迭代器有什么区别？**


# 迭代的概念简述

> 循环就是迭代吗？
> 
> 答：不是，但是迭代与循环有着千丝万缕的联系。

迭代是一个做有限次或者 “无限次” **重复动作的过程**、在这个过程里上一次重复动作的**结束状态**是下一次重复动作的**开始状态。**每一次重复都可以称之为一次迭代。

相比于单纯的循环、迭代有一个**额外的限制条件** —— 必须存在着**记录状态**的记录员，用来保存上一次迭代的结束状态。

下面这个代码说明了循环和迭代的区别，其中模拟的迭代过程中的变量 i 就是记录员：

```python
#单纯的循环
while True:
print('This is Loop')

#模拟的迭代过程
i = 0
while True:
    print('This is iter of No.' + str(i))
    i += 1
```

结果：
```python
#单纯的循环
This is Loop
This is Loop
This is Loop
...
...
```

```python
#模拟的迭代过程
This is iter of No.0
This is iter of No.1
This is iter of No.2
...
...
```

# 可迭代对象
python 的 for 循环由于脸盲，所以**只认识可迭代对象（Object of Iterable）**，并根据可迭代对象**提供的服务**构建整个循环的过程。

作为 python 最基础的单元，操作for 循环的过程大家一定都熟悉，其中最简单、也最常用的操作的便是遍历列表对象了，如下代码所示：

```python
list_obj = [1,2,3,4,5,6]

for i in list_obj:
    print(i)
```

代码虽然简单、但是必须注意到这几点：
1. **这是一个迭代的过程：**每一次重复， “i 记录员” 都根据上一次迭代的结束位置（list的索引位）向后移了一位。
2. for 循环是怎么认识 list 类型对象的？它不是只认识可迭代对象吗？

#### 可迭代对象如何被识别

for 循环能够认识 list 类型对象的原因很简单、因为列表对象实现了序列协议、而**所有的序列对象均可迭代。**

**但是，可迭代对象又不仅仅局限于序列对象。**

下面说明 for 循环**如何判断对象是否可以迭代**：

1. python 解释器在需要迭代一个对象（obj_i）时，会调用内置函数 iter（obj_i）方法获取该对象的迭代器。

2. 如果 iter（obj_i）抛出了异常或者返回的对象不是迭代器，那么 obj_i 就不可迭代。

若将列表对象换成整数对象，则不可迭代，因为 iter（obj_i）会抛出TypeError异常 

运行：
```python
list_i = [1,2,3]

print(iter(list_i))

for i in list_i:
    print(i)
```
结果
```python
<list_iterator object at 0x107217940> #调用iter（obj_i）返回列表迭代器
1
2
3
```
运行：
```python
a = 1
iter(a)
```
结果：
```
Traceback (most recent call last):
  File "....", line 22, in <module>
    iter(a)
TypeError: 'int' object is not iterable
```
运行：

```python
a = 1
for i in a:
    print(i)
```
结果和直接调用iter（a）一致



> for 循环使用可迭代对象提供的服务——迭代器构建整个循环，迭代器具体是什么在下文说明

可以看出、在判断对象是否迭代上、内置函数 iter () 方法是**是否返回迭代器**最关键的部分，下面介绍它的工作流程：

1. 检查对象**是否实现了 “__iter__” 方法**，如果实现了就调用它，并获取一个迭代器。
2. 如果**没有实现 iter 特殊方法，但是实现了 getitem 特殊方法**， Python 会创建一个迭代器，尝试按顺序**（从索引 0 开始）**获取元素。
3. 如果尝试失败，Python 抛出 TypeError 异常，通常会提示“C object is not iterable”（C 对象不可迭代），其中 C 是目标对象所属的类。

>如果使用getitem特殊方法构建不出迭代器，那么是因为其没有从零开始索引

下面我构建两个自定义的可迭代类，用来帮助你理解上面这段话。

>这俩个例子来自《流畅的 python 》第十四章


**例一，实现了序列协议的对象可迭代**
```python
import re
import reprlib

RE_WORD = re.compile('\w+')


class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)  # ①

    def __getitem__(self, index):
        return self.words[index]  # ②
        
```

**①：**re.findall 函数返回一个字符串列表，里面的元素是正则表达式的**全部非重叠匹配。**

**②：**该类实现了 getitem 特殊方法，用于**返回从零开始索引的索引位对应的单词。**

运行如下代码：
```python
s = Sentence('"The time has come," the Walrus said,')

print(iter(s)) # ③

for word in s:  # ④
    print(word)
```

结果如下：
```python
<iterator object at 0x105667898> 
The  
time
has
come
the
Walrus
said
```
**③：** 调用 iter 方法，该方法使用Sentence 类的 getitem 特殊方法构造了这个迭代器对象

**④：** 可以正常的使用for循环

**例二、实现了 iter 特殊方法返回迭代器的类，其对象可迭代**


```python
import re
import reprlib

RE_WORD = re.compile('\w+')


class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)

    def __iter__(self):
        return iter(self.words)  # ①

```
**①：** 没有实现 getitem 特殊方法，而实现了 iter 特殊方法，并调用内置的 iter 函数返回迭代器

运行如下代码：
```python
s = Sentence('"The time has come," the Walrus said,')

print(iter(s)) # ②

for word in s:
    print(word)
```

结果如下：
```python
<iterator object at 0x105667898> 
The
time
has
come
the
Walrus
said
```

**②：** iter（）方法使用了 Sentence 类的 iter 特殊方法构造了这个迭代器对象


所以，在 python 中判断对象是否是可迭代对象，**_最准确的方法_** 就是调用内置的 iter(obj) ,如果能够获取到迭代器，那么该对象可迭代。

iter（）的工作流程解释了为何序列对象均是可迭代对象的原因，因为**序列对象均可以使用下标获取对应值，即实现了"从零开始索引"的 “__getitem__( )” 方法**

那么，我们也可以理解为***如果类实现了 “iter( )” 特殊方法 或 "从零开始索引"的 “getitem( )” 特殊方法，那么此类的对象是可迭代对象。***


-------

我们已经讲清楚了什么是可迭代对象，以及判断对象是否可迭代最准确的方法。

接下来，开始说明迭代器对象到底是什么。

# 迭代器
在说明它究竟是个什么样的对象时，我们先来搞明白，它和可迭代对象之间的暧昧关系。
#### 可迭代对象与迭代器
在讲述可迭代对象时，我们多次提到了迭代器，并且，迭代器是都由内置的 iter 方法返回的。

内置的 iter 方法可以从可迭代对象的__iter__方法中**获取迭代器**，也可以利用可迭代对象中的__getitem__方法**构造并取得迭代器**。

从上述的行为来看、他们之间的关系是：**python为了实现迭代，从可迭代对象中获取迭代器**

>还记得迭代的概念吗？我说明了相比于单纯的循环、迭代有一个**额外的限制条件** —— 必须存在着**记录状态**的记录员，用来保存上一次迭代的结束状态。那么，可以理解为，迭代器就是 “隐形的记录员” ，它保存着迭代的状态。

-------
#### 如何自定义一个迭代器
>那么迭代器到底是一个什么样的对象呢？
>
>答：实现了**__next__方法和__iter__方法**的对象。

判断一个对象是否是迭代器，**_最精确的方法_** 就是使用 isinstance() 方法去判断该对象是不是 **collections.abc.Iterator** 抽象基类的子类了，该抽象基类的定义如下图（来自《流程的 python 》第十四章）：

![](/img/in-post/post-python-iterator-and-generator/15557524281862.jpg)

可以发现，该抽象基类有两个特殊函数__next__和__iter__,其中__iter__返回对象自身。

事实上，因为该抽象基类实现了类方法__subclasshook__，所以只要自定义类实现了__next__和__iter__特殊方法，该类的实例就是迭代器了。**此时，就算该类不继承 collections.abc.Iterator 使用 isinstance 也会返回 True。**

>引用《流畅的 python 》中的说法，collections.abc.Iterator 是行为有点像鸭子类型的天鹅类型
>
>如果你不知道什么是鸭子类型，什么是天鹅类型，我建议你去看《流畅的 python 》的第十一章

接下来，我将对上一个小节中实现的 Sentence 类进行改造、让其的__iter__特殊方法返回我们自定义的迭代器。

>很惭愧，这个例子也来自《流畅的 python 》，因为我找不到更好的例子了。

```python
class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)

    def __iter__(self):  
        return SentenceIterator(self.words) # ①

class SentenceIterator:

    def __init__(self, words):
        self.words = words  
        self.index = 0  

    def __next__(self): # ②
        try:
            word = self.words[self.index]  # ③
        except IndexError:
            raise StopIteration()  # ④
        self.index += 1  # ⑤
        return word  # ⑥

    def __iter__(self):  # ⑦
        return self
```
对例子的解释：  
**①：** 在这里__iter__方法没有像上一版一样调用内置的iter()方法获取迭代器，而是返回了自定义的迭代器对象。  
**②:** 自定义的迭代器实现__next__方法。  
**③:** 获取 self.index 索引位上的单词。  
**④:** 如果 self.index 索引位上没有单词，那么抛出 StopIteration 异常。  
**⑤:** 递增 self.index 的值，这就是**记录员本员了**。  
**⑥:** 返回本次迭代获取的单词。  
**⑦** 实现 self.__iter__ 方法，返回自身。  

在本例中，**迭代器类没有继承 collections.abc.Iterator** 只是实现了__next__和__iter__特殊方法。

如果，该类继承了 collections.abc.Iterator 方法，**只需重写__next__即可**，因为这里__iter__的方法体与基类一致。

接下来，我们来测试一下这个自定义的迭代器类：

运行测试代码：

```python
s = Sentence('"The time has come," the Walrus said,')

print(iter(s)) # ①

print("-----------")

for word in s: # ②
    print(word)

print("-----------")
s_iter  = iter(s) # ③

while True:# ④
    try:
        print(next(s_iter))  # ⑤
    except StopIteration:# ⑥
        del s_iter # ⑦
        break # ⑧
```

得到如下结果：

```python
<__main__.SentenceIterator object at 0x10a235908>
-----------
The
time
has
come
the
Walrus
said
-----------
The
time
has
come
the
Walrus
said
```
对例子的解释：   
**①:** 查看获得的迭代器。  
**②:** 使用 for 循环对 Sentence 对象进行迭代。  
**③:** 手动获取迭代器，赋给 s_iter。  
**④:** 使用 while 循环对 Sentence 对象进行迭代。  
**⑤:** 手动调用迭代器 s_iter 的__next__方法。  
**⑥:** 如果没有字符了，迭代器会抛出 StopIteration 异常。  
**⑦:** 释放对 s_iter 的引用，即废弃迭代器对象。  
**⑧** 退出循环。  

在上例中，使用自定义的迭代器**满足了 for 循环的需要**，不仅如此，例子还给出了迭代 Sentence 对象的 while 循环版本。

在本例中、**使用 for 循环和使用 while 循环得到的结果一致、事实上他们的实现流程也基本一样。**

##### for循环的执行流程

1. 在 for-in 循环的版本中、python 会调用 iter() 函数取得 Sentence 对象的迭代器，记为 A 
2. 在**每一次循环中**先调用一遍 next(A),即 `A.__next__()`并将返回值赋给变量 word
3. **以此为基础执行 for 循环体中的代码块**（在本例中是 print(word)）。

#### 迭代器的作用
从设计模式的角度来说、迭代器有两个重要作用：

**1、迭代器为遍历不同的聚合结构提供了统一的接口（支持多态遍历）、且无需暴露该聚合结构的内部表示。**

当我们想要从头开始迭代列表对象时，是相对容易的，因为其的索引是统一的数字（从 0 开始到len - 1 ）,但是如果用索引来遍历字典，便会出错，如下所示：
运行：
```python
li = [6,53,27,3]

for i in range(len(li)):
    print(li[i])

dict = {'a' : 1 ,'b' : 2, 'c' : 3}

for i in range(len(dict)):
    print(dict[i])
```
结果：
```python
6
53
27
3
Traceback (most recent call last):
  File "....", line 44, in <module>
    print(dict[i])
KeyError: 0
```   
在不知道字典内部结构的前提下，使用索引遍历字典是几乎不可能的。但是**如果使用迭代器遍历二者**，那么就不会出错，如下所示：

运行:
```python
li = [6,53,27,3]

for i in li:
    print(i)

print("------")

dict = {'a' : 1 ,'b' : 2, 'c' : 3}

for i in dict:
    print(dict[i])
```
结果：

```python
6
53
27
3
------
1
2
3
```

上面的例子中，for 循环没有尝试从索引遍历他们，而是使用内置的 iter（）获取了 li 和 dict 的迭代器、再通过迭代器来迭代他们，这就提供了一种统一的迭代方法，**只要这个对象是可迭代的，那么就可以使用迭代器进行迭代。**

-------

**2、迭代器支持对聚合对象的多种遍历。**

可迭代对象的__iter__方法是迭代器的工厂，每一次调用它都会返回一个新的独立的迭代器,基于这种行为，我们就可以对可迭代对象进行多种遍历了。

例如：
运行如下代码：
```python
li = [6,53,28,3]

for i in li:
    print(i)

print("-----------")

for i in li:
    print(i+10)
```
结果：

```python
6
53
28
3
-----------
16
63
38
13
```

这是一个极简单的例子、它说明了从同一个可迭代对象获取的每一个迭代器都是独立的。


-------

我已经讲明白了迭代器究竟是什么，它是实现了特殊函数__next__和__iter__类的实例；也讲明白了迭代器与可迭代器之间的关系，python 从可迭代对象中获取迭代器；最后还将了迭代器模式的作用和好处。

接下来，我们将说明生成器，它能够让你**不用自定义迭代器类就可以享受迭代器模式带来的好处**，甚至可以说，**生成器是 python 中内置的迭代器模式**。

# 生成器

> 在大部分的 python 程序员眼中，生成器是优雅的迭代器,从 python 语言的角度来说，这句话是对的。
> 
> 我会在下一节具体说明，在我的理解中生成器与迭代器的区别 

生成器可以用来避免自己实现迭代器类，**从容而优雅的用生成器函数返回生成器从而代替迭代器。**

是的，在接口实现方面，生成器**也实现了特殊函数__next__和__iter__**，所以能够完美的代替迭代器。

#### 生成器对象如何获取

**生成器对象有两种获得方式：**
1. 调用生成器函数，其返回值就是生成器
2. 使用生成器表达式获取生成器对象

使用这两种方式返回的生成器是 types.GeneratorType 类型的，所以判断一个对象是否是生成器，**最准确的方法**应该是使用 isinstance 方法判断是否是 types.GeneratorType 对象。

-------

>生成器函数返回生成器，生成器产出或生成值

**生成器函数其实就是函数体中含有 yield 关键字的函数。**如下例就是一个生成器函数：

```python
def f():
    x=0
    while True:
        x += 1
        yield x

g = f()
print(g)
print(next(g))
```
结果：
```python
<generator object f at 0x10de82408>
1
```

-------

**生成器对象调用__next__的流程：**

把生成器传给 next(...) 函数时，生成器函数会向前，执**行函数定义体中的下一个 yield 语句**，返回产出的值，**并在函数定义体的当前位置暂停。**

**最终，函数的定义体返回时，外层的生成器对象会抛出 StopIteration 异常**——这一点与迭代器协议一致。

-------


>如果你不懂得 yield 关键字的基本用法，我建议你看[ CSDN 上冯爽朗的博文](https://blog.csdn.net/mieleizhi0522/article/details/82142856) 和 [廖雪峰的博文](https://www.liaoxuefeng.com/article/001373892916170b88313a39f294309970ad53fc6851243000)
>
> 这俩篇博文将 yield 的基本用法说的相当明白



**使用生成器表达式能够返回生成器对象**，如下例：


```python
g = (i for i in range(1,10))

print(g)
print(next(g))
```
结果:


```python
<generator object <genexpr> at 0x10ee48408>
1
```


-------

可以看出、这两种实现方式中，**生成器表达式跟像是语法糖**：可以代替简单的生成器函数。

>[语法糖](https://baike.baidu.com/item/%E8%AF%AD%E6%B3%95%E7%B3%96/5247005?fr=aladdin)：使用更少、更易读的代码，实现同样的功能。

但是、生成器函数更加灵活、可以**使用多个语句实现相对复杂的逻辑。**，同时生成器函数有名称，因此可以重用。

#### 使用生成器函数代替__iter__
    
可迭代对象中的__iter__用于获取迭代器，因为生成器实现了迭代器协议、所以我们可以使用生成器函数代替__iter__方法。

下面是用经典的迭代器模式实现的斐波那契数列类，Fibonacci 中只有__iter__方法用于返回迭代器。

```python
class Fibonacci:

    def __iter__(self):
        return FibonacciGenerator()


class FibonacciGenerator:

    def __init__(self):
        self.a = 0
        self.b = 1

    def __next__(self):
        result = self.a
        self.a, self.b = self.b, self.a + self.b
        return result

    def __iter__(self):
        return self
```

FibonacciGenerator 迭代器实现了[惰性计算](https://baike.baidu.com/item/%E6%83%B0%E6%80%A7%E8%AE%A1%E7%AE%97/3081216?fr=aladdin)、即不用事先计算好需要的元素放到列表里，而是每一次迭代都根据上一次迭代的结束状态实时的计算本次的产出值。

如果使用生成器函数，可以很优雅的将代码行数缩短，如下代码：


```python
class Fibonacci:

    def __iter__(self):
        a, b = 0, 1
        while True:
            yield a
            a, b = b, a + b
```

因为 Fibonacci 类本身**没有其他的行为**，你甚至可以使用**生成器函数代替这个类**


```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b
```

这就将 19 行的代码缩短到了短短 5 行。

这证明了使用生成器实现迭代器的功能是多么的优雅！

# 生成器和迭代器的区别
>python 不严格区分生成器和迭代器。
>如果要严格区分它们，你只能从设计模式的角度来区分他们了。

#### python 中生成器和迭代器不分彼此

**1. 从接口实现的角度来说，所有的生成器是迭代器**

生成器实现了特殊函数__next__和__iter__，即实现了迭代器协议、所以所有的生成器都是迭代器。

**2. 从实现方式来看，所有的生成器是迭代器，但迭代器可以不是生成器**

生成器那一小节中，我们说明了生成器的实现方式：
1. 调用生成器函数，其返回值就是生成器
2. 使用生成器表达式获取生成器对象

并说明了这两种方式返回的都是 types.GeneratorType 类型，而因为 **types.GeneratorType 实现了迭代器接口**，所以所有的生成器都是迭代器。

而我们可以**通过实现经典的迭代器模式，编写不是生成器类型的迭代器。**所以有的
迭代器不是生成器。

#### 从设计模式的角度分析

在《**设计模式：可复用面向对象软件的基础》**中，说明了**迭代器和生成器的定义：**
1. 迭代器**用于遍历集合**， 从中产出元素。
    *  迭代器**不能修改从数据源中读取的值**，只能原封不动地产出值。
2. 生成器可能**无需遍历集合就能生成值**。
    *  生成器不仅能产出集合中的元素，还可能会产出**派生自元素的其他值**。

事实上，在 python 中对于**获取遍历集合的迭代器**，已经相当简单了，诸如列表类型、字典类型、集合类型等对象，都可以通过调用内置的 iter（）函数来获取他们的迭代器。

例如，在可迭代对象一节中、Sentence 类的__iter__方法就是这样实现的。

**这些用iter（）函数获取的迭代器几乎都只能原封不动的产出值**，因为 python 设计者想要遵循迭代器模式。

-------

python 设计者想要遵循迭代器模式不仅仅体现在这个方面，从**实现生成器的角度上也能看出一点端倪。**

下面是生成器一节中的写的 Fibonacci 类，该类的__iter__方法返回自定义的迭代器类

```python
class Fibonacci:

    def __iter__(self):
        return FibonacciGenerator()


class FibonacciGenerator:

    def __init__(self):
        self.a = 0
        self.b = 1

    def __next__(self):
        result = self.a
        self.a, self.b = self.b, self.a + self.b
        return result

    def __iter__(self):
        return self
```

你可能已经注意到了，在这个经典的迭代器模式的代码中实现的**迭代器的职能却和生成器类似**，即无需遍历集合就能产出值。

这段代码一方面说明了，在 python 中迭代器和生成器几乎不做区分，因为这个迭代器其实做的事情和生成器类似。

另一方面也说明了，python 的设计者对于实现生成器方面**更倾向于你使用生成器函数**。

你也看到了，使用生成器函数可以将这 19 行代码替换成只有 5 行，所以在实现具有 “无需遍历集合就能生成值” 这一特性的对象时，**你应该使用生成器函数。**这是 python 设计者支持的。

-------

**总的来说，python 中的生成器和迭代器没法严格区分**、你可以实现迭代器使其做到生成器才能做到的事，而从句法的角度来说，所有的生成器都是迭代器。

但是、作为一名优秀的 python 程序员，我们应该理解 python 设计者的苦心，遵从他们的建议，即在迭代数据集合时，使用 iter（）获取迭代器、而在设计用于“无需遍历集合就能生成值” 或者 “产出派生自原数据集合” 类型的对象时，我们使用生成器函数获取生成器。













