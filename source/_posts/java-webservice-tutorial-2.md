layout: "post"
title: "Java Webservice实践（2）构建Webservice客户端"
date: "2016-10-20 21:39:04"
tags: [webservice]
categories: [webservice]
---

### 概述
总体而言，构建Webservice客户端的方式有很多种，概括起来分为两个环节：客户端stub生成和调用远程服务。对于生成stub而言，主要的方式是有两种：**通过wsdl静态生成然后导入客户端代码**和**通过Java反射机制动态生成（利用了JAXB API）**。

动态生成这里不介绍了，介绍一下静态生成的方法，具体可以细分为以下几种：
- jdk1.6+/bin中自带工具: `wsimport`
- cxf/bin中的工具: `wsdl2java`
- axis/bin中工具

调用的方法可以分为：
- 通过stub静态调用
- 通过JaxWsProxyFactoryBean半动态调用
- 通过CXF动态调用：
 + DynamicClientFactory方式（对于非JAX-WS标准）
 + JaxWsDynamicClientFactory方式（针对JAX-WS标准）
- 通过Axis动态调用：
 + Service.createCall()
— 通过HTTPURLConnection的方式调用（手动组装SOAP报文）
- 通过JAX Dispatch API的方式构建SOAP报文并调用

<!--more-->

### Stub的生成
#### 通过wsimport生成stub

wsimport调用的方法是：`wsimport -d [目录] -keep -Xnocompile -verbose [wsdl]`
其中`-d`后面表示生成文件存放的目录，`-keep`表示生成源文件，`-verbose`表示显示生成的详细过程，`-Xnocompile`表示不自动编译源码。
生成之后，导入到客户端的项目中。生成文件的结构类似下图：
![wsimport-java-structure](/img/webservice/wsimport.png)

接着，就可以在客户端调用了
```
WcgWebService serverStub = new WcgWebService();
com.up.client.WcgWsInterface client = serverStub.getWcgWsImplPort();
String resp = client.wsRequest("service", "requestMsg");
System.out.println(resp);
```

#### 通过wsdl2java生成

wsdl2java的调用方法与wsimport类似：`sh wsdl2java -d src -client http://localhost:8080/cxf-server/wcg/webservice/interface\?wsdl`
其中`-d`指定输出的目录，`-client`输出客户端代码，也可以指定`-server`输出服务端代码
生成之后，导入客户端的项目中。生成的文件结构类似下图：
![wsdl2java](/img/webservice/wsdl2java.png)

接着，在客户端调用如下：
```
WcgWebserviceInterface client = new Webservice().getWcgWebserviceImplPort();
String requestResponse = client.request("param", "param");
System.out.println(requestResponse);
```

#### 通过Axis客户端的工具生成
TODO


### 调用方式

####  通过CXF静态调用
将生成的stub引入到客户端的代码中进行调用。例如，对于通过`wsimport`生成的stub，我们可以用如下的方式调用：
```
UpWsAccess_Service serviceStub = new UpWsAccess_Service(new URL("http://0.0.0.0:8080/cxf-server/ws/request?wsdl"));
UpWsAccess action = serviceStub.getUpWsAccessPort();
Response response = action.request("demo", "<msg></msg>");
System.out.println("RETURN: [" + response.getStatus() + ", " + response.getResponseMsg() + "]");
```
其中`getUpWsAccessPort`这个方法也就是获取了webservice的某一个服务，wsdl中的每一个port代表的都是一个可供调用的服务。

这种方式的明显有点就是具有强类型支持，并且使用起来非常方便。并且，能够让人只管的理解webservice的本质：`通过客户端的stub来调用remote的服务`。
但是缺点就是比较“重量级”，需要一整套的流程。而且如果服务端的wsdl变了的话，那么客户端就需要重新生成一套新的stub了。这在变更的时候非常不方便。

