title: Python parameters
tags: [Python]
categories: 
- Python
- 语言机制
---

### 提纲
本文主要从以下几个方面分析python的参数机制：
1. 固定参数(位置参数)
2. 默认参数
3. 可变参数
4. 关键字参数

在每个部分中，我们区分**函数的定义**和**函数的调用**。
对于函数定义时的参数，我们称为parameter（形参）；对于函数调用时的参数，我们称之为argument(实参)。区分这两点对于解释清楚一些混淆的东西很重要

所以，以上的参数机制其实都是指的是函数定义时的形参，而不是调用时的实参！

<!--more-->

### 位置参数 positional parameter(固定参数)
1. 函数定义时：其中的x,n 都是固定**形参**
```python
def func(x, n):
    pass
```
2. 函数调用时：如果不指定名字，则传入的两个**实参**按照位置顺序依次赋给**形参**x和n
```python
func('abc', 10)
```

由于是位置参数，所以参数名在调用的时候是没有意义的，只有参数的顺序才有意义。因为形参会根据顺序依次匹配实参。

### 默认参数 default parameter
如果不传递n的话，上面的调用会报错：缺少位置**实参**
```python
func('abc')
# output: TypeError: func() missing 1 required positional argument: 'n'
```
这时候就可以靠默认参数来帮助我们省事，不用每次都给默认形参传递实参了

1. 函数定义时：
 ```python
def func(x, n=0):
    pass
```
 注意点：
 - 默认参数定义**必须**在所有的位置参数之后，并且默认参数的后面不能再有位置参数
 - 默认参数的必须指向不变的对象，否则会掉坑
 ```python
 def add_end(L=[]):
    L.append('END')
    return L

>>> add_end()
['END']
>>> add_end()
['END', 'END']

# Python函数在定义的时候，默认参数L的值就被计算出来了，即[]，因为默认参数L也是一个变量，
# 它指向对象[]，每次调用该函数，如果改变了L的内容，则下次调用时，默认参数的内容就变了，不再是函数定义时的[]了。
# 所以，定义默认参数要牢记一点：默认参数必须指向不变对象！

# 正确的做法：通过None这个不变的默认参数来做
def add_end(L=None):
    if L is None:
        L = []
    L.append('END')
    return L
 ```

2. 函数调用时：
 ```python
 # 可以不提供默认参数的实参，也可以提供
 func('abc')  
 func('abc', 10)
 # 对于多个默认参数的情况，如果按顺序提供，可以不指定默认形参的名字；
 # 否则需指定名字
 def func2(x, n=1, m=2):
    pass
 func2('abc', m=23, n=24)
 ```
 注意：不按顺序提供实参的情况仅仅适用于默认参数部分。也就是无论如何，必须先按顺序提供位置参数，之后提供的默认实参才有不按顺序一说。下面的调用是错误的：
 ```python
 func(n=10, 'acb')
 # output: SyntaxError: non-keyword arg after keyword arg
 ```


### 可变参数
在定义函数的时候，传入的位置参数个数不确定的时候使用

不用可变参数怎么做？转化为list或tuple：
```python
def calc(numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum
calc([1, 2, 3])
calc((1, 2, 3))
```
利用可变参数就可以直接传入，不需要显示转化了`calc(1, 2, 3)`

1. 函数定义
使用\*表达式即可：
```python
def calc(*numbers):
    # the same with above
```
2. 函数调用：可以传入任意个数的实参，包括0个
```python
calc(1, 2, 3)

# 用于对已有的列表进行操作
nums = [1, 2, 3]
calc(*nums)
```
注意：函数调用的时候，实参做了拷贝，原有的实参不变！

可变参数可以同时定义在位置参数后面：
```python
def func(x, *args):
    print(x)
    print(args)
func('abc')
# output: 'abc' ()
func('abc', 'efg') # （2）
# output: 'abc' ('efg')
func('abc', *'efg') # （3）
# output: 'abc', ('e', 'f', 'g')

def func2(x='123', *args):
    print(x)
    print(args)
func('abc')
# output: 'abc' ()
func('abc', 'efg') # （4）
# output: 'abc' ('efg')
func('abc', *'efg') # （5）
# output: 'c', ('f', 'g')   
```
 - 优先匹配位置参数, 无论有没有默认值。有默认值覆盖默认值，没有默认值赋值。剩下来的部分才给可变参数！
 - 注意(2)和(3)调用方式的不同
 - 注意(4)和(5)结果的不同

### 关键字参数（keyword parameter）
可变参数允许你传入0个或任意个参数，这些可变参数在函数调用时自动组装为一个tuple。而关键字参数允许你传入0个或任意个含参数名的参数，这些关键字参数在函数内部自动组装为一个dict。

1. 函数定义：
 ```python
 def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)
 ```

2. 函数调用：
 可以只传入位置参数
 ```python
 person('Michael', 30)
 ```
 也可以传入任意个数任意名字的关键字实参
 ```python
 person('Bob', 35, city='Beijing')
 person('Adam', 45, gender='M', job='Engineer')
 ```

 传入dict的方式：
 ```python
 extra = {'city': 'Beijing', 'job': 'Engineer'}
 person('Jack', 24, city=extra['city'], job=extra['job'])
 person('Jack', 24, **extra)
 ```

 注意：注意kw获得的dict是extra的一份拷贝，对kw的改动不会影响到函数外的extra


### 命名关键字参数

