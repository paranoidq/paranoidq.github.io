title: java.util.ResourceBundle 源码分析
tags: [java, ResourceBundle, 源码, 未完成]
categories: [java]
---

### ResourceBundle简介
顾名思义，ResourceBundle主要就是管理Java程序的一些配置资源的工具类。但是这个管理与一般管理不同的地方在于:

- locale-independent, 即ResourceBundle封装了本地化的读取方法，并且根据Locale参数读对应的本地化配置，从而能够使程序自动在不同地区载入不同的配置文件(`name_CN.properties`, `name_US.properties`等)。JavaDoc说法:
```
 1. be easily localized, or translated, into different languages
 2. handle multiple locales at once
 3. be easily modified later to support even more locales
```

- 在没有指定Locale的情况下，自动载入默认配置`name.properties`
- 带有缓存功能
- 线程安全


### ResourceBundle使用
```java
public class ResourceBundleUtil {
    private ResourceBundleUtil() {}

    public static ResourceBundle newResourceBundle(String resourcePath) {
       return ResourceBundle.getBundle(resourcePath, Locale.ENGLISH);
    }

    public static ResourceBundle newResourceBundle(String resourcePath, Locale locale) {
        return ResourceBundle.getBundle(resourcePath, locale);
    }

    public static void main(String[] args) {
        ResourceBundle rb = ResourceBundleUtil.newResourceBundle("with_classpath");
        String value1 = rb.getString("key1");
        System.out.println(value1);

        // 注意，如果不在clsspath的root目录下，需要指定全名
        ResourceBundle rb2 = ResourceBundleUtil.newResourceBundle("i18n.within_folder");
        value1 = rb2.getString("key1");
        System.out.println(value1);
    }
}
```

自定义ResourceBundle例子
```java
public class MyResources extends ResourceBundle {
    public Object handleGetObject(String key) {
        if (key.equals("okKey")) return "Ok";
        if (key.equals("cancelKey")) return "Cancel";
        return null;
    }
    
    // keySet() is inherted from super class
    public Enumeration<String> getKeys() {
        return Collections.enumeration(keySet());
    }
    
    // Overrides handleKeySet() so that the getKeys() implementation
    // can rely on the keySet() value.
    protected Set<String> handleKeySet() {
        return new HashSet<String>(Arrays.asList("okKey", "cancelKey"));
    }
}
```

### ResourceBundle类结构
```plain
ResourceBundle
    |__ ListResourceBundle
    |__ PropertyResourceBundle
```

### ResourceBundle源码特性

#### 工厂方法
通过工厂方法`getBundle()`返回ResourceBundle的子类对象，处理不同的配置资源加载过程。

#### 实现类
ResourceBundle本身是abstract，实际使用的是两个实现类。
ListResourceBundle将配置资源看做key/value组成的列表，而PropertyResourceBundle使用properties来维护配置资源。

可以自己实现ResourceBundle，需要实现两个方法：`handleGetObject()` 和 `getKeys()`。另外需要注意的是，自己实现的ResourceBundle类要保证线程安全性，因为可能被多个线程同时使用。（ResourceBundle的非abstract方法和两个已知实现类的方法都是线程安全的）

#### 线程安全性
ResourceBundle的非abstract方法和两个已知实现类的方法都是线程安全的。
另外，自己实现的ResourceBundle类的`handleGetObject()`和`getKeys()`要保证线程安全性，因为可能被多个线程同时使用。

#### 资源加载过程的控制：ResourceBundle.Control
可以控制资源的搜索顺序、bundle的格式或缓存方式等。两种方式控制ResourceBundle加载配置资源的过程：
1. 在`getBundle()`的参数中指定Control实例
2. 通过指定`ResourceBundleControlProvider`的实现类。这个实现类会在ResouceBundle类被加载的时候就检测到，如果实现类针对某一个base name提供了Control对象，那么加载这个base name时的默认行为就会被改变。如果有多个providers针对同一个base name，那么选择第一个provider。

相关方法：

```java
ResourceBundle.getBundle(String, Locale, ClassLoader, Control);
```

#### Cache management
`getBundle()`返回的ResourceBundle会被默认缓存起来，从而下次请求同样的配置名时，会返回缓存过的ResourceBundle实例。
使用者可以选择不缓存、控制缓存时间（通过`time-to-live`变量），也可以清空cache。
相关方法：

```java
ResourceBundle.clearCache();
ResourceBundle.Control.getTimeToLive();
ResourceBundle.Control.needsReload();
```

### ResourceBundle源码分析


### 参考
java.util.ResourceBundle源码(JDK1.8)
[http://blog.csdn.net/haiyan0106/article/details/2257725](http://blog.csdn.net/haiyan0106/article/details/2257725)
