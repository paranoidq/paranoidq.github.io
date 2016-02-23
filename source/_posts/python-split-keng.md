title: python split的坑
date: 2016-02-23 09:34:04
tags: [python, 碎片]
categories: python
---


```python
line = "#9999#calillemilla#11: ElQuerido92, theola3lanclosb, Abigaille19, myherolancelot, dzastin03, DECIJEZ, lostsoul1987, canderse, ktornbjerg, L1nc0ln, BigDumbFate,"

sp1 = line.split(":")[0]
sp2 = sp1.split(sep="#")
print(sp1)
print(sp2)
```
<!--more-->

输出：
```
#9999#calillemilla#11
['', '9999', 'calillemilla', '11']
```

由于sp1的第一个字符就是`#`，所以在按照`#`分割的时候，会出现sp[0]为空的情况。因此要取实际的名字，应该用`sp[2]`，而不是`sp[1]`
