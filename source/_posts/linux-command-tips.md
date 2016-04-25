title: Linux命令行Tips
date: 2016-01-28 17:08:12
tags: [Linux]
categories: 
- Linux
- 命令
---

### 原文链接
[Unix高手的十个习惯](http://www.ibm.com/developerworks/cn/aix/library/au-badunixhabits.html#six)
[Unix高手的另外十个习惯](https://www.ibm.com/developerworks/cn/aix/library/au-unixtips/)

### 在单个命令中创建目录树
```
$ mkdir -p tmp/a/b/c
$ mkdir -p project/{lib/ext,bin,src,doc/{html,info,pdf}, demo/stat/a}
```
<!--more-->
### 更改路径，而不要移动存档
直接解压到指定的目录，而不用移动存档然后再解压
```
$ tar xvf -C tmp/a/b/c newarc.tar.gz
```

### 将命令与控制操作符组合使用
#### 仅当另一个命令返回零退出状态时才运行某个命令
如果目录不存在，则不会解压
```
$ cd tmp/a/b/b && tar xvf ~/archive.tar
```
#### 仅当另一个命令返回非零退出状态时才运行某个命令
如果已经存在目录，则不会再新建了
```
$ cd tmp/a/b/c || mkdir -p tmp/a/b/c
```
#### 结合起来
```
$ cd tmp/a/b/c || mkdir -p tmp/a/b/c && tar xvf -C tmp/a/b/c ~/archive.tar
```


### 谨慎使用变量
最好将变量调用包括在双括号中
如果直接在字母数字文本后面使用变量名称，还要将变量名称包括在方括号中，使其与周围的文本区分。否则shell可能将尾随的文本解释为变量名的一部分，从而返回null（下面的例子-4）
```
$ ls tmp/       - 1
a b
$ VAR="tmp/*"   - 2
$ echo $VAR
tmp/a tmp/b
$ echo "$VAR"   - 3
tmp/*
$ echo $VARa    - 4

$ echo "${VAR}a"
tmp/*a
$ echo ${VAR}a
tmp/a
```

### 使用反斜杠管理长输入
```
$ cd tmp/a/b/c || \
mkdir -p tmp/a/b/c \
tar xvf -C tmp/a/b/c ~/archive.tar
```

### 在列表中对命令分组
#### 在 Subshell 中运行命令列表
用小括号
```
$ ( cd tmp/a/b/c/ || mkdir -p tmp/a/b/c && \
> VAR=$PWD; cd ~; tar xvf -C $VAR archive.tar ) \
> | mailx admin -S "Archive contents"
```

#### 在当前 Shell 中运行命令列表
用大括号，并且括号与实际命令之间包括空格，并且列表中的最后一个命令以分号结尾
```
$ { cp ${VAR}a . && chown -R guest.guest a && \
> tar cvf newarchive.tar a; } | mailx admin -S "New archive"
```


### 了解何时 grep 应该执行计数
避免通过管道将grep发送到wc -l来对输出进行计数。grep 的 -c 选项提供了对与特定模式匹配的行的计数，并且一般要比通过管道发送到 wc 更快
```
$ grep -c and tmp/a/longfile.txt
```

### 匹配输出中的某些字段，而不只是对行进行匹配
用awk
```
$ ls -l | awk '$6=="Dec"'
```

### 停止对cat使用管道
grep 的一个常见的基本用法错误是通过管道将 cat 的输出发送到 grep 以搜索单个文件的内容。这绝对是不必要的，纯粹是浪费时间，因为诸如 grep 这样的工具接受文件名作为参数。您根本不需要在这种情况下使用 cat。
```
$ grep and tmp/a/longfile.txt
```

时间测量
```
~ $ time cat tmp/a/longfile.txt | grep and
2811

real    0m0.015s
user    0m0.003s
sys     0m0.013s
~ $ time grep and tmp/a/longfile.txt
2811

real    0m0.010s
user    0m0.006s
sys     0m0.004s
~ $
```

### 使用文件名完成
在bash shell里面是Tab键。

查看shell：
```
$ echo $0
$ ps -p $$
```

### 使用历史扩展
使用!$获得前一个命令使用的文件名
```
$ grep pickles this-is-a-long-lunch-menu-file.txt
pastrami on rye with pickles and onions
$ vi !$
```

### 重用以前的参数
!:1 操作符返回某个命令使用的第一个文件名.
在第一个命令中，将一个文件重新命名为更有意义的名称，但为了保持原始文件名可用，创建了一个符号链接。重新命名文件 kxp12.c 以提高可读性，然后使用 link 命令来创建到原始文件名的符号链接，以防在其他位置使用该文件名。!$ 操作符返回 file_system_access.c 文件名，而 !:1 操作符返回 kxp12.c 文件名，该文件名是上个命令的第一个文件名。
```
$ mv kxp12.c file_system_access.c
$ ln –s !$ !:1
```

### 使用 pushd 和 popd 管理目录导航
```
$ pushd .
~ ~
$ pushd /etc
/etc ~ ~
$ pushd /var
/var /etc ~ ~
$ pushd /usr/local/bin
/usr/local/bin /var /etc ~ ~
$ dirs
/usr/local/bin /var /etc ~ ~
$ popd
/var /etc ~ ~
$ popd
/etc ~ ~
$ popd
~ ~
$ popd
```

### 查找大型文件
```
find / -size + 10000K -xdev -exec ls lh {}\;
```

### 不使用编辑器创建临时文件
创建临时文件
```
$ cat > my_tmp_file.txt
This is my tmp file
^D
$ cat my_tmp_file.txt
This is my tmp file
```

向已有文件附加内容
```
$ cat >> my_temp_file.txt
More text
^D
$ cat my_temp_file.txt
This is my temp file text
More text
```

### 使用 curl 命令行实用工具
```
$ curl –s http://www.srh.noaa.gov/data/ALY/RWRALY | grep BUFFALO
$ curl -o archive.tar http://www.somesite.com/archive.tar
```

### 有效利用正则表达式
[对话Unix-正则表达式](http://www.ibm.com/developerworks/cn/aix/library/au-speakingunix9/)

### 确定当前用户
```
$ whoami
paranoidq
```

在脚本中使用
```
if [ $(whoami) = "root" ]
then 
    echo "you cannot run this script as root"
    exit 1
fi
```

### 使用 awk 处理数据
```
$ cat text
testing the awk command
$ awk '{ i = length($0); print i }' text
23
$ awk '{ i = index($0,”ing”); print i}' text
5
$ awk 'BEGIN { i = 1 } { n = split($0,a," "); while (i <= n) {print a[i]; i++;} }' text
testing 
the
awk
command
```
统计功能
```
$cat sales
Gene,12,23,7
Dawn,10,25,15
Renee,15,13,18
David,8,21,17
$ awk -F, '{print $1,$2+$3+$4}' sales
Gene 42
Dawn 50
Renee 46
David 46
```

[AWK语言基础](http://www.ibm.com/developerworks/cn/views/aix/tutorials.jsp?cv_doc_id=172390)

