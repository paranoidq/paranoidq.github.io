title: Python generator
tags: [python, language]
categories: python
---


### python generator的设计考虑
在一般的程序设计中，function的调用是这样的：从第一行开始，一种执行到return语句结束。这种模式有一个特点:
1. 单一入口（entry point), 无论调用多少次，都从函数第一行开始
2. 无状态（Stateless），即函数的local变量每次都重新初始化，上一次被调用的状态丢失
3. return返回控制权给caller,除非再次调用，控制权不会回来。

而在有些情况下，我们希望的函数可以：yield一些值，也即函数暂时交出控制权给caller,你暂时不要销毁我的本地变量一些状态，因为将来还需要caller返回控制权，我可以继续执行下去。这就是yield以及生成器的设计初衷。

<!--more-->

generator需要负责两件事：
1. 向一个正常函数一行生成并返回值
2. 保存并恢复自己在交出控制权前一刻的内部‘状态’

正常的函数可以理解为subroutine,而generator可以理解为[coroutine](http://en.wikipedia.org/wiki/Coroutine)。但是python从语言层面并没有定义协程的概念。

### 什么是generator
1. generator就是一种迭代器（iterator)，满足迭代器的基本特性，即generator每次执行的时候只会生成一个value, 而不是全部生成
2. generator function是含有yield的function, 由它产生的迭代器叫做generator.
3. generator在返回值的时候使用yield，下次执行的时候从yield下一行开始执行，函数的上一次执行状态依然保留。
4. generator使用了与iterator相同的内建函数`next()`来获取下一个值，而next()又负责调用generator的`__next__()`。(这里可理解为python自动将带有yield的函数做了封装，使其成为了iterator) for循环隐式调用了next(generator)

### 什么是迭代器iterator,以及iterable类型
- 凡是可作用于for循环的对象都是Iterable类型；
- 凡是可作用于next()函数的对象都是Iterator类型，它们表示一个惰性计算的序列；

因此：
1. list、dict、str等是Iterable但不是Iterator（因为没有next()函数，并且序列创建时就已知），不过可以通过iter()函数获得一个Iterator对象
```python
from collections import Iterator
import itertools

isinstance((x for x in range(10)), Iterator)  # True
isinstance([], Iterator)  # False
isinstance({}, Iterator)  # False
isinstance(iter([]), Iterator)  # True

# 因为itertools.permutations生成的Permutations对象定义了`__next__()`函数和`__iter__()`函数，所以它具备Iterator的特性
l = [x for x in range(10)]
ll = itertools.permutations(l)
isinstance(ll, Iterator)   # True
```


### python generator的几点注意
1. generator和generator function不同，g function是定义，generator是generator function的一个实例，一个g function可以根据参数的不同生成不同的generator，即使参数相同，产生的也是不同的generator实例。
2. generator只能迭代一次，再次迭代需要重新生成实例；并且只能向前迭代。
3. generator中可以有return语句，遇到return,则generator直接抛出StopItreation异常，结束迭代过程


### 如何创建generator
1. 使用yield定义函数，然后用函数生成一个或多个generator实例
```python
def fibs(iterations):
    n, a, b = 0, 0, 1
    while n < iterations:
        yield b
        a, b = b, a+b
        n += 1
        
for i in fibs(10):
    print(i)
```

2. 使用生成器表达式
```python
g = (x*x for x in range(4))
print(g) # <generator object <genexpr> at 0x101b35288>
for x in g:
    print(x)  # 只能迭代一次
print(isinstance(g, types.GeneratorType))  # True

# 注意区分：生成器是不会直接算出所有结果的，而列表生成式则会直接计算所有结果，并且输出
l = [x*x for x in range(4)]
print(l) # [0, 1, 4, 9]

# 注意区分： 用itertools可以将一个列表迭代器化，但通常这样做没有意义。当然，itertools可以用于复制，合并迭代器等，这些功能就比较有用了
l = [x*x for x in range(4)]
ll = itertools.permutations(l)
print(ll)  # <itertools.permutations object at 0x1012e6a98>
print(list(ll))  # 用list可以将迭代器的结果全都算出来
print(isinstance(ll, Iterator))  # True
print(isinstance(ll, types.GeneratorType))  # False, 并不是Generator
```



### 使用yield的好处
yield本质就是利用python内建的机制，生成一个迭代器。并且这个迭代器对应的函数的中间结果是python自动帮你维护的。简单来说，迭代器与yield的区别：
1. 迭代器需要构建类，实现相应的`next()`函数和`__iter__()`函数；而yield不需要
2. yield的中间结果由python机制保存，无需手动维护；而迭代器需要将函数功能封装为类，手动维护每次迭代的中间结果(也即手动维护`next()`函数与上一轮结果的关系)
3. 迭代器本质上也是解耦了函数的计算过程和结果获取，但是其实现需要较复杂的编程维护；yield借助python的内建机制，实现了函数计算过程和结果获取的解耦。

