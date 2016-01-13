title: SoftReference及其缓存应用
tags: [java, softreference, 缓存]
categories: [java]
---

### 参考原文
[http://www.javaspecialists.eu/archive/Issue015.html](http://www.javaspecialists.eu/archive/Issue015.html)

### SoftReference

#### Strong Reference
首先解释强引用，是Java默认的引用形式，即一个对象被一个变量直接引用的情况，如：
```java
Object o = new Object();
```
这种情况下，对象是不会被GC清除的，除非变量的引用解除（例如超出作用域之后，变量被清空）。

#### WeakReference
WeakReference是对普通引用的封装，表明一个对象被一个变量弱引用。与强引用不同的是，弱引用的对象如果没有其他的强引用指向它，那么GC依然会回收它。
```java
WeakReference<Object> wr = new WeakReference<Object>()
```

#### WeakHashMap
设计一个缓存时，需要将缓存的对象保存在一个map中，这里如果普通的HashMap存在问题，可能导致内存OOM。原因在于HashMap保存的是强引用，即使缓存对象在客户端的引用失效了，由于HashMap强引用的存在，这个缓存对象依然不会被GC回收。这就导致如果长时间读写HashMap缓存，会产生OOM。
Sun提供的解决方案是：WeakHashMap。WeakHashMap的key是以WeakReference的形式存在的，一旦这个key对象在程序中没有了其他强引用，那么GC就会在后续考虑回收整个entry，从而保证内存不会一直被这些已经没有用的缓存对象填满。

WeakHashMap的问题在于，实际上作为一个缓存应用，还是希望缓存的对象尽可能多待会儿，也即并不希望一旦没有人使用缓存对象，就立马让GC回收，万一之后马上有人要用了呢?!这里我们就引入了SoftReference来解决这个问题。

#### SoftReference
比WeakReference更"weak"，即即使没有外部的强引用指向缓存对象，GC依然不回收。只有等到JVM的内存快满了的时候，才回收这些SoftReference对象。这不正是我们希望达到的缓存特性么？(值得一提的是，SUN没有提供这种比WeakHashMap更合理的缓存实现形式，不知为何)


### 摆脱GC控制，用SoftReference实现一个更适合缓存的HashMap

实现的几点优化：
- 每次改变Map(put, remove, clear)或获取map size的时候都去遍历查看哪些SoftReference对象被GC回收了。如何检查？很简单，通过一个ReferenceQueue来实现。
- 自己设计一个LinkedList，保存最近被访问的缓存对象的强引用，避免被GC回收这些最近使用的对象。
- 使用装饰模式来包装原来的HashMap方法

``` java
package me.util.cache;

import java.lang.ref.ReferenceQueue;
import java.lang.ref.SoftReference;
import java.util.*;

/**
 * Created by paranoidq on 16/1/13.
 */
public class SoftHashMap<K, V> extends AbstractMap<K, V>{

    private final Map<K, SoftValue<V>> hash = new HashMap<>();

    // manually keep the hard reference to recently used objects
    private final int HARD_SIZE;
    private final LinkedList<V> hardCache = new LinkedList<>();

    // track garbage collected objects
    private final ReferenceQueue<V> queue = new ReferenceQueue<>();

    public SoftHashMap() {
        this(100);
    }

    public SoftHashMap(int hardSize) {
        this.HARD_SIZE = hardSize;
    }

    @Override
    public V get(Object key) {
        V result = null;
        SoftReference<V> softRef = hash.get(key);
        if (softRef != null) {
            result = softRef.get();
            if (result == null) {
                hash.remove(key);
            } else {
                hardCache.addFirst(result);
            }
            if (hardCache.size() > HARD_SIZE) {
                hardCache.removeLast();
            }
        }
        return result;
    }


    @Override
    public V put(K key, V value) {
        processQueue();
        SoftValue<V> sv = hash.put(key, new SoftValue<>(key, value, queue));
        if (sv == null) {
            return null;
        } else {
            return sv.get();
        }
    }

    @Override
    public V remove(Object key) {
        processQueue(); // throw out garbage collected values first
        SoftValue<V> sv = hash.remove(key);
        if (sv == null) {
            return null;
        } else {
            return sv.get();
        }

    }

    @Override
    public void clear() {
        hardCache.clear();
        processQueue(); // throw out garbage collected values
        hash.clear();
    }

    @Override
    public int size() {
        processQueue(); // throw out garbage collected values first
        return hash.size();
    }

    private void processQueue() {
        SoftValue<V> sv;
        while ((sv = (SoftValue<V>)queue.poll()) != null) {
            System.out.println(sv.key + "=" + sv.get() + ", and has been collected by GC.");
            hash.remove(sv.key);
        }
    }

    private static class SoftValue<V> extends SoftReference<V> {
        private final Object key;

        public SoftValue(Object key, V value, ReferenceQueue<V> q) {
            super(value, q);
            this.key = key;
        }
    }

    @Override
    public Set<Entry<K, V>> entrySet() {
        throw new UnsupportedOperationException();
    }



    /*
    test
     */

    private static final int _1MB = 1024*1024;
    private static void print(Map<String, Integer> map) {
        System.out.println("One=" + map.get("One"));
        System.out.println("Two=" + map.get("Two"));
        System.out.println("Three=" + map.get("Three"));
        System.out.println("Four=" + map.get("Four"));
        System.out.println("Five=" + map.get("Five"));
    }
    private static void testMap(Map<String, Integer> map) {
        System.out.println("Testing " + map.getClass());
        map.put("One", new Integer(1));
        map.put("Two", new Integer(2));
        map.put("Three", new Integer(3));
        map.put("Four", new Integer(4));
        map.put("Five", new Integer(5));
        print(map);
        // 注意，这里直接写10MB的话，无法满足测试条件
        //byte[] block = new byte[11*_1MB]; // 10 MB
        byte[][] bs = new byte[10][];
        for (int i = 0; i < 10; i++) {
            bs[i] = new byte[_1MB];
        }
        
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        print(map);
    }

    public static void main(String[] args) {

        //testMap(new HashMap());
        testMap(new SoftHashMap(2));
    }
}

```

关于测试程序的几点说明:

1. `byte[] block = new byte[11*_1MB]`希望占用JVM空间，然后触发GC回收SoftReference的缓存对象，但是并不成功。原因在于:
 大对象直接在老年代分配，而不会占用Eden空间，也就是Map的存储空间。从这张图可以看出，确实是占用了Old空间，而不是Eden。图中7、8行分别是OC(老年代总空间)和OU(老年代使用空间)。

 ![header](/img/2016-1-13-0.png)![old](/img/2016-1-13-1.png)

 并且由于有强引用的存在，因此虽然Old快满了，GC却无法回收。导致实际测试中，没有任何GC，然后直接爆OOM，虽然Eden和Survivor还有很多空间。（可见无法回收的大对象非常消耗JVM）

2. 正确的方法是，逐步添加1MB的对象。这样保证对象确实会占用年轻代的空间(Eden和Survivor)。如图，产生了minorGC和FullGC，证明Map中的对象被GC清除了。并且，fullGC之后，bytes对象被移到了老年代中，占用了8906KB的空间。还有大约1MB在Eden区中。
![header](/img/2016-1-13-0.png)![eden](/img/2016-1-13-2.png)
![eden](/img/2016-1-13-3.png)

3. 我们运行程序的结果也表明，确实SoftHashMap的前面几个对象在JVM内存不足的时候被GC回收了。运行程序的JVM参数：`-Xms15m -Xmx15m `。也可以加上`--verbose:gc`在console中打印GC情况。
```plain
Testing class me.util.cache.SoftHashMap
One=1
Two=2
Three=3
Four=4  
Five=5

Three=null, and has been collected by GC.
Two=null, and has been collected by GC.
One=null, and has been collected by GC.

One=null
Two=null
Three=null
Four=4
Five=5
```

4. 如何监控JVM的使用情况
 ```plain
 jps # 找到运行的java程序的vid
 jstat -gc [vid] 1000
 ```
 关于jstat输出：
 ```plain
 S0C: Current survivor space 0 capacity (kB).

S1C: Current survivor space 1 capacity (kB).

S0U: Survivor space 0 utilization (kB).

S1U: Survivor space 1 utilization (kB).

EC: Current eden space capacity (kB).

EU: Eden space utilization (kB).

OC: Current old space capacity (kB).

OU: Old space utilization (kB).

MC: Metaspace capacity (kB).

MU: Metacspace utilization (kB).

CCSC: Compressed class space capacity (kB).

CCSU: Compressed class space used (kB).

YGC: Number of young generation garbage collection events.

YGCT: Young generation garbage collection time.

FGC: Number of full GC events.

FGCT: Full garbage collection time.

GCT: Total garbage collection time.
 ```

### 参考资料
[Java Reference](http://san-yun.iteye.com/blog/1683558)
[SoftReference](https://docs.oracle.com/javase/7/docs/api/java/lang/ref/SoftReference.html)
[JVM监控与优化](http://www.cnblogs.com/zhguang/p/Java-JVM-GC.html)
[Oracle jstat](https://docs.oracle.com.javase/8/docs/technotes/tools/unix/jstat.html)


