title: Python super详解（译 + 进一步理解）
tags: [python, super]
categories: 
- python
---

## 翻译：Python's super() considered super!
### 基础

如果你没有惊讶于Python内置的super()，那么很可能你并没有真正知道它能做什么以及它如何有效的使用。本文章就主要在已有的python super()解释的基础上做出进一步的深入，主要包括：
- 提供了实际的使用cases
- 给出了理论模型，演示它如何工作
- 展示如何使super()发挥它的作用
- 使用super()的建议
- 真实的例子

本文的例子同时适用于python2和python3版本

首先，一个例子：子类继承内置的类，并且扩展了内置类的方法
```python
class LogoingDict(dict):
    def __setitem__(self, key, value):
        logging.info('Setting to %r' % (key, value))
        super().__setitem__(key, value)
```
上面的例子中，LoggingDict完成了dict的同样的工作——update元素，只不过扩展了功能，在update元素之前先打log了，然后通过super()将实际update的工作**代理**给了dict对象

如果没有super()，我们可以这样做：`dict.__setitem__(self, key, value)`，但是问题在于：这种硬编码的方式不利于程序的扩展性。利用super()实际上是一种`**间接引用**（computed indirect reference）。

间接引用的好处之一：**隔离**。不用在是函数内部制定代理类的具体名字。如果要修改base class为另一个类，那么`super()`会自动切换给代理类，而硬编码的方式则要修改具体实现。
```python
class LogoingDict(SomeOtherMapping):        # new base class
    def __setitem__(self, key, value):
        logging.info('Setting to %r' % (key, value))
        super().__setitem__(key, value)     # no change needed
```

间接引用的另一个好处：**动态**。可以在运行时自由指定间接引用指向的类。引用指向的具体计算方式依赖两点：
1. 调用super的class
2. 实例的基类的继承树

第一点往往与源码有关，在例子中，super()的调用者是`LoggingDict.__setitem__()`，这是固定的。
第二点则是关键的动态性所在（我们可以创建具有复杂继承关系的子类）。一个logging ordered dictionary，不改变我们已有的类：
```python
class LoggingOD(LoggingDict, collections.OrderedDict):
    pass
```
新类的继承树：`LoggingOD, LoggingDict, OrderedDict, dict, object`。注意：OrderedDict在dict的前面，因此，调用super()的`LoggingDict.__setitem__()`现在就把具体的upate任务代理给了OrderedDict，而不是上一个例子中的dict。

仔细考虑一下：在这个例子中，我们并没有改变LoggingDict的源码，而是新增了一个子类，这个新增子类的唯一逻辑就是组合了两个已有的类，并且控制他们的继承顺序。而super()则自动根据新类定义的继承顺序发挥了它的动态性能力！

### 基类的查找顺序
实际上，这里我称为检索顺序或继承树的说法，正式的叫法应该是：**方法解析顺序(Method Resolution Order, MRO)**。想要知道一个类的MRO可以用`__mro__`属性：
```python
>>> pprint(LoggingOD.__mro__)
(<class '__main__.LoggingOD'>,
 <class '__main__.LoggingDict'>,
 <class 'collections.OrderedDict'>,
 <class 'dict'>,
 <class 'object'>)