注意理解函数计算过程和结果获取的解耦。这里举个例子（来源于Python Cookbook1.3节）：
```python
# 1.3 保留最后的N个元素
from collections import deque

def search(lines, pattern, history=5):
    previous_lines = deque(maxlen=history)
    for li in lines:
        if pattern in li:
            yield li, previous_lines
            # 使用yield可以将搜索过程和使用搜索结果的代码解耦,
            # 即不需要显示指定每一步的结果存储到容器中去
        previous_lines.append(li)

if __name__ == '__main__':
    with open(r'../../cookbook/somefile.txt') as f:
        for line, prevlines in search(f, 'python', 5):
            for pline in prevlines:
                print(pline, end='')
            print(line, end='')
            print('-' * 20)
```
如果不用yield，调用search的函数就需要知道serch是如何保存结果的，map还是list还是set。当然你可以用迭代器去做，也能达到解耦的目的，但是为了解耦，迭代器需要额外定义一个类，显然增添了很多工作，不如yield来得方便。

### 使用yield的场景和例子

#### 例子
参考文献2中提到的一个很好的例子：文件读取。如果直接对文件对象调用 read() 方法，会导致不可预测的内存占用。好的方法是利用固定长度的缓冲区来不断读取文件内容。通过 yield，我们不再需要编写读文件的迭代类，就可以轻松实现文件读取。
```
 def read_file(fpath): 
    BLOCK_SIZE = 1024 
    with open(fpath, 'rb') as f: 
        while True: 
            block = f.read(BLOCK_SIZE) 
            if block: 
                yield block 
            else: 
                return
```


### 允许向generator传递参数，并且随着迭代改变
[PEP342](http://www.python.org/dev/peps/pep-0342/) 规定可以向generator迭代过程中传递参数，（创建时传递参数很明显可以）

`other = yield number` 意味着，yield number, 并且当调用者send给我number，同时将other设置为number。

**原则上必须至少send一次None，至于迭代过程中是否需要send，视情况而定。有时候至少需要send一次正常值，才能保证参数不为None，从而执行不会报错。本例就是这样的情况。**

```python
def get_primes(number):
    while True:
        if is_prime(number):
            number = yield number
        number += 1

def print_successive_primes(iterations, base=10):
    prime_generator = get_primes(base)
    prime_generator.send(None)
    for power in range(iterations):
        print(prime_generator.send(base ** power))

```
这里的`send(None)`是必须的，否则会报错`TypeError: can't send non-None value to a just-started generator`。原文解释如下：

 When you're using send to "start" a generator (that is, execute the code from the first line of the generator function up to the first yield statement), you must send None. This makes sense, since by definition the generator hasn't gotten to the first yield statement yet, so if we sent a real value there would be nothing to "receive" it. Once the generator is started, we can send values as we do above.

需要在先send一个None,让generator执行到yield的地方(实际上get_primes(base)中的base被None覆盖了)，然后第一个send的值才能被yield接受。这里的第一个send(None)相当于启动了生成器。

`get_primes(base)`中的base被None覆盖了。经测试，将base替换为任意值都无关紧要，同时如果直接调用next(primer_generator)，则会报错：`Cannot use += for None and int(1)`，必须要send一个正常值才可以


### 更接近现实的案例
上述的例子基本在实际开发中很少写，这里给出一个更为接近实际的案例：

``` python
import random

def get_data():
    """Return 3 random integers between 0 and 9"""
    return random.sample(range(10), 3)

def consume():
    """Displays a running average across lists of integers sent to it"""
    running_sum = 0
    data_items_seen = 0

    while True:
        data = yield
        data_items_seen += len(data)
        running_sum += sum(data)
        print('The running average is {}'.format(running_sum / float(data_items_seen)))

def produce(consumer):
    """Produces a set of values and forwards them to the pre-defined consumer
    function"""
    while True:
        data = get_data()
        print('Produced {}'.format(data))
        consumer.send(data)
        yield

if __name__ == '__main__':
    consumer = consume()
    consumer.send(None)
    producer = produce(consumer)

    for _ in range(10):
        print('Producing...')
        next(producer)
```


### yield的基本原理
假设generator: g和调用g的函数caller,主要的流程：
- 当caller调用到next(g)的时候，执行generator function
- 执行到yield时，返回控制权给caller，同时记录g的当前状态（‘state’）
- caller获得控制权，并使用g返回的值
- 下一次调用next(g)的时候，g恢复上一次的state，并从yield的下一行开始继续执行
- 当next(g)调用时，如果g没有可以返回的值，抛出StopIteration异常，g结束



### 参考
1. [http://pyzh.readthedocs.org/en/latest/the-python-yield-keyword-explained.html](http://pyzh.readthedocs.org/en/latest/the-python-yield-keyword-explained.html)
2. [http://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/](http://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/)
3. [Improve your python-yield and generators explained](http://www.jeffknupp.com/blog/2013/04/07/improve-your-python-yield-and-generators-explained/)

