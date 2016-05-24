title: Java的类加载器
date: 2016-01-28 15:30:49
tags: [java, classLoader, jvm]
categories: 
- java
---

### 什么是类加载器
- 基本职责就是根据一个指定的类的名称，找到或者生成其对应的字节代码，然后从这些字节代码中定义出一个 Java 类，即 java.lang.Class类的一个实例
- 也可以加载图片等应用所需的资源

首先关注：java的内置类加载器的几种类型
- 启动类加载器（Bootstrap ClassLoader）：这个类加载器是Java虚拟机本身的一部分，这个类将负责存放<JAVA_HOME>\lib 目录下的类。要注意的一点是，这个类加载器是无法被用户直接引用的。并不继承自 java.lang.ClassLoader
- 扩展类加载器（Extention ClassLoader）：这个类加载器是由sun.misc.Launcher$ExtClassLoader实现的，负责加载<JAVA_HOME>\lib\ext 目录中class。开发者可以直接使用扩展类加载器
- 应用程序类加载器（Application ClassLoader）：这个类加载器是有sun.misc.Launcher$AppClassLoader实现的。这个类加载器是系统默认的类加载器。它根据 Java 应用的类路径（CLASSPATH）来加载 Java 类（在命令行中可以直接使用-cp或者-classpath命令进行指定）。一般来说，Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader()来获取它
<br/>

>**也就是说如果我们希望取代默认的类加载器，需要自己实现classloader，并且往往需要违反委托模型。违反委托模型的用意在于，不让父加载器拦截类，从而让类顺利的被我们的classLoader加载。**


### 委派模型

