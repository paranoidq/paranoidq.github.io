title: Python Cookbook Useful Cases
tags: [python, cookbook]
categories: python
---

如题， Python Cookbook中有用的cases。

<!--more-->

3.11 随机选择(`random.choice()`, `random.sample()`, `random.shuffle()`)

4.2 代理迭代(`__iter__()`)
4.4 实现自定义对象的迭代协议(1. 使用`yield`或`yield from`， 2. 使用`__iter__()`-繁琐)
4.5 反向迭代(`__reversed__`或`reversed(known_size_list)`)
4.6 迭代器切片(`itertools.islice()`)
4.7 跳过可迭代对象的开始部分(`itertools.dropwhile()`)
4.10 跟踪迭代序列的索引值(`enumerate()`)。非常有用！
4.11 同时迭代多个序列(`zip()`, `zip_longest()`)
4.12 多个对象执行相同的操作(`itertools.chain()`)
4.13 创建数据处理管道(`yield`)
- 多个迭代器组合成一个工作流

4.14 展开嵌套的序列(`yield from`)
4.15 顺序迭代合并后的排序迭代对象(`heapq.merge()`)
- 可用于合并多个排序文件这样的操作

4.16 用迭代器代替while无限循环
- 简化重复调用的函数，如IO函数调用。不需要显式在while中判断结束标记。(`iter(lambda)`)

5.2 打印输出到文件
- 指定print(file=f)，且f必须以文本方式打开

5.5 文件不存在才能写入(`xt`或`xb`)
5.8 固定大小记录的文件迭代(一般二进制文件较多，利用`iter()`)
