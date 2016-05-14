title: Python实用Cases
date: 2016-05-14 20:22:44
tags: [Python, 碎片, TODO]
categories:
- Python
- Cookbook
---

### 如何列出目录下的所有文件

[https://taizilongxu.gitbooks.io/stackoverflow-about-python/content/39/README.html#](https://taizilongxu.gitbooks.io/stackoverflow-about-python/content/39/README.html#)
```python
from os import listdir
from os.path import isfile, join
onlyfiles = [ f for f in listdir(mypath) if isfile(join(mypath,f)) ]
```
<!--more-->

```
from os import walk

f = []
for (dirpath, dirnames, filenames) in walk(mypath):
    f.extend(filenames)
    break
```