```

如果想按照我们想的MRO创建子类，那么首先需要了解MRO的计算机制：
>MRO的序列包括：本类，基类以及基类的基类们，直到object为止。一个类始终出现在它的基类前面，如果有多个同级基类，那么这些基类的顺序依照声明的顺序排列。

上述例子遵从MRO的规范：
- LoggingOD在它的基类LogginDict和OrderedDict前面
- LoggingDict在OrderedDict前面，因为`LoggingOD.__bases__`的声明顺序是(LoggingDict, OrderedDict)
- LogginDict在它的基类dict前面
- OrderedDict在它的基类dict前面
- dict在它的基类object前面

解析约束的过程被称作`线性化(linearizatoin)`。有很多论文研究这方面的内容，但是我们只需要知道两点即可：
- 基类永远出现在派生类后面
- 如果有多个基类，基类的相对顺序保持不变。




### 实践建议
super()的作用是将本类方法的调用代理给继承树中的某一个基类实例去完成。这里给出三个注意点：
- 保证通过super()调用的基类方法必须存在
- 调用者和被调用者需要有匹配的函数签名
- 调用super()的方法，每次出现都同样必须使用super()

1): 我们先看这一点：调用者的参数与被调用方法的参数一致。
这跟普通的方法调用不同，普通的方法调用在的时候被调用的方法是已知的，但是有了super()，在本类编码的时候被super()调用的方法是未知的。想象一下，我们可以之后定义一个subclass，从而在正在编写的class的MRO中引入新的类，改变MRO的顺序，从而可能改变super()实际调用的类！

我们的方法是：利用位置参数指定固定的签名。例如，在`__setitem__()`中，就保持了固定的两个位置参数：key和value。这种方法在LoggingDict也有体现，即`__setitem__()`与dict有同样的函数签名。

更灵活的方法：让继承树中的所有方法都接受这样的参数:`关键字参数 + 可变关键字参数`，并且每一层取走自己想要的参数，并通过`**kwargs`向上一层forward余下的参数，最终在调用链的最后一层使得可变关键字参数为空(`object.__init__()`不需要任何参数)。
```python
classs Shape:
    def __init__(self, shapename, **kwargs):
        self.shapename = shapename
        super().__init__(**kwargs)

class ColoredShape(Shape):
    def __init__(self, color, **kwargs):
        self.color = color
        super().__init(**kwargs)

cs = ColoredShape(color='red', shapename='circule')
```

2): 如何确定目标函数存在？
上面的例子就是最简单的case，即object有我们调用的方法，因此无论什么样的继承树都会有我们的目标方法，不会出现AttributeError。

那么对于object不存在的方法，我们的处理方法是：编写一个root类包含我们的目标方法，并且在object前面被调用。这个root类的职责就是'吞掉'方法的调用，不让super()继续向上层类传递（因为上层类没有我们的目标方法了，再传递就会最终出现AttributeError）。

Root的draw方法还可以利用防御性编程的策略，即用assert来确保调用链的上层没有draw方法了。这是为了避免子类可能错误的继承了一个没有声明Root为基类的类。
```python
class Root：
    def draw(self):
        # the delegatioin chain stops here
        assert not hasattr(super(), 'draw')

class Shape(Root):
    def __init__(self, shapename, **kwargs):
        self.shapename = shapename
        super().__init__(**kwargs)
    def draw(self):
        print('Drawing. Setting shape to: ', self.shapename)
        super().draw()

class ColoredShape(Shape):
    def __init__(self, color, **kwargs):
        self.color = color
        super().__init__(**kwargs)
    def draw(self):
        print('Drawing. Setting color to: ', self.color)
        super().draw()

cs = ColoredShape(color='blue', shapename='square')
cs.draw()
```

如果一个子类希望在MRO中引入其他类，那么这些其他类也必须继承自Root，从而确保draw方法不会到达object，而无法被Root.draw阻止下来。这个约定必须写在文档中，就像pyhon中所有的自定义异常都必须继承自BaseException一样。

3): 调用链的每层函数的调用都加上super()方法即可，这是约定。

### 如何引入'异类'（non-cooperative class）
有时候，我们也要想引入一些第三方的类，这些类并没有针对super设计或者没有遵循Root的约定。解决方法是：利用适配器包装一下。
例如下面的Moveable类并没有super()调用，并且它的`__init__()`方法函数签名与object不一致，并且它没有继承Root。
```python
class Moveable:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def draw(self):
        print('Drawing at position: ', self.x, self.y)
```

如果你希望把这个类引入ColoredShape的层次中，你需要做一个adapter：
```python
class MoveableAdapter(Root):
    def __init__(self, x, y, **kwargs):
        self.moveable = Moveable(x, y)
        super().__init__(**kwargs)
    def draw(self):
        self.moveable.draw()
        super().draw()