**注意，导入的的stub直接使用会出现类似如下的错误：**
```
Caused by: com.sun.xml.bind.v2.runtime.IllegalAnnotationsException: 1 counts of IllegalAnnotationExceptions
Two classes have the same XML type name "{http://xxx.yyyy.com}createProcessResponse". Use @XmlType.name and @XmlType.namespace to assign different names to them.
    this problem is related to the following location:
        at xxx.yyy.gwfp.ws.dto.CreateProcessResponse
        at private xxx.yyy.gwfp.ws.dto.CreateProcessResponse xxx.yyy.gwfp.ws.jaxws_asm.CreateProcessResponse._return
        at xxx.yyy.gwfp.ws.jaxws_asm.CreateProcessResponse
    this problem is related to the following location:
        at xxx.yyy.gwfp.ws.jaxws_asm.CreateProcessResponse
```
原因是JAX-WS对webservice里面得每个方法都生成一个类，生成的类名为: methodName + "Response",所以就回导致生成的类和原来的类有两个相同的xml type。最直接的解决方法就是修改@WebService中的name属性，与类的名字不同即可。参考[这篇文章](http://www.cnblogs.com/happyPawpaw/p/4370967.html)。


#### 通过Axis静态调用

调用方式：
```
NativeUpayServiceLocator locator = new NativeUpayServiceLocator();
PayFeeInfo payData = new PayFeeInfo();
payData.setBusinessID("123456");
payData.setChargeDate("20161011121212");
payData.setRealPayFee(1231.0f);
payData.setSerialNo("123457");
PayFeeInfo[] pay = new PayFeeInfo[1];
pay[0] = payData;
PayResult result = locator.getNativeUpayPort().writePayFee("", "C0100001",pay );
```

这里需要注意的是：需要指定编码方式时，可以通过以下方式设置
```
Stub._setProperty(Call.CHARACTER_SET_ENCODING, "GBK");
```
参考自：[http://stackoverflow.com/questions/9093078/apache-axis-charset-wrong-encoding-on-tomcat](http://stackoverflow.com/questions/9093078/apache-axis-charset-wrong-encoding-on-tomcat)

#### 通过CXF动态调用

可以看出Stub调用的方式比较简单，只需要根据wsdl生成好对应的代理类就可以了，一般3-4行代码可以搞定。但是缺点就是接口不能随意改动，并且扩展起来不方便。
因此CXF提供了动态调用的API，利用Java反射的机制（ASM）动态解析WSDL文件并生成类，从而完成调用，这样的方式就很灵活。客户端完全不需要具体的类型信息，只需要制动调用什么方法，传递什么参数就可以了。

```
JaxWsDynamicClientFactory clientFactory = JaxWsDynamicClientFactory.newInstance();
Client client = clientFactory.createClient("http://localhost:8080/cxf-server/ws/request?wsdl");
Object[] result = client.invoke("request", "KEVIN", "<msg>adfsfsd</msg>");
```

`JaxWsDynamicClientFactory`这种方式的缺点也很明显：
- 首先，不能处理复杂的输入和返回(complex type)。输入类型可以进行一些指定，这点可以做到；但是对于返回类型，由于是根据wsdl动态生成的Response类，因此客户端无法通过编码的方式将Object强制转换为complex type。也就是像下面这样的方式是会包`ClassCastException`的，即使在引入了Stub生成的Response的情况下：
 ```
 // Will throw ClassCastException
 Response response = (Response) result[0]; 
 ```

- 其次，由于需要解析wsdl，并且生成相应的类，因此`JaxWsDynamicClientFactory`生成客户端的开销还是比较大的，实际运行中如果不采用一定的缓存策略，那么很难处理大量请求的场景。
- 最后，虽然这种方式是动态的，但是客户端在调用的时候依然需要明确调用哪个方法，这个方法需要什么参数。从而，服务端对于接口的改动依然会影响到客户端的代码，这一点是无法避免的。正如[StackOverflow中的回答](http://stackoverflow.com/questions/16871098/ws-dynamic-client-and-complextype-parameter)一样，无论选择静态还是动态的方式，客户端一定需要知道服务调用的具体接口细节。


**注意，在调用的时候如果使用了老版本的CXF(如2.2.12)可能会出现版本bug导致的错误：**
```
javacTask: 目标发行版 1.5 与默认的源发行版 1.7 冲突
Exception in thread "main" java.lang.IllegalStateException: 
Unable to create JAXBContext for generated packages: 
Provider com.sun.xml.bind.v2.ContextFactory could not be instantiated: 
javax.xml.bind.JAXBException: "com.bd.service" doesnt contain ObjectFactory.class or jaxb.index
```
解决方法可以参考[这篇文章](http://www.yyjjssnn.cn/articles/703.html)将CXF的版本进行升级，或用JDK1.6的版本。


#### 通过CXF半动态调用
利用JaxWsProxyFactoryBean不需要生成stub，但是需要知道接口。因此我们将这种方式称为半动态的调用。例如，下面的代码只需要引入`UpWsAccess`这个服务端对应的接口即可，而不需要一整套的stub，相比静态调用的方式还是方便了不少的。
```
JaxWsProxyFactoryBean factory = new JaxWsProxyFactoryBean();
factory.setServiceClass(UpWsAccess.class);
factory.setAddress("http://localhost:8080/cxf-server/ws/request");
UpWsAccess client = (UpWsAccess) factory.create();
Response response = client.request("nmdl/reqry.do", "<msg>sdfsdf</msg>");
```

#### 通过Axis动态调用
```
String namespace = "http://ws.wcg.unionpay.com";  // 定义命名空间
String serviceName = "wcgWebService"; // 定义服务名
QName service = new QName(namespace, serviceName);
String operationName = "WsRequest";

Call call = new Service().createCall();
call.setPortTypeName(service);
call.setOperationName(new QName(namespace, operationName));
call.setProperty(...)
call.addParameter("service", ParameterMode.IN);
call.addParameter("requestMsg", ParameterMode.IN);
call.setReturnType();
Object[] inParams = new Object[]{"demo", "demo"};
String ret = (String)call.invoke(inParams);
```
这种调用方式跟前文中的`JaxWsDynamicFactory`一样，只是传递参数的时候更直观了一些。缺点也是显而易见的，只能返回简单类型的response，一旦返回的是complex type，由于complex是借助于反射动态生成的，因此无法将Object强制转换为具体的类型。

#### 通过HTTPURLConnection调用
这种调用方式就需要手动去组装SOAP报文，是一种最为原始的方式。但其实这种方式体现除了Webservice的本质，就是SOAP + HTTP POST。只不过这种方式在便利程度和扩展性方面不是很友好。
具体的代码如下：
```
private static String buildSOAP() {
    String soap = "<soapenv:Envelope xmlns:soapenv=\"http://schemas.xmlsoap.org/soap/envelope/\" xmlns:ws=\"http://ws.wcg.unionpay.com\">\n" +
            "    <soapenv:Header/>\n" +
            "    <soapenv:Body>\n" +
            "    <ws:WsRequest>\n" +
            "    <service>apram</service>\n" +
            "    <requestMsg>param</requestMsg>\n" +
            "    </ws:WsRequest>\n" +
            "    </soapenv:Body>\n" +
            "    </soapenv:Envelope>";
    return soap;
}

public static void main(String[] args) throws Exception {
    String soap = buildSOAP();

    HttpURLConnection connection = (HttpURLConnection) (new URL("http://localhost:8080/cxf-server/ws/request?wsdl")).openConnection();
    connection.setDoInput(true);
    connection.setDoOutput(true);
    connection.setRequestMethod("POST");
    connection.setRequestProperty("Content-Length", String.valueOf(soap.length()));
    connection.setRequestProperty("Content-Type", "application/xop+xml; charset=utf-8; type=\"test/xml\"");
    connection.setRequestProperty("SOAPAction", "xxx");
    connection.setUseCaches(false);
    connection.connect();

    connection.getOutputStream().write(soap.getBytes("UTF-8"));
    connection.getOutputStream().flush();
    connection.getOutputStream().close();

    InputStream in = connection.getInputStream();
    String resp = IOUtils.toString(in);
    System.out.println(resp);
}
```

#### 通过SOAPConnection的方式调用
相比于之前的HttpURLConnection方式，SOAPConnection的方式稍微高级一些，不需要通过字符串的方式手动拼报文了，SOAPConnection的API会帮你拼，你只需要做一些定制化的工作就可以了，并且可以进行一些复杂的配置。

参考这篇文章：[http://blog.csdn.net/wanghuan203/article/details/9219565](http://blog.csdn.net/wanghuan203/article/details/9219565)


### Axis和Apache的对比
||Axis2|CXF|
|-----|-----|-----|
|目标|WebService引擎|简易的SOA框架，可以作为ESB|
|ws* 标准支持	|不支持WS-Policy	|WS-Addressing，WS-Policy， WS-RM， WS-Security，WS-I Basic Profile
|数据绑定支持	|XMLBeans、JiBX、JaxMe 、JaxBRI、ADB|	JAXB, Aegis, XMLBeans, SDO, JiBX
|spring集成	|不支持|	支持
|应用集成	|困难|	简单
|多语言	|支持C/C++	|不支持
|部署	|web应用|	嵌入式
|服务监控和管理|	支持|	不支持


### 参考资料
- [http://www.cnblogs.com/holbrook/archive/2012/12/12/2814821.html](http://www.cnblogs.com/holbrook/archive/2012/12/12/2814821.html)
- [http://blog.csdn.net/zolalad/article/details/31735791](http://blog.csdn.net/zolalad/article/details/31735791)
- [http://stackoverflow.com/questions/16871098/ws-dynamic-client-and-complextype-parameter](http://stackoverflow.com/questions/16871098/ws-dynamic-client-and-complextype-parameter)