如果要限制关键字参数的名字，就可以用命名关键字参数，例如，只接收city和job作为关键字参数

1. 函数定义：
```python
def person(name, age, *, city, job):
    print(name, age, city, job)
```
 定义时可以有缺省值：由于指定名字，所以带缺省值的parameter不关心顺序
 ```python
def person(name, age, *, city='beijing', job):
    pass
```

2. 函数调用：
```python
person('Jack', 24, city='Beijing', job='Engineer')

# 必须传入参数名(除非使用定义了的缺省值，连值也不传)，否则会报TypeError
person('Jack', 24, 'Beijing', 'Engineer')
TypeError: person() takes 2 positional arguments but 4 were given
```

### 参数的组合
必须是如下顺序：
**必选参数、默认参数、可变参数/命名关键字参数和关键字参数**

```python
def f1(a, b, c=0, *args, **kw):
    print('a =', a, 'b =', b, 'c =', c, 'args =', args, 'kw =', kw)

def f2(a, b, c=0, *, d, **kw):
    print('a =', a, 'b =', b, 'c =', c, 'd =', d, 'kw =', kw)


>>> f1(1, 2)
a = 1 b = 2 c = 0 args = () kw = {}
>>> f1(1, 2, c=3)
a = 1 b = 2 c = 3 args = () kw = {}
>>> f1(1, 2, 3, 'a', 'b')
a = 1 b = 2 c = 3 args = ('a', 'b') kw = {}
>>> f1(1, 2, 3, 'a', 'b', x=99)
a = 1 b = 2 c = 3 args = ('a', 'b') kw = {'x': 99}
>>> f2(1, 2, d=99, ext=None)
a = 1 b = 2 c = 0 d = 99 kw = {'ext': None}
```

另外，对于任意函数，都可以通过类似func(*args, **kw)的形式调用它，无论它的参数是如何定义的

反之，接受任何参数的函数，可以定义为func(*args, **kw)这种形式。这种定义并不好，实际上，没有通用的规则在一个接受任何参数的函数内部做处理。但是对于decorator等一些应用，接受任何参数的设定就非常有用，因为decorator不关心包装的函数参数是什么，它确实可能需要一个这样的机制来传参。


### 注意点：

1. 关键字参数会覆盖位置参数的默认值
```python
def func(a='a', b='b', c='c', **kwargs):
    print('a:%s, b:%s, c:%s' % (a, b, c))
func(**{'a': 'z', 'b': 'd', 'c': 'r'})  # 关键字参数会覆盖前面的默认值
# output: a:z, b:d, c:r
```
2. 关键字参数与默认参数不同
 - 在定义的适合，默认参数本质上还是给了默认值的位置参数，必须定义在关键字参数的前面；而关键字参数应该最后定义，并且需要`**`表达式
 - 在调用时候，默认参数部分会优先匹配，匹配之后剩下来的才给关键字参数。所以在函数调用的时候谈论关键字参数实际上没有意义，它只是函数定义时的一个为了扩展用的占位符而已
  ```python
  def func(a='a', b='b', c='c', **kwargs):
      print(a, b, c)
      print(kwargs)

  func()
  # output: a b c {}
  func(**{'a': 'z', 'b': 'd', 'c': 'r'})
  # output: z d r {} # 注意，全部先匹配了默认参数
  func(**{'a': 'z', 'b': 'd', 'c': 'r', 'd':'z'})
  # output: z d r {'d': 'z'}

  # 可以用kwargs为没有提供默认值的位置参数提供值
  def func(a, b='b', c='c', **kwargs):
      print(a, b, c)
      print(kwargs)
  func(**{'a': 'z', 'b': 'd', 'c': 'r'})
  # output: z d r {}
  # 如果kwargs里面没有为a提供值，那么就会报TypeError了
  func(**{'b': 'd', 'c': 'r'})
  # output: TypeError: func() missing 1 required positional argument: 'a'
  ```


### Best Practices:
1. [PEP-3102](https://www.python.org/dev/peps/pep-3102/): 定义了keyworkd-only 参数（本质就是命名关键字参数），避免模糊不清地被位置参数匹配。（"keyword-only" arguments: arguments that can only be supplied by keyword and which will never be automatically filled in by a positional argument）

 有时候使用者希望函数可以接受可变参数，同时接受一些以keword形式传递的实参。如果不允许在可变参数后面定义named-keyword参数的话，唯一的解决办法是同时定义`*args`和`**kwargs`，然后手动抽取其中的一些entry。
 Why? 看实参传递的顺序，non-keyword实参 > keyword实参。所以如果定义在可变参数前，可能会被位置参数匹配掉(也就是不是真正意义上的keyword参数，而可能被解释器认为是带默认值的位置参数)。

 PEP3102允许在可变参数后面定义regular parameter,作为keyword-only arguments。永远不会被位置参数匹配，必须指定名字。

 定义方式：
 ```python
 # 同时有可变参数存在， 可变参数会suck up所有的non-keyword实参
 def sortwords(*wordlist, case_sensitive=False):
 # 没有可变参数存在, * means不允许任何可变参数存在
 def sortwords2(*, case_sensitive):
 ```
 显然，case_sensitive只能以keyword的方式赋值，不会被位置参数匹配。








### 参考
1. [廖雪峰的Python教程](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001431752945034eb82ac80a3e64b9bb4929b16eeed1eb9000)
2. [PEP3102](https://www.python.org/dev/peps/pep-3102/) 