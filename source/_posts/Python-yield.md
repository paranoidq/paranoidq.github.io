title: Python yield
date: 2015-12-19 23:59:20
tags: [python, language]
categories: python
---

### 关于Python yield的解析

已有文章分析的很透彻了，我这里不再贴原文，传送门：
1. [http://pyzh.readthedocs.org/en/latest/the-python-yield-keyword-explained.html](http://pyzh.readthedocs.org/en/latest/the-python-yield-keyword-explained.html)
2. [http://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/](http://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/)


### 使用yield的好处
yield本质就是利用python内建的机制，生成一个迭代器。并且这个迭代器对应的函数的中间结果是python自动帮你维护的。简单来说，迭代器与yield的区别：
1. 迭代器需要构建类，实现相应的`next()`函数和`__iter__()`函数；而yield不需要
2. yield的中间结果由python机制保存，无需手动维护；而迭代器需要将函数功能封装为类，手动维护每次迭代的中间结果(也即手动维护`next()`函数与上一轮结果的关系)
3. 迭代器本质上也是解耦了函数的计算过程和结果获取，但是其实现需要较复杂的编程维护；yield借助python的内建机制，实现了函数计算过程和结果获取的解耦。

注意理解函数计算过程和结果获取的解耦。这里举个例子（来源于Python Cookbook1.3节）：
```
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

传送门2中提到的一个很好的例子：文件读取。如果直接对文件对象调用 read() 方法，会导致不可预测的内存占用。好的方法是利用固定长度的缓冲区来不断读取文件内容。通过 yield，我们不再需要编写读文件的迭代类，就可以轻松实现文件读取。
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

### yield的实现机制
将在另一篇博客中论述

