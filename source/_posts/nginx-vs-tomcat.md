title: nginx与tomcat有什么区别
tags: [nginx, tomcat, 服务器, 碎片]
categories: [nginx, tomcat]
---

### 解析

Apache/Nginx应该叫做`HTTP Server`；而Tomcat则是一个`Web App Server`，更准确的说是JSP/Servlet的Web App Server，因为其他语言开发的web应用无法在Tomcat上运行。

<!--more-->

Http Server的核心是Http协议层面的传输和访问控制，包括代理、负载均衡等。客户端通过Http协议访问Http Server上的文件资源(HTML文件、图片等)，然后Http Server如实将文件通过Http协议传输给客户端。当然，通过`CGI`技术也可以对Http Server传输的内容进行一些处理。
大多数时候，Nginx主要作为Tomcat前端的负载均衡器和代理，负责请求的转发和静态内容的直接返回。因为其高效的IO机制[2]，能够显著提高系统的吞吐率。

Web App Server，应用执行的容器。它首先需要支持开发语言的 Runtime（对于 Tomcat 来说，就是 Java），保证应用能够在应用服务器上正常运行。其次，需要支持应用相关的规范，例如类库、安全方面的特性。对于 Tomcat 来说，就是需要提供 JSP/Sevlet 运行需要的标准类库、Interface 等。为了方便，
应用服务器往往也会集成 HTTP Server 的功能，但是不如专业的 HTTP Server 那么强大，所以应用服务器往往是运行在 HTTP Server 的背后，执行应用，将动态的内容转化为静态的内容之后，通过 HTTP Server 分发到客户端。
Tomcat和Jetty，WebLogic同属一类。

<hr/>
>"tomcat用在java后台程序上，java后台程序难道不能用apache和nginx吗？"

—— 不能。apache和nginx不是servlet容器。什么是servlet容器呢？即实现HttpServletRequest、HttpServletResponse、HttpSession等等接口，解析http请求，通过类加载器加载对应的servlet实现类并调用，也就是说servlet容器必须由java或者基于jvm的语言实现。
**本质上，Servlet是J2EE规范的一部分，规定了容器和Web App必须遵循的接口规范。容器必须按照接口解析Java类，然后处理请求；同样Web App也只有按照规范来编写实现类，才能被容器所加载解析，从而完成特定的功能。**


### 参考
1. [https://www.zhihu.com/question/32212996](https://www.zhihu.com/question/32212996)
2. [http://90112526.blog.51cto.com/6013499/1059700](http://90112526.blog.51cto.com/6013499/1059700)
