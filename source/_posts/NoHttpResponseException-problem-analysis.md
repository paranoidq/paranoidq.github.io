title: NoHttpResponseException问题分析
date: 2016-10-17 10:11:30
tags: [httpclient, tcp]
categories: [java]
---

### 场景
在性能测试的时候，使用apache httpclient连续发送报文到server进行压测。压测的配置如下：
```
总数：10000
并发：400
每个发送线程的QPS限制：20
```
出现的问题是： 大概1000左右的报文会出现 `NoHttpResponseException [server ip] failed to respond`。调低并发度到300一下时，不会出现这样的异常。
<!--more-->

### 原因分析

查阅相关资料，官方对于`NoHttpResponseException`解释是：
>In some circumstances, usually when under heavy load, the web server may be able to receive requests but unable to process them. A lack of sufficient resources like worker threads is a good example. This may cause the server to drop the connection to the client without giving any response. HttpClient throws NoHttpResponseException when it encounters such a condition. In most cases it is safe to retry a method that failed with NoHttpResponseException.

大致意思跟我之前构想的一样，在某些情况下，通常在重负载下时，Web服务器可能能够接收请求，但无法处理它们。缺乏足够的资源，比如工作线程，这可能会导致服务器断开客户端连接，并且没有给予任何回应。当它遇到这样的条件HttpClient会抛出NoHttpResponseException。此异常是由于服务器端过载而拒绝接受请求（不再响应）所致。
但是细想，为什么服务端会drop掉连接呢？首先贴一则查阅到的stackoverflow上的分析：
>Most likely persistent connections that are kept alive by the connection manager become stale. That is, the target server shuts down the connection on its end without HttpClient being able to react to that event, while the connection is being idle, thus rendering the connection half-closed or 'stale'. Usually this is not a problem. HttpClient employs several techniques to verify connection validity upon its lease from the pool. Even if the stale connection check is disabled and a stale connection is used to transmit a request message the request execution usually fails in the write operation with SocketException and gets automatically retried. However under some circumstances the write operation can terminate without an exception and the subsequent read operation returns -1 (end of stream). In this case HttpClient has no other choice but to assume the request succeeded but the server failed to respond most likely due to an unexpected error on the server side.

大致含义：httpclient的连接一般由一个poolConnectionManager管理，而往往会有keep-alive的设置，即保持这个连接多久。但是当server端主动关闭连接的时候，由于**Connection可能处于idle状态，因此pool没有通知到HttpClient对应的connection，从而造成了connection处于半关闭的状态**。
因此在这种状态下，之后connection从pool中被唤醒或取出，然后发送报文在某些情况下write会在没有报异常的情况下结束，然后httpclient尝试从connection中read应答时返回了-1（注意，不是没有返回，否则应该是WaitTimeoutException）。httpclient认为server返回了应答，但实际上没有，因此抛出了NoHttpResponseException异常。


进一步思考：为什么server会主动关闭Connection呢？如果是处理能力有限，那么应该更加利用connection才对，而不应该主动drop。我的思考如下：
>1.假设server的连接限制是10，那么显然，如果现在只有5个请求来了，那么会创建5个connection。
2.如果现在来了20个请求，那么显然server是处理不过来的

### 解决方案

#### 1. 增加服务器处理能力
这种方法在我们的项目中有效，我们通过修改JBoss服务器的两个参数来达到增强服务器处理能力的效果：
- maxThreads 最大线程数
- acceptCount 最大等待线程数

Jboss的线程模型：（引用自[http://blog.csdn.net/lengyuhong/article/details/6319069
](http://blog.csdn.net/lengyuhong/article/details/6319069
)）
![jboss-thread-model](/img/jboss-thread-model.gif)


#### 2. retry
```
httpClientBuilder.setRetryHandler(new StandardHttpRequestRetryHandler(3, true));
```

#### 3. 限制keepAlive的时间
```
    ConnectionKeepAliveStrategy connectionKeepAliveStrategy = new ConnectionKeepAliveStrategy() {
            @Override
            public long getKeepAliveDuration(HttpResponse httpResponse, HttpContext httpContext) {
                return 20 * 1000; // tomcat默认keepAliveTimeout为20s
            }
        };
    PoolingHttpClientConnectionManager connManager = new PoolingHttpClientConnectionManager(20, TimeUnit.SECONDS);
    connManager.setMaxTotal(200);
    connManager.setDefaultMaxPerRoute(200);
    RequestConfig requestConfig = RequestConfig.custom()
        .setConnectTimeout(10 * 1000)
        .setSocketTimeout(10 * 1000)
        .setConnectionRequestTimeout(10 * 1000)
        .build();
    HttpClientBuilder httpClientBuilder = HttpClientBuilder.create();
    httpClientBuilder.setConnectionManager(connManager);
    httpClientBuilder.setDefaultRequestConfig(requestConfig);
    httpClientBuilder.setRetryHandler(new DefaultHttpRequestRetryHandler());
    httpClientBuilder.setKeepAliveStrategy(connectionKeepAliveStrategy);
    HttpClient httpClient = httpClientBuilder.build();
    ClientHttpRequestFactory requestFactory = new HttpComponentsClientHttpRequestFactory(httpClient);
    restTemplate = new RestTemplate(requestFactory);
```
主要是增加keepalive的策略，但这又带来一个问题，所有的连接只有20秒，无法使用长连接的性能优势

#### 4. closeIdleConnections
PoolingHttpClientConnectionManager.closeIdleConnections 方法 
```
@Override
public void closeIdleConnections(final long idleTimeout, final TimeUnit tunit) {
    if (this.log.isDebugEnabled()) {
        this.log.debug("Closing connections idle longer than " + idleTimeout + " " + tunit);
    }
    this.pool.closeIdle(idleTimeout, tunit);    
}
```

三种方案暂且列出，准备实践一下效果，后续再更新到博客中来。相比而言，retry的方案比较sb，但是估计很好用；keepAlive限制需要考虑场景，某些需要长连接的场景下可能不是很合适，并且把统一把连接过期关掉，显然会带来不小的性能损失。最后一种方案，应该是keepAlive的折中优化，本质上是如果connection一直被pool拿出来使用，那我就不关闭你；如果connection一直idle，那么过了一段时间我就主动关闭你了。这样就避免了keepAlive统一限制的问题，同时如果一个connection被pool拿出来用了，那么显然他不可能被server关闭；而idle connection被关闭了，所以不会出现上述server端drop connection而pool又无法通知idle connection的问题。

### 参考资料
1. [https://my.oschina.net/sannychan/blog/485677](https://my.oschina.net/sannychan/blog/485677)
2. [http://stackoverflow.com/questions/10558791/apache-httpclient-interim-error-nohttpresponseexception](http://stackoverflow.com/questions/10558791/apache-httpclient-interim-error-nohttpresponseexception)
3. [http://stackoverflow.com/questions/10570672/get-nohttpresponseexception-for-load-testing/10680629#10680629](http://stackoverflow.com/questions/10570672/get-nohttpresponseexception-for-load-testing/10680629#10680629)
4. [https://www.nuxeo.com/blog/using-httpclient-properly-avoid-closewait-tcp-connections/](https://www.nuxeo.com/blog/using-httpclient-properly-avoid-closewait-tcp-connections/)
5. [http://luan.iteye.com/blog/1820054](http://luan.iteye.com/blog/1820054)





