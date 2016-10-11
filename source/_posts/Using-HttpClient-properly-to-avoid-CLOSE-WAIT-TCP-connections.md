title: (翻译) Using HttpClient properly to avoid CLOSE_WAIT TCP connections
date: 2016-10-11 20:46:41
tags: [httpclient, tcp]
categories: [java]
---

原文链接：[https://www.nuxeo.com/blog/using-httpclient-properly-avoid-closewait-tcp-connections/](https://www.nuxeo.com/blog/using-httpclient-properly-avoid-closewait-tcp-connections/)

在我帮助我的客户debug一个TCP connection关于CLOSE\_WAIT状态的问题时，我发现我们错误的使用了HttpClient。在这个问题上，如果你试图google [HttpClient CLOSE\_WAIT](http://www.google.com/search?q=HttpClient+CLOSE_WAIT)，你会发现很多人跟我们一样存在疑惑。但是关于这个问题，很多的解答不够直观，甚至[官方文档](http://hc.apache.org/httpclient-legacy/tutorial.html)都是错误的。所以我在这篇文章中进行了分析。


Apache HttpClient的基本用法如下：
```
HttpClient httpClient = new HttpClient();
HttpMethod method = new GetMethod(uri);
try {
    int statusCode = httpClient.executeMethod(method);
    byte[] responseBody = method.getResponseBody();
    // ...
    return stuff;
} finally {
    method.releaseConnection();
}
```
<!--more-->

但事实上，这是不够的。问题在于释放connection使得connection对于HttpClient而言重新可用，而非真正的关闭该connection，原因在于使用了Http1.1协议，HttpClient可以在同一个connection中批量发送后续的请求(?)。

尽管，server端可能单向关闭了连接，但是客户端的connection还是打开的，并且一直持续到下一次尝试从connection中读报文时（此时，客户端才会意识到服务端已经关闭了这个连接）。TCP就是采用这种方式工作的。上面的情况我们称之为`半关闭的连接`，**因为close()操作仅仅意味着我不会再往发送任何数据了，但是我还是可以从已经“closed”的连接中读取数据，只要另一端没有调用这个close()操作。**[译注: 理解这一点非常重要]

因此，当HttpClient实例超过作用域的时候，它会被GC标记为可回收状态，但是GC并不会立即回收它。在GC真正回收它之前，HttpClient内部的connection仍然处于打开的状态，此时的TCP状态就处于CLOSE\-WAIT。

为了解决这个问题，最简单的方法是在调用method之前设置如下代码：
```
method.setRequestHeader("Connection", "close");
```
这会导致HttpClient在接受完应答报文之后立即关闭connection。

另外一个方法是在finally块中添加如下代码：
```
httpClient.getHttpConnectionManager().closeIdleConnections(0);
```
效果应该跟上面的一样，都是让connection在idle之后立即销毁connection。

更好的方法是：不要每次都new HttpClient，而重用经过`MultiThreadedHttpConnectionManager`配置的一个client实例。当然，这种情况下，就需要最终记得清理MultiThreadedHttpConnectionManager。
```
private MultiThreadedHttpConnectionManager connectionManager;
private HttpClient httpClient;

public void init() {
    connectionManager = new MultiThreadedHttpConnectionManager()
    // ... configure connectionManager ...
    httpClient = new HttpClient(connectionManager);
}


public void shutdown() {
    connectionManager.shutdown();
}


public String process(String uri) {
    HttpMethod method = new GetMethod(uri);
    try {
        int statusCode = httpClient.executeMethod(method);
        byte[] responseBody = method.getResponseBody();
        // ...
        return stuff;
    } finally {
        method.releaseConnection();
    }
}
```