具体参见: [https://www.ibm.com/developerworks/cn/java/j-lo-classloader/](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)

![classloader-proxy-model](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/image001.jpg)

几点注意：

1. 真正完成类的加载工作的类加载器和启动这个加载过程的类加载器，有可能不是同一个。真正完成类的加载工作是通过调用 defineClass来实现的；而启动类的加载过程是通过调用 loadClass来实现的。前者称为一个类的定义加载器（defining loader），后者称为初始加载器（initiating loader）。**在 Java 虚拟机判断两个类是否相同的时候，使用的是类的定义加载器。也就是说，哪个类加载器启动类的加载过程并不重要，重要的是最终定义这个类的加载器**
 一个类的定义加载器是它引用的其他类的初始加载器，如Outer引用了类Inner，则由Outer的定义加载器负责启动Inner的加载过程。
2. loadClass()抛出的异常：java.lang.ClassNotFoundExeption
3. defineClass()抛出的异常：java.lang.ClassDefFoundError 
4. 自定义的类加载器都“继承“自AppClassLoader。当然在实现上是继承java.lang.ClassLoader
5. 类加载器在成功加载某个类之后，会把得到的 java.lang.Class类的实例缓存起来

### 实现一个自己的加载器
[https://www.ibm.com/developerworks/cn/java/j-lo-classloader/](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)
```java
package me.util.classes;

import java.io.*;

/**
 * @see https://www.ibm.com/developerworks/cn/java/j-lo-classloader/
 *
 * Created by paranoidq on 16/1/11.
 */
public class FileSystemClassLoader extends ClassLoader{

    private String rootDir;

    public FileSystemClassLoader(String rootDir) {
        this.rootDir = rootDir;
    }


    /**
     * 一般来说，自己开发的类加载器只需要覆写 findClass(String name)方法即可。
     * java.lang.ClassLoader类的方法 loadClass()封装了前面提到的代理模式的实现。
     *      该方法会首先调用 findLoadedClass()方法来检查该类是否已经被加载过；
     *      如果没有加载过的话，会调用父类加载器的 loadClass()方法来尝试加载该类；
     *      如果父类加载器无法加载该类的话，就调用 findClass()方法来查找该类。
     * 因此，为了保证类加载器都正确实现代理模式，在开发自己的类加载器时，最好不要覆写 loadClass()方法，而是覆写 findClass()方法。
     *
     * 类 FileSystemClassLoader的 findClass()方法首先根据类的全名在硬盘上查找类的字节代码文件（.class 文件），
     * 然后读取该文件内容，
     * 最后通过 defineClass()方法来把这些字节代码转换成 java.lang.Class类的实例。
     *
     * @param className
     * @return
     * @throws ClassNotFoundException
     */
    @Override
    public Class<?> findClass(String className) throws ClassNotFoundException {
        byte[] classByteArray = getClassByteArray(className);
        if (classByteArray == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(className, classByteArray, 0, classByteArray.length);
        }
    }

    private String convertClassNameToPath(String className) {
        return rootDir + File.separator + className.replace('.', File.separatorChar) + ".class";
    }

    private byte[] getClassByteArray(String className) {
        String path = convertClassNameToPath(className);
        try {
            InputStream is = new FileInputStream(path);
            ByteArrayOutputStream os = new ByteArrayOutputStream();
            int bufferSize = 4096;
            byte[] buffer = new byte[bufferSize];
            int bytesRead;
            while ((bytesRead = is.read(buffer)) != -1) {
                os.write(buffer, 0, bytesRead);
            }
            return os.toByteArray();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
       
        // another not preferred implementation

//        try {
//            InputStream is = getClass().getResourceAsStream(path);
//            if (is == null) {
//                return null;
//            }
//            /**
//             * Note that while some implementations of {@code InputStream} will return
//             * the total number of bytes in the stream, many will not.  It is
//             * never correct to use the return value of this method to allocate
//             * a buffer intended to hold all data in this stream.
//             *
//             * 这种方式有风险. 不能保证大小正确
//             */
//            byte[] buffer = new byte[is.available()];
//            is.read(buffer);
//            return buffer;
//        } catch (IOException e) {
//            throw new RuntimeException(e);
//        }
    }

}
```

注意点：
1. 覆写findClass方法，而不是loadClass方法
2. 读取is的时候，不要依赖不确定的available()方法返回不一定正确的buffer大小
3. 要传入类的全限定名称

**注意**：在测试的时候，如果试图加载的类本身处于classPath下的话，覆写findClass并不会改变双亲委派模型，因此仍然会被AppClassLoader拦截。要测试不同的classLoader加载的类不一样，必须让appClassLoader不能拦截（通过加载URL访问的某个类或不在classPath下面的类）


### 线程上下文类加载器

主要用于解决类加载委托机制不能工作的情况：

Java提供的很多服务提供者接口(Service Provider Interface, SPI)，允许第三方为这些接口提供实现。例如：JDBC、JCE、JNDI、JAXP等。
这里的问题在于，接口由rt.jar核心类库提供，而实现则是由第三方提供。
SPI接口代码经常需要加载具体的实现类，而核心的接口由BootstrapClassLoader加载，具体实现则是在classPath类路径下面，由appClassLoader加载。由上面的委托机制可知，由BootstrapClassLoader是无法引用AppClassLoader的（等于是反向委托），因此这种单向的委托无法完成需求。

而线程上下文类加载器则解决了这个问题，等于是一种”犯规“，如果不做任何的设置，Java 应用的线程的上下文类加载器默认就是系统上下文类加载器（AppClassLoader），从而利用这种”作弊“可以让BootstrapClassLoader反向委托AppClassLoader去加载实现类。

基本上涉及SPI的加载，都是通过这种方法来完成的。

### 类加载器与 OSGi
OSGI是java自定义类加载器机制的一个典型应用。

OSGi 中的每个模块（bundle）都包含 Java 包和类。模块可以声明它所依赖的需要导入（import）的其它模块的 Java 包和类（通过 Import-Package），也可以声明导出（export）自己的包和类，供其它模块使用（通过 Export-Package）。也就是说需要能够隐藏和共享一个模块中的某些 Java 包和类。这是通过 OSGi 特有的类加载器机制来实现的。
OSGi 中的每个模块都有对应的一个类加载器。它负责加载模块自己包含的 Java 包和类。当它需要加载 Java 核心库的类时（以 java开头的包和类），它会代理给父类加载器（通常是启动类加载器）来完成。当它需要加载所导入的 Java 类时，它会代理给导出此 Java 类的模块来完成加载。模块也可以显式的声明某些 Java 包和类，必须由父类加载器来加载。只需要设置系统属性 org.osgi.framework.bootdelegation的值即可。


### 参考
[https://www.ibm.com/developerworks/cn/java/j-lo-classloader/](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)
[深入探索Java热部署](http://blog.sae.sina.com.cn/archives/841)
