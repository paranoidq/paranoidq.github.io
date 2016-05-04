title: Java Exception处理总结
date: 2016-05-04 15:00:21
tags: [Java, Exception]
categories:
- Java
- 语言机制
---

### 基本理解

异常的本质：
- 编程错误导致的异常，如NullPointerException和IllegalArgumentException。
- 客户端错误导致的异常，如解析一个不正常的xml文档抛出异常。
- 资源错误导致的异常，如系统内存不足、网络连接失败、数据库连接失败。

Java两种异常类型：
- Unchecked Exception
- Checked Exception
均继承于Exception

<!--more-->

### 实践经验

#### 1.异常发生时客户端如何应对？
如果客户端可以采取行动恢复，则用checked exception。否则就应该使用unchecked exception。
尽量使用unchecked exception，不要强迫客户端API必须处理它们。
对于所有的编程错误，使用unchecked exception。

#### 2.保持封装性
不要将针对某些特性实现的checked exception用到更高层次去。例如不要让SQLException扩散到逻辑层，因为逻辑层不需要知道SQLException。大部分情况下，应该将其转化为unchecked exception抛出
```java
public void dataAccessCode() {
    try {
        //... some code that throws SQLException
    } catch(SQLException e) {
        throw new RuntimeException(e);
    }
}
```

大多数例如FileNotFoundException的异常客户端无法恢复，此时其实应该通过返回一些信息给最上层逻辑，然后由业务逻辑决定提示用户如何重新操作。(??)如果强行抛出checked exception，会导致客户端API必须处理无法处理的异常，同时错误的信息无法给到最上层的用户逻辑层。

**实际上，在项目中应该讲这个异常转化为自定义的runtime异常，记录SQLException的异常信息，然后在外层可以选择同一catch之后log。之所以是转化为runtime异常而不是checked exception，是因为大多数情况下，这种SQLException无法恢复，系统要做的工作实际上仅仅是log或者给用户“友好”的提示**

#### 3.自定义异常
如果自定义异常没有有用额外信息，那么直接用RuntimeException即可。

#### 4.异常文档化
@throws标签指明API可能抛出的__所有__异常，包括checked exception和unchecked exception。

#### 5.最外层catch RuntimeException
虽然runtime exception是程序员犯下的错误，但是为了程序在运行的时候不崩溃，所以仍然要在最外层catch所有的异常，并打log，便于之后的bugfix。([ulricqin.com/post/java-exception](ulricqin.com/post/java-exception))

#### 6.抛出和捕获
**尽早抛出，尽晚捕获**

尽早抛出能够具体显示异常发生的位置。
尽晚捕获：只有在代码能够处理时才捕获，否则应该继续抛出。过早捕获会导致上层能够处理的调用程序无法获得异常通知。

#### 7.利用try with resources打开和自动关闭资源
```java
try(MyResouce mr = new MyResource()) {
    // ...
} catch (Exception e) {
    throw new RuntimeException(e);
}
```
当运行至try-catch块之外后，资源会被运行环境自动关闭，无需finally。


#### 8.避免空catch块
要么抛出，要么就要在catch中做出具体的处理。

#### 9.不要用异常做控制流之用


### 如何构建优雅的Java异常处理框架
- 设计一个系统的异常基类AppException，继承与RuntimeException，包含自定义的异常处理逻辑。
- 系统中所有的异常都应该转译为AppException，当异常发生时，客户端接收AppException，并根据内部的原始信息进行处理。
- 根据不同的层次和逻辑，设计子类，如AppDAOExcepion等，根据需要将原始异常都转化为自定义的子类异常抛出，由于这些自定义异常是unchecked exception，因此不会破坏中间层的代码的API。
- 根据需要，在恰当的层次捕获这些AppException，然后做处理、提示和log。


具体而言，例如在三层架构中dao-service-controller:
- 自定义异常，包括基类和分类的异常。
- service将dao层和自身的异常进行转译封装，按照具体的分类，然后抛出为AppException。
- controller根据需要处理这些异常信息，可以集中捕获并进行log。

Spring的异常框架可以借鉴一下！！！

### 参考

1. [异常处理最佳实践](blog.jobbole.com/18291/)
2. [ulricqin.com/post/java-exception](ulricqin.com/post/java-exception)
3. [blog.csdn.net/hp91035/article/details/49305225](blog.csdn.net/hp91035/article/details/49305225)



