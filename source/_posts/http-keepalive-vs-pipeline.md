title: Http KeepAlive的理解
date: 2016-11-29 21:11:10
tags: [http, network, keepalive]
categories: [network]
---

本篇结合资料，阐述了Http协议中的keep alive的理解。
<!--more-->

### keepalive为什么不能连续发送多个请求报文
知乎原贴：[https://www.zhihu.com/question/26515427](https://www.zhihu.com/question/26515427)

keepalive之前，是发出一个request,然后等待收取 response,在没有 conten-length的时候利用关闭链接的事件来判断 response接收完毕。因为那个时候网站就是一个HTML的文本，你仅仅需要请求一次就可以得到整个页面了。

后来,随着网页表现形式的多样化，一个页面的资源增多，构成一个页面的文件数不再是一个，为了快速加载网页(是快速，理论上你只开一个socket用完了关，再用再开也行)，同时并行地打开多个socket来获取资源(网络IO时间和页面渲染时间的比重越大，这个效果越明显)。

这个时候有一个问题就来了，我们为什么不在 一个socket上连续发送多个request。原因是因为HTTP1.1 及更低版本的协议，并没有一个字段用来区分一个response是归属于哪一个request的。但HTTP 2 就有这个字段了。因此在HTTP1.1 及更低版本，你只能在发送一个request之后，等待response的到来。

直到今日，应用最广泛的依然是HTTP1.1协议,这就造成了目前浏览器都是并行加载的,是的都是并行的。

在真正试图解决你的疑问的之前，我们来看一下,从发出request之前到接收respon之后，都发生了什么。
- 你向浏览器的地址栏输入一个域名.如 http://www.zhihu.com
- 浏览器向你的本地DNS服务器请求解析该域名,即将你的http://www.zhihu.com 解析为真实的IP地址.详细协议请查询RFC文档，其中对DNS协议的格式内容，指令意义，压缩算法，等都作出了规定。
- 拿到ip地址之后，发起TCP 握手(3次)，详情请看计算机网络TCP协议部分
- 握手成功，构造request,即 HTTP 中request请求.并发送到目的地。有关HTTP协议的内容请查阅RFC文档可以购买HTTP权威指南作为参考和释疑.
- 服务器接受到一个完整的request(该边界的指定一般是conten-length,chunked也有),根据用户的request内容运算出相应的response。
- 服务器将response 沿着request建立的连接，向浏览器(客户端)发送数据。
- keepalive的时候不关闭该连接，没有keepalive的时候发起tcp close,4次握手
- 浏览器根据接收到的response开始渲染页面。

至此，一个网页的打开过程完毕，我们从中提取出耗时的部分。
- DNS查询时间(一来一回,走UDP协议) 网络IO
- tcp 建立连接握手 网络IO
- request构造时间(cpu运算)
- request发送完毕时间(网络IO)
- 服务器接收request运算构造response(CPU运算,特指构造response过程中没有任何IO操作)
- 服务器发送response到客户端的时间(网络IO)
- 服务器关闭连接时间(IO)
- 客户端接收数据渲染页面时间(cpu运算)。

至此，一个流程就这样简单地构造完毕了。

好了，我们的目标是,尽可能地缩短完成一个页面加载的时间。
那么我们就需要不断地削减上述时间中的某一个。
到底什么才是最好的方案，将随着上述时间的比重不断地变化，没有什么是最优的。

浏览器默认的常用做法是，并行打开多个连接向服务器请求资源.（这里要注意）请求第一个页面的时候，只有一个连接(有的浏览器为了加速后面的多tcp问题，会有预连接,但是效果却因为资源不一定在一个ip上或者服务器不支持过多的连接而没有太大效果)，这里请求道的response之后解析出其关联的其他资源，才开始并行连接获取资源。

但是,tcp的握手和关闭是一个想当耗时,而且重复的过程(当传输速度变快的时候这个的比重相当大)，因此 keepalive的作用就是用来 复用已经打开的tcp连接.

这里有个数学问题，请求6个资源是，开3个连接复用三个呢,还是开6个连接,这就看情况了。

1.浏览器已经并行了，这个跟你想的不一样.
2.keepalive，是复用的意思，不是连续发出多个request.(这是HTTP1.1)，不算是串行。

个人看法：
HTTP1.1 request和response 的无标识符问题(即response无法指明是哪一个request的)，就造就了现在这种情况。HTTP 2 协议解除了这个问题。

HTTP 1.1，也没有一个request 多个 response的机制，没request，光有response的机制。关于这个特性HTTP 2有实现，但是争议依然比较大。
