title: Java动态代理的几种实现方式
date: 2016-01-18 15:48:27
tags: [java, 动态代理]
categories: [java]
---

### 静态代理的问题
![proxy pattern](http://images.techhive.com/images/idge/imported/article/jvw/2000/11/jw-1110-proxy-100157716-orig.gif)
1. 紧耦合：代理类必须实现被代理对象的接口
2. 硬编码：项目中大量充斥着类似\**proxy这样的类

如何解决问题？实际上也就是解决依赖的问题，代理类的创建不依赖于硬编码，想什么时候创建就什么时候创建，本质上也就是动态构建类和实例吧。JDK动态代理就是利用了Java的反射机制动态构建代理类和实例的。

<!--more-->

### JDK动态代理

```java
public interface Subject {
    void doSomething();
}

public class RealSubject implements Subject {
    public void doSomething() {
        System.out.println("This is real object.");
    }
}

/**
 * Jdk 动态代理必须代理接口,不能代理正常的类.
 *
 * 创建速度快于Cgi,但是运行速度大约比Cgi慢10倍.
 *
 * Created by paranoidq on 16/1/18.
 */
public class JdkProxy implements InvocationHandler {

    private Object proxied;

    public JdkProxy(Object proxied) {
        this.proxied = proxied;
    }

    /**
     *
     *
     * @param proxy The proxy parameter passed to the invoke() method is the dynamic proxy object implementing the interface.
     *              Most often you don't need this object.
     * @param method
     * @param args
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("proxy: " + proxy.getClass());
        System.out.println("method: " + method);
        System.out.println("args: " + args);
        if (method.getName().contains("do")) {
            System.out.println("method contains do*");
        }
        Method[] methods = proxy.getClass().getDeclaredMethods();
        for (Method m : methods) {
            System.out.println(m.getName());
        }
        /**
         * output:
         *      equals
         *      toString
         *      hashCode
         *      doSomething !!!
         */
        Class[] interfaces = proxy.getClass().getInterfaces();
        for (Class c : interfaces) {
            System.out.println(c);  // interface me.util.proxy.jdkproxy.Subject
        }
        /**
         * output:
         *      interface me.util.proxy.jdkproxy.Subject
         */
        return method.invoke(proxied, args); // 在实际对象上invoke方法,同时传入参数
    }


    public static void main(String[] args) {
        Subject subject = new RealSubject();
        Subject proxy = (Subject) Proxy.newProxyInstance(
                Subject.class.getClassLoader(),
                subject.getClass().getInterfaces(),
                new JdkProxy(subject));

        proxy.doSomething();
        System.out.println("=======");
        System.out.println(proxy); // toString的调用同样会dispatch到invoke,因此会被也"包装"
    }
}

```
JDK动态代理类的字节码是由Java在运行时通过反射动态生成的。

![dynamic proxy in java](http://www.techavalanche.com/wp-content/themes/Levels/images/upload/2011/08/Proxy_Diagram.jpg)

上面的例子基本已经显示了JDK代理的重要特性，下面整理说明一些重点：（主要参考Oracle JavaDoc）
 
1. 非interface的方法不会被代理，因为代理实例通过newProxyInstance创建的时候，只能给定Interface，子类的特性它一概不知。
2. 但是，java.lang.Object中的`hashCode`, `equals`, `toString` 方法会被代理到invoke，然后包装起来执行
3. invoke()的返回值会传递给代理实例，从而返回给客户端，因此客户端的代理实例声明的返回值类型要注意匹配。
4. 代理实例本身会被传递给invoke，作为第一个参数，即proxy。可以通过这个获取代理实例及其类型信息（代码中，我们获得了代理实例实际上有doSomething()这个方法，因为代理实例也继承了接口Subject！**所以说为什么要传入classloader啊，因为实际上是Java在用bytecode生成一个实现了Subject接口的动态代理类啊！这不就是隐式地在用反射构建一个类么？**） 
5. invoke代理的函数的参数列表以数组形式给出，对基本类型做了默认的boxing。另外，注意，在invoke内部可以任意修改这个参数数组，这里Java没有约束。（当然，一般来说修改函数的参数是很危险的，尤其还是这种经过代理的调用，会让调用方完全不知情！）

### CGLib动态代理

```java
/**
 * cgi代理可以代理任何类,采用的方式是创建类的子类,然后在子类中调用父类的方法,并织入aop的逻辑
 *
 * 创建慢,但运行性能快于jdk.
 * 适用于对象创建少,长期使用的情况,如singleton.
 *
 * Created by paranoidq on 16/1/18.
 */
public final class CgLibProxy implements MethodInterceptor {

    private Enhancer enhancer = new Enhancer();


    public Object getProxy(Class clazz) {
        enhancer.setSuperclass(clazz);  // 设置被代理类, CgLib根据字节码生成被代理类的子类
        enhancer.setCallback(this);
        return enhancer.create();
    }


    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {

        System.out.println("Before method");
        // invoke()会造成循环调用, 因为调用的还是子类对象的方法, 而子类对象的方法还是会被拦截.
        Object result = methodProxy.invokeSuper(o, objects);
        System.out.println("After method");
        return result;
    }

    public static void main(String[] args) {
        CgLibProxy proxyHandler = new CgLibProxy();
        // proxy normal class: RealSubject
        RealSubject proxy = (RealSubject) proxyHandler.getProxy(RealSubject.class);
        proxy.doSomething();
    }
}
```

1. 使用ASM（JAVA字节码处理框架）在内存中动态的生成被代理类的子类
2. 可以代理没有接口的类(JDK动态代理则不行)
3. 通过字节码技术为被代理的类创建子类，并在子类中采用方法`intercept`拦截所有父类方法的调用
4. 显然，基于第三点，CGlib不能代理final类
5. pom包: cglib + asm (底层依赖于asm)

### 参考

[Java Doc](https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/proxy.html)
[http://blog.csdn.net/janice0529/article/details/42884019](http://blog.csdn.net/janice0529/article/details/42884019)
[http://tutorials.jenkov.com/java-reflection/dynamic-proxies.html](http://tutorials.jenkov.com/java-reflection/dynamic-proxies.html)
[http://www.techavalanche.com/2011/08/24/understanding-java-dynamic-proxy/](http://www.techavalanche.com/2011/08/24/understanding-java-dynamic-proxy/)
