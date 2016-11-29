title: HttpClient使用实践
date: 2016-11-29 20:51:09
tags: [httpclient, java]
categories: [java]
---

本篇主要总结在HttpClient使用过程中的最佳实践、注意点以及踩到的坑。
<!--more-->

### 包含连接池的HttpClient
```
PoolingHttpClientConnectionManager poolingHttpClientConnectionManager = new PoolingHttpClientConnectionManager();
poolingHttpClientConnectionManager.setDefaultMaxPerRoute(WcgProperties.WCG_HTTP_CLIENT_POOL_MAX_CONNECTION_PER_ROUTE);
poolingHttpClientConnectionManager.setMaxTotal(WcgProperties.WCG_HTTP_CLIENT_POOL_MAX_CONNECTION_PER_TARGET);

HttpClientBuilder clientBuilder = HttpClientBuilder.create();
clientBuilder.setConnectionManager(poolingHttpClientConnectionManager)
        .setConnectionTimeToLive(WcgProperties.WCG_HTTP_CLIENT_POOL_TTL, TimeUnit.SECONDS)
        .setConnectionManagerShared(true)
        .setKeepAliveStrategy(new DefaultConnectionKeepAliveStrategy() {
            @Override
            public long getKeepAliveDuration(HttpResponse response, HttpContext context) {
            return WcgProperties.WCG_HTTP_CLIENT_KEEP_ALIVE_TIMEOUT * 1000;
            }
        });
client = clientBuilder.build();

// 是否开启idle connection auto close
if (WcgProperties.ENABLE_IDLE_CONNECTION_AUTO_CLOSE == 1) {
    IdleConnManageUtil.closeIdleConnectionsPeriodically(poolingHttpClientConnectionManager);
}

/**********************************************/
public class IdleConnManageUtil {
    public static void closeIdleConnectionsPeriodically(PoolingHttpClientConnectionManager connectionManager) {
        ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(new ThreadFactory() {
            @Override
            public Thread newThread(Runnable runnable) {
                Thread thread = new Thread(runnable);
                thread.setDaemon(true);
                return thread;
            }
        });
        scheduledExecutorService.scheduleAtFixedRate(new IdleConnectionMonitorThread(connectionManager), 60, WcgProperties.WCG_HTTP_IDLE_CONNECTION_ALIVE_TIMEOUT, TimeUnit.SECONDS);
    }
}
```
注意点：
- 可以理解为`setDefaultMaxPerRoute`针对单个IP，`setMaxTotal`针对所有请求
- `setKeepAliveStrategy`可以自定义TCP连接KeepAlive的时间。但是需要注意的是，如果TCP连接在池中空闲的期间到期了，那么即使Server端关闭连接，Client端的这个TCP socket也感知不到。此时Server处于`CLOSE_WAIT`状态，知道TCP Socket从池中被唤醒或者超过CLOSE_WAIT时间或者被池自己关掉。当TCP被唤醒时，C端会感知到S端的关闭连接操作，因此会关闭这个TCP链路，重新建立一条链路发送下一次请求。（等于说每次request之前都会检查一下从池中取出的TCP链路是否有效，通过这个机制来保证不会出现C端拿着无效的TCP链路去请求）但是同样，以上的机制可能会导致S端有很多的`CLOSE_WAIT`状态，严重时可能造成Socket耗尽，这点需要注意。
- `closeIdleConnectionsPeriodically`是自定义函数，可以定时关闭连接池中长时间不用的连接

### 发送request的几种方法和注意点
#### 推荐的方式
```
response = this.client.execute(request, new ResponseHandler<Response>() {
    @Override
    public Response handleResponse(HttpResponse response) throws ClientProtocolException, IOException {
        return Response.create(
                response.getStatusLine().getStatusCode(),
                response.getStatusLine().getReasonPhrase(),
                HttpClient.this.getHttpResponse(
                        IOUtils.toByteArray(response.getEntity().getContent()),
                        paramsPack)
        );
    }
});
```
这种方式下，HttpClient将会替你**回收**连接。这里用**回收**而不是销毁，是因为HttpClient在默认的情况下，是会进行连接复用的。

#### 自行控制的方式
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

调用`method.releaseConnection();`释放了connection使得connection对于HttpClient而言重新可用，而非真正的关闭该connection，原因在于使用了Http1.1协议，HttpClient可以在同一个connection中批量发送后续的请求。


#### 自行控制的方式2
```
do {
    CloseableHttpResponse resp = httpClient.execute(httpGet);
    try {
    // Do what you have to do 
    // but make sure the response gets closed no matter what
    // even if do not care about its content
    } finally {
        resp.close();
    }        
} while (nextPage);
```

注意：必须要在finally块中调用`resp.close()`，否则这次请求会一直占用连接，不会被回收。而**默认的情况下，一个正常new出来的HttpClient实例只会给一个route留2个connection** [1]，因此一旦没有关闭，那么在多线程的情况下，上面的代码至多只能执行2次就会阻塞。

另一种释放连接的方法是：调用`EntityUtils.consume(resp);`，该函数内部会调用`resp.close()`从而释放连接。但是需要注意的是这种方式存在安全隐患，要明确resp不会过大或者是恶意的应答，从而挤爆缓冲区。最好能判断一下resp的大小，对于明显过大的应答要进行过滤。

### 取消request
```
HttpGet get = new HttpGet();
get.abor1. 
```


### 参考资料：
1. [http://stackoverflow.com/questions/22069821/apache-httpclient-4-3-3-execute-method-for-a-get-request-blocks-and-never-return](http://stackoverflow.com/questions/22069821/apache-httpclient-4-3-3-execute-method-for-a-get-request-blocks-and-never-return)

