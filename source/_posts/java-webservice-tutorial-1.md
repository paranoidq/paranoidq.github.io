title: Java Webservice实践（1）构建简单的Webservice服务
date: 2016-10-20 20:45:04
tags: [webservice]
categories: [webservice]
---

### JAX-WS
JAX-WS(Java API for XML Web Services)，是SOAP协议的一个Java的实现规范，这个新规范是为了简化基于SOAP的Java开发。JAX-WS规范其实就是一组XMLweb services的JAVA API，JAX-WS允许开发者可以选择RPCoriented或者message-oriented来实现自己的web services。通过使用Java™ API for XMLWeb Services (JAX-WS) 技术设计和开发 Web服务，可以带来很多好处，能简化 Web 服务的开发和部署，并能加速 Web 服务的开发。

JAX-WS将本地的远程调用转换为XML协议（一般为SOAP格式），从而开发者不需要编写任何SOAP组装消息代码；同样，在服务端的JAX-WS会将SOAP消息解析为具体的函数调用，并返回结果。

![jax-runtime](/img/webservice/jax-ws-tutorials.gif)

<!--more-->
### 实现一个webservice的需要的大致流程

1. 服务端，定义服务接口（SEI: service endpoint interface）,并提供相关的实现类（SIB: service implementation bean）
2. 通过JAX-WS服务API接口发布为webservice服务，从而可以产生一个公开的?wsdl网页，能够访问到提供了哪些服务
3. 客户端通过JAX-WS的API根据公开的wsdl生成本地的代理（Stub）来实现对于远程方法的调用，调用过程就像在调用本地方法一样，JAX-WS Runtime会处理SOAP报文组装和网络传输等底层细节

 当然，JAX-WS 也提供了一组针对底层消息进行操作的API调用，你可以通过Dispatch 直接使用SOAP消息或XML消息发送请求或者使用Provider处理SOAP或XML消息。

### 一个简单的实现
#### 首先定义接口：
```
package com.up.ws.server;

import javax.jws.WebMethod;
import javax.jws.WebParam;
import javax.jws.WebResult;
import javax.jws.WebService;

/**
 * @author paranoidq
 * @since 0.1
 */
@WebService(targetNamespace = "http://ws.wcg.unionpay.com")
public interface WcgWsInterface {

    @WebMethod(operationName = "WsRequest")
    @WebResult(name = "WsResponse")
    String request(@WebParam(name = "service") String service, @WebParam(name = "requestMsg") String requestMsg);
}
```
注意点：
1. 这里定义了targetNamespace, 如果没有定义，则默认为该类的包名
2. 用@WebMethod注解来指定了operationName，如果没有指定，则操作名默认为方法名
3. 用@WebResult定义了返回值的名字，如果没有指定，默认为response
4. 用@WebParamd定义了参数名字，如果没有指定，默认为param1（或arg1？）

实际上，这些注解提高了wsdl的可读性。

#### 定义实现类
```
package com.up.ws.server;

import javax.jws.WebService;
import javax.jws.soap.SOAPBinding;

/**
 * @author paranoidq
 * @since 0.1
 */
@WebService(endpointInterface = "com.up.ws.server.WcgWsInterface", serviceName = "wcgWebService", targetNamespace = "http://ws.wcg.unionpay.com")
@SOAPBinding(style = SOAPBinding.Style.DOCUMENT)
public class WcgWsImpl implements WcgWsInterface {

    @Override
    public String request(String service, String requestMsg) {
        return "hello world";
    }
}
```
这里指定了实现类对应的接口，以及服务名。同时，指定了SOAPBinding的类型。
一般而言，SOAP消息的格式有两种：RPC和DOCUMENT。(**后面再理解区别**)


#### 发布接口
这里我们采用两种简单的方式发布接口
1. 第一种方式：
```
IHelloServices impl = new WcgWsImpl();  
// 创建WebServices服务接口  
WcgWsInterface factory = new JaxWsServerFactoryBean();  
// 注册webservices接口  
factory.setServiceClass(WcgWsInterface.class);  
// 发布接口  
factory.setAddress("http://localhost:8090/webservice");  
factory.setServiceBean(impl);  
// 创建服务  
factory.create();  
```

2. 第二种方式：
```
Endpoint.publish("http://localhost:8090/webservice", new WcgWsImpl());
```

发布之后，我们就可以通过`http://localhost:8090/webservice?wsdl`查看提供的服务了。
需要注意的是：CXF看上去没有在tomcat容器中运行，其实还是有容器的，而容器就是jetty。因此，需要我们在pom文件中包含相应的依赖。这里列出所有的依赖：(利用Endpoint的方式发布的时候，不需要`cxf-rt-frontend-jaxws`这个依赖)
```
<dependency>
    <groupId>commons-discovery</groupId>
    <artifactId>commons-discovery</artifactId>
    <version>20040218.194635</version>
</dependency>

<dependency>
    <groupId>org.apache.cxf</groupId>
    <artifactId>cxf-api</artifactId>
    <version>2.2.12</version>
</dependency>

<dependency>
    <groupId>org.apache.cxf</groupId>
    <artifactId>cxf-rt-transports-http</artifactId>
    <version>2.2.12</version>
</dependency>
<dependency>
     <groupId>org.apache.cxf</groupId>
     <artifactId>cxf-rt-frontend-jaxws</artifactId>
     <version>2.2.12</version>
 </dependency>

<dependency>
    <groupId>org.apache.cxf</groupId>
    <artifactId>cxf-rt-transports-http-jetty</artifactId>
    <version>2.2.12</version>
</dependency>
```

**注意**：`cxf-rt-transports-http-jetty`这个依赖一定要包含，否则会出现`CXF BusException No DestinationFactory for Namespace: http://cxf.apache.org/transport/http`的错误, 参考：
[http://stackoverflow.com/questions/24447280/cxf-busexception-no-destinationfactory-for-namespace-http-cxf-apache-org-trans](http://stackoverflow.com/questions/24447280/cxf-busexception-no-destinationfactory-for-namespace-http-cxf-apache-org-trans)

这样我们的一个webservice就构建好了。
下一篇将讲解如何构建客户端来访问我们的服务。
