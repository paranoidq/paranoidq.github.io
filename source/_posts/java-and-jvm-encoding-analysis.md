title: Java程序和Jvm编码机制的分析
date: 2016-10-25 21:55:12
tags: [encoding, 基础]
categories: [java]
---

本文起源于一次测试环境下构造报文发送时对于java编码的疑惑，经过查找资料和分析总结而成。对于JVM和Java代码中的编码问题和原理进行了较为深入的分析。如有错漏，欢迎指正。

<!--more-->

首先上一个原理图，借用自[知乎](https://www.zhihu.com/question/30977092)。个人认为非常恰当地描述了整个编码的原理，除了output部分不够详细。

![java-jvm-encoding](/img/java-jvm-encoding.jpg)

### Java程序内部编码的分析
该过程的分析参考了[这篇文章](http://blog.csdn.net/dslztx/article/details/47005107)。
<br/>
1. 在IDE中编写java代码然后保存，此时会根据`file.encoding`或`system.defaultEncoding`将文件编码为二进制的流，然后写入磁盘。
 - 这个过程用到了系统编码或文件编码，一般为GBK或者UTF-8等
<br/>
2. 然后IDE进行编译或javac进行编译：
 - 这个过程会从磁盘读取二进制流，然后根据`file.encoding`或`system.defaultEncoding`来解码出原始文件的内容
 - 可以指定解码的方式，如：
    - `javac -encoding utf-8 Main.java`
    - `maven compiler plugin中指定encoding`
 - 然后javac编译器会将读取的文件编译为Unicode的格式，因此此时源文件中的所有字符都是Unicode编码方式存储在内存中的*.class文件中
 - 然后将内存中的.class文件以二进制流的方式输入磁盘，使用系统默认编码即可（**实际上，这时候不存在编码的问题了，因为都是Unicode字符**）
 - 需要注意的是：在简体中文的Windows上，平台默认编码会是GBK，那么**javac就会默认假定输入的Java源码文件是以GBK编码的**。javac能够正确读取文件内容并将其中的字符串以Unicode输出到Class文件里，就跟自己写个程序以GBK读文件以UTF-8写文件一样。如果实际输入的确实是GBK编码（GBK兼容ASCII编码）的文件，那么一切都会正常。但如果实际输入的是别的编码的文件，例如超过了ASCII范围的UTF-8，那javac读进来的内容就会出问题，就“乱码”了
 - 对于任何的字符串，如String a= "我是字符串"而言，不论在java源码中以何种编码方式存在，编译为class文件之后，就全都是Unicode了。也就是，**我们可以认为，源文件的编码方式丢失了**。
<br/>
3. 编译过程完成之后，运行程序，此时JVM开始载入.class文件来运行，此时，在JVM内存中的各个字符仍然是Unicode编码的方式存在

4. 当产生output的行为时，就需要显式或隐式做编码转换的工作了，也即是**从Unicode转换为另一种具体的编码**。output的行为可以有多种：
 - 取得字节数组，即a.getBytes()或a.getBytes("GBK")。此时就从Unicode转换为了系统默认编码方式的二进制表示或指定编码方式的二进制表示
 - 显示在命令行界面上，如System.out.println(a)。此时隐式调用了转换和解码，将Unicode转为了系统默认编码的二进制表示，然后利用系统默认编码方式解码出具体的字符显示出来。
 - 输出为文件，同样系统默认编码方式或指定编码方式
 - 网络传输
 - 需要注意的是：**单纯以上过程只是在Unicode和具体某种编码之间转，不会出现乱码问题**。只有在不同的具体编码之间，才可能出现乱码问题。并且**编码本身没有问题，乱码只会出现在需要解码的时候**。
   > 一个典型的错误是：new String(a.getBytes("UTF-8"), "GBK");
    1) 其中a.getBytes()是按照UTF-8将a在内存中的Unicode表示转换出来，也就是这是一个编码的过程，使用了UTF-8编码方式
    2) 而new String()这个过程是将转换得到的byte[]数组用GBK解码，然后显示实际的字符
    3) 显然会发生乱码。这个问题的关键在于，没有理解字符串在JVM内存中的表示形式，以为是UTF-8的形式存储的。


### Java外部编码的分析
java程序中的字符串很多来自于外部，这个时候就需要注意乱码的问题了
- 首先，外部过来的字符串肯定是以某种编码方式编好的byte[]。数组，无论从文件读取、命令行输入还是从网络传输，本质上都是byte[]
- java程序需要将这些byte[]读取到JVM中进行处理，也就是**从具体的编码方式转换为Unicode的过程，因此提前知道数据源的编码方式才能正确转换**。这点很重要，如果不指定编码方式，那么默认就会采用系统编码，此时就有可能解码解的不对。
- 尽量要使用带有明确指定编码方式的method，而不要不知情的情况下使用了系统默认的编码，否则就可能造成解码错误，解出了乱码。


### 参考资料：
- [http://blog.csdn.net/jessenpan/article/details/15505515](http://blog.csdn.net/jessenpan/article/details/15505515
)
- [https://www.zhihu.com/question/30977092
](https://www.zhihu.com/question/30977092
)
- [http://bbs.csdn.net/topics/390806040](http://bbs.csdn.net/topics/390806040)
- [http://xm-koma.iteye.com/blog/2074191](http://xm-koma.iteye.com/blog/2074191)





