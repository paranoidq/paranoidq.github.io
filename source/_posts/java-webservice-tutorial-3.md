title: Java Webservice实践（3）在tomcat容器中发布webservice服务
date: 2016-10-20 21:39:04
tags: [webservice]
categories: [java]
---

### 概述
在[第一篇实践](http://paranoidq.github.io/2016/10/20/java-webservice-tutorial-2/)中，我们讲述了如何用standalone的方式部署一个webservice服务。但是很多情况下，我们需要将webservice部署在web服务器上，因此本文实践如何在tomcat服务器上部署webservice。

本文中，我们将讲述web容器的两种配置部署方式，一种是**基于Java Servlet的方式**，一种是**基于Spring框架的方式**。我们知道CXF本身设计就是与Spring无缝集成的，因此基于Spring框架的方式更方便通用；但是，有时候我们也希望通过简单的servlet就能够快速搞定一个webservice的发布。

 <!--more-->

### 基于spring framework
基于Spring的配置方式很简单，我们只需要关联对应的webservice bean，然后定义访问路径就可以了。

项目的结构类似下图：
![cxf-spring-intergeration](/img/webservice/spring-config-1.png)

#### Step 1
在`webservice-definition-beans.xml`中，我们给出jaxws的endpoint的定义，配置某一个webservice的实现类以及这个服务**对应于项目路径的相对URL**。需要注意的是，这里可以直接写具体的实现类，而不一定需要引用`service-definition-beans.xml`中定义的bean。当然，如果统一在spring的beans定义文件中定义了，那么在webservice的xml定义中只需要简单引用一下即可。
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:jaxws="http://cxf.apache.org/jaxws"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                       http://www.springframework.org/schema/beans/spring-beans.xsd
                       http://cxf.apache.org/jaxws http://cxf.apache.org/schemas/jaxws.xsd">
    <!--<import resource="classpath:service-definition-beans.xml"/>-->
    <import resource="classpath:META-INF/cxf/cxf.xml" />
    <import resource="classpath:META-INF/cxf/cxf-extension-soap.xml" />
    <import resource="classpath:META-INF/cxf/cxf-servlet.xml" />

    <jaxws:endpoint id="webservice" implementor="com.cup.wcg.access.webservice.WcgWsImpl" address="/request"/>

</beans>
```
#### Step 2
配置完webservice bean的xml文件之后，配置web.xml:
```
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>WEB-INF/webservice-definition-beans.xml</param-value>
</context-param>
<servlet>
    <servlet-name>CXFServlet</servlet-name>
    <servlet-class>org.apache.cxf.transport.servlet.CXFServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>CXFServlet</servlet-name>
    <url-pattern>/ws/*</url-pattern>
</servlet-mapping>
<listener>
    <listener-class>
        org.springframework.web.context.ContextLoaderListener
    </listener-class>
</listener>
```
其中CXFServlet指明了路径到webservice的映射关系。


### 基于Servlet
参考这篇文章[https://www.oschina.net/question/54100_26066](https://www.oschina.net/question/54100_26066)

#### Step 1
继承CXFNoSpringServlet，并override其中的loadBus方法
``` 
import javax.servlet.ServletConfig;
import javax.xml.ws.Endpoint;
 
import org.apache.cxf.transport.servlet.CXFNonSpringServlet;
  
public class WebServiceServlet extends CXFNonSpringServlet {
    private static final long serialVersionUID = -5314312869027558456L;
 
    @Override
    protected void loadBus(ServletConfig servletConfig) {
        super.loadBus(servletConfig);
        System.out.println("#####################");
        Endpoint.publish("/helloWorldService", new HelloWorldServiceImpl());
    }
}
```

#### Step 2
配置web.xml，指定webapp的URL以及对应的servlet
```
<servlet>
  <servlet-name>webservice</servlet-name>
  <servlet-class>com.crazycoder2010.webservice.cxf.server.servlet.WebServiceServlet</servlet-class>
</servlet>
<servlet-mapping>
  <servlet-name>webservice</servlet-name>
  <url-pattern>/webservice/*</url-pattern>
</servlet-mapping>
```
部署之后就可以通过`http://localhost:8080/CXF-Server/webservice/helloWorldService?wsdl`访问webservice服务了。

### 参考资料
- [https://www.oschina.net/question/54100_26066](https://www.oschina.net/question/54100_26066)
