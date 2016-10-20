title: Java Webservice实践（2）构建Webservice客户端
date: 2016-10-20 21:39:04
tags: [webservice]
categories: [webservice]
---

### 概述
总体而言，构建Webservice的方式有很多种，概括起来可以分为以下几种：
1. 通过Stub的方式，用工具生成客户端的代理，具体的工具可以细分为很多：
 a. jdk1.6+/bin中自带工具: `wsimport`
 b. cxf/bin中的工具: `wsdl2java`
 c. axis/bin中工具
2. 通过CXF动态调用：
 a. DynamicClientFactory
 b. JaxWsDynamicClientFactory
3. 通过Axis动态调用：
 a. Service.createCall()
4. 通过HTTPURLConnection的方式调用（手动组装SOAP报文）
5. 通过SOAPConnection的方式调用

<!--more-->

### Stub的生成
#### 直接拷贝接口
拷贝接口，然后利用JaxWsServerFactoryBean方式进行调用
```
JaxWsProxyFactoryBean factory = new JaxWsProxyFactoryBean();
factory.setServiceClass(CXFDemo.class);
factory.setAddress(ADDRESS);
CXFDemo client = (CXFDemo)factory.create();
```

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

#### 通过Axis生成

调用方式：
...

这里需要注意的是：需要指定编码方式时，可以通过以下方式设置
```
Stub._setProperty(Call.CHARACTER_SET_ENCODING, "GBK");
```
参考自：[http://stackoverflow.com/questions/9093078/apache-axis-charset-wrong-encoding-on-tomcat](http://stackoverflow.com/questions/9093078/apache-axis-charset-wrong-encoding-on-tomcat)


### 通过CXF动态调用

通过以上的几种方法，可以看出Stub调用的方式比较简单，只需要根据wsdl生成好对应的代理类就可以了，一般3-4行代码可以搞定。但是缺点就是接口不能随意改动，并且扩展起来不方便。



### 通过Axis动态调用
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

### 通过HTTPURLConnection调用
这种调用方式就需要手动去组装SOAP报文，是一种最为原始的方式。但其实这种方式体现除了Webservice的本质，就是SOAP + HTTP POST。只不过这种方式在便利程度和扩展性方面都是很不友好的。
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
    connection.setRequestProperty("SOAPAction", "soap_action");
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

### 通过SOAPConnection的方式调用
相比于之前的HttpURLConnection方式，SOAPConnection的方式稍微高级一些，不需要通过字符串的方式手动拼报文了，SOAPConnection的API会帮你拼，你只需要做一些定制化的工作就可以了，并且可以进行一些复杂的配置。


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