class MovableColoredShape(ColoredShape, MoveableAdapter):
    pass

MovableColoredShape(color='red', shapename='triangle',
                    x=10, y=20).draw()
```

### 原文
[Python's super() considered super!](https://rhettinger.wordpress.com/2011/05/26/super-considered-super/)

## 进一步理解
### super的本质
主要来自于[知乎-laike9m的回答](http://zhihu.com/question/20040039/answer/57883315)，少量删改。

不要一说到 super 就想到基类！super 指的是 MRO 中的下一个类！
super干的事情其实是这个：
```python
def super(cls, inst):
    mro = inst.__class__.mro()
    return mro[mro.index(cls) + 1]
```
两个参数分别作了两件事情:
1. inst负责生成MRO的list
2. 通过cls定位当前的MRO中的index,并返回mro[index+1]

一个例子：
```python
class Root(object):
    def __init__(self):
        print('this is root')

class B(Root):
    def __init__(self):
        print('enter B')
        # print(self)  # <__main__.D object at 0x...>
        super().__init__()  # python3中不用写成super(B, self).__init__()
        print('leave B')

class C(Root):
    def __init(self):
        print('enter C')
        super().__init__()
        print('leave c')

class D(B, C):
    pass

d = D()
print(D.__mro__)
print(B.__mro__)
print(C.__mro__)
```

输出：
```
enter b
enter c
this is root
leave c
leave b
(<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.Root'>, <class 'object'>)
(<class '__main__.C'>, <class '__main__.Root'>, <class 'object'>)
(<class '__main__.B'>, <class '__main__.Root'>, <class 'object'>)
```
因此，实际上super()调用的时MRO中的下一个类的对应方法，所以不难理解enter b之后是enter c而不是thi is root。因为C是B的下一个，至于为什么C是下一个，那就要看上文翻译中讲的MRO规范了。

需要注意的是，这里的MRO是self生成的，指的是self这个instance对应的类的MRO，self不同，MRO也不同。例如d的MRO就是第一行MRO，而如果instance是B，则MRO就是第二行了。而在上面的例子中，self一直是d。super().func是把实例的MRO中相对于当前类的下一个类的func执行，这个实例并非一定是当前类的，并且如果下一个类的func不再以super的方式调用，则调用终止（但是不建议，除非到了object或者Root）。

注意super继承只能用于新式类，用于经典类时就会报错。
- 新式类：必须有继承的类，如果没什么想继承的，那就继承objcet
- 经典类：没有基类，如果此时调用super就会出现错误：“super() argument 1 must be type, not classobj”

### super用在何处？
主要来自于[知乎-松鼠奥利奥的的回答](http://zhihu.com/question/20040039/answer/13772641)，少量删改。

super主要用于解决多继承的问题，直接用类名调用基类的方法在单继承的时候没问题，但是如果使用多继承，则会涉及到查找顺序（MRO）、重复调用（钻石继承）等问题。

如果没有复杂的继承结构，super作用不大。而复杂的继承结构本身就是不良设计。对于多重继承的用法，现在比较推崇 Mixin 的方式，也就是
- 普通类多重继承只能有一个普通父类和若干个 Mixin 类（保持主干单一）
- Mixin 类不能继承普通类（避免钻石继承）
- Mixin 类应该单一职责（参考 Java 的 interface 设计，Mixin 和此极其相似，只不过附带实现而已）
如果按照上述标准，只使用 Mixin形式的多继承，那么不会有钻石继承带来的重复方法调用，也不会有复杂的查找顺序 —— 此时 super 是可以有无的了，用不用全看个人喜好，只是记得千万别和类名调用的方式混用就好。


Python的多继承类是通过mro的方式来保证各个基类的函数被逐一调用，而且保证每个基类函数只调用一次（如果每个类都使用super）

### 源码级别的解释
[http://blog.csdn.net/johnsonguo/article/details/585193](http://blog.csdn.net/johnsonguo/article/details/585193)