title: Java Enum详解
date: 2016-01-17 19:43:24
tags: [java, enum, todo]
categories: [java]
---

### 不用Enum的基本写法
```java
public class CurrencyDenom { 
    public static final int PENNY = 1; 
    public static final int NICKLE = 5; 
    public static final int DIME = 10; 
    public static final int QUARTER = 25; 
} 
public class Currency { 
    private int currency; 
    // CurrencyDenom.PENNY,CurrencyDenom.NICKLE, 
    // CurrencyDenom.DIME,CurrencyDenom.QUARTER 
}
```
缺点：
1. 类型不安全，可以给currency赋值任何数，而不是限制在CurrentyDenom中的一种。编译器无法检查。
2. 无意义的输出，输出NICKLE，只会得到5，而没有NICKLE。
3. 需要以CurrencyDenom.PENNY的形式才能访问钱币值（尽管可以static import的方式解决）。

<!--more-->

### 用Enum解决上面的问题
```java
public enum Currency {
    PENNY(1), 
    NICKLE(5), 
    DIME(10), 
    QUARTER(25)
    ;

    private int value;
    Currency(value) {
        this.value = value;
    }
}
```
1. 使用Currency作为变量的类型，严格限制变量只能取定义的几个值，type-safe。
2. toString()会输出PENNY等有意义的值。
3. 可以直接访问PENNY。

### Enum的基本解释
1. java关键字
2. 类似于Class或Interface
3. enum对象都继承自java.lang.Enum
4. enum中定义的常量默认是static final的
5. 可以用于switch语句

### Enum的关键点
#### 基本特性

1. type safe: 必须引用定义的常量值，不可以随意赋值，即使直观上一样，例如硬币的面值，1直观上也可以，但是没有在enum中定义常量，就是编译不能通过。编译器保证type-safe。
 ```java
public enum Currency { 
    PENNY, NICKLE, DIME, QUARTER }; 
    Currency coin = Currency.PENNY; 
    coin = 1; //compilation error 
 ```

2. 类似于Class或Interface，可以定义构造器（私有）、方法和域。
3. 可以给常量以参数，等于是调用了有参构造器
 ```java
public enum Currency { 
    PENNY(1), NICKLE(5), DIME(10), QUARTER(25); 
    private int value; 
    private Currency(int value) { this.value = value; } 
};
 ```
4. enum常量默认是static final，不能更改
5. 因为是常量，所以可以用`==`直接比较两个enum对象
 ```java
 Currency usCoin = Currency.DIME; 
 if(usCoin == Currency.DIME){ 
    System.out.println("enum in java can be compared using =="); 
}
 ```


#### 高级特性
1. 可以重写java.lang.Enum的方法，例如`toString()`等。继承的方法有：
 ```java
 toString();
 // 貌似我查了一下源码，可重写的方法只有toString()了。
 ```
 注意，下面几个方法在JDK1.8的版本中，我查看了下，不能继承了。[http://www.cnblogs.com/frankliiu-java/archive/2010/12/07/1898721.html](http://www.cnblogs.com/frankliiu-java/archive/2010/12/07/1898721.html)这篇参考的博文中的相关说法我认为可能是之前的JDK版本。
 ```java
 // 单例，所以所有子类维持固定的语义
 public final boolean equals(Object other) {
        return this==other;
    }
 // 单例，与equals维持同样的语义即可
 public final int hashCode() {
    return super.hashCode();
}
 // 不支持clone，很明显，单例！
 protected final Object clone() {
    throw new CloneNotSupportedException();
 }

 // 定义顺序，不允许更改
 public final int ordinal() {
        return ordinal;
    }

 // 比较的是ordinal！
 public final int compareTo(E o) {
        Enum<?> other = (Enum<?>)o;
        Enum<E> self = this;
        if (self.getClass() != other.getClass() && // optimization
            self.getDeclaringClass() != other.getDeclaringClass())
            throw new ClassCastException();
        return self.ordinal - other.ordinal;
    }
 // 不允许有finalized方法
 protected final void finalize() { }
 ```
2. 编译器生成的values()静态方法，返回enum对象的数组。 顺序与定义顺序相同。
 ```java
 for(Currency coin: Currency.values()){ 
    System.out.println("coin: " + coin); 
 }
 ```
  java.lang.Enum自带静态方法`valueOf()`，当然可以直接用于子类。
  (**为什么valueOf在超类可以定义，而values()则要由编译器生成呢？？**)
3. ordinal()方法: 返回枚举值在枚举类种的顺序。这个顺序根据枚举值声明的顺序而定
```java
Currency.DIME // 返回3
```
4. EnumSet和EnumMap是支持Enum存储的高效的集合类，应该尽可能使用它们。
 EnumSet提供一些工厂方法，创建存储enum实例的集合，例如`EnumSet.of()`。它的存储方式与普通的set不同。简单一点：当小于64个实例的时候，使用`RegularEnumSet`，而大于64个实例的时候，使用`JumboEnumSet`。([两者的不同点](http://java67.blogspot.com/2013/11/difference-between-regularenumset-and-jumboenumset-java.html))

5. enum可以实现接口。默认已经实现了Serializable和Comparable接口。
 ```java
 public enum Currency implements Runnable{ 
    PENNY(1), NICKLE(5), DIME(10), QUARTER(25); 
    private int value; 
    ............ 

    @Override public void run() { 
        System.out.println("Enum in Java implement interfaces"); 
    } 
}
 ```
6. 可以在enum中定义抽象方法，但是定义常量的时候必须提供实现。（在enum实现策略模式中很有用）
 ```java
 public enum Currency {
    PENNY(1) {
        @Override
        public String color() {
            return "copper";
        }
    },
    NICKLE(5) {
        @Override
        public String color() {
            return "bronze";
        }
    },
    DIME(10) {
        @Override
        public String color() {
            return "silver";
        }
    },
    QUARTER(25) {
        @Override
        public String color() {
            return "silver";
        }
    };
    private int value;

    public abstract String color();

    private Currency(int value) {
        this.value = value;
    }
} 

System.out.println("Color: " + Currency.DIME.color());
 ```

7. enum的默认序列化读取方式被禁用
 ```java
 private void readObject(ObjectInputStream in) throws IOException,
        ClassNotFoundException {
        throw new InvalidObjectException("can't deserialize enum");
    }

 private void readObjectNoData() throws ObjectStreamException {
        throw new InvalidObjectException("can't deserialize enum");
    }
 ```
 [关于Java Enum类型的序列化机制解析](http://mysun.iteye.com/blog/1581119)

8. convert String to Enum: `valueOf()`
 `public static <T extends Enum<T>> T valueOf(Class<T> enumType, String name)`
 - 静态方法
 - 需要给定enum类名和常量的名字，返回enum的某一个常量实例。如果找不到这个常量实例，则抛出`IllegalArgumentException`。

9. convert Enum to String: `name()`
 java.lang.Enum中的final方法，返回定义时的常量名字，与toString的区别在于`toString()`默认返回name，但是可以被子类重写，但是`name()`不可以重写，它一定返回exact defined name。

### Enum使用案例
1. 单例模式
```java
/**
 * Created by paranoidq on 16/1/16.
 */
public enum SingletonWithEnum {

    RED(1),
    BLUE(2),
    GREEN(3)
    ;

    /**
     *  私有变量
     */
    private int nCode;

    SingletonWithEnum(int nCode) {
        this.nCode = nCode;
    }
    
    @Override
    public String toString() {
        return String.valueOf(this.nCode);
    }

    /**
     * Other methods
     */
    public void otherMethods() {
        System.out.println("Other methods");
    }
}
```
 [Why Enum as a singleton is better in java](http://javarevisited.blogspot.com/2012/07/why-enum-singleton-are-better-in-java.html)

2. 策略模式
 ```java
 public class Match { 
    private static final Logger logger = LoggerFactory.getLogger(Match.class); public static void main(String args[]) { 
        Player ctx = new Player(Strategy.T20); 
        ctx.play(); 
        ctx.setStrategy(Strategy.ONE_DAY); 
        ctx.play(); 
        ctx.setStrategy(Strategy.TEST); 
        ctx.play(); 
    } 
} 
 ```
 ```java
 class Player{ 
    private Strategy battingStrategy; 
    public Player(Strategy battingStrategy){ 
        this.battingStrategy = battingStrategy; 
    } 
    public void setStrategy(Strategy newStrategy){ 
        this.battingStrategy = newStrategy; 
    } 
    public void play(){ 
        battingStrategy.play(); 
    } 
}
 ```
 ```java
 enum Strategy { 
    /* Make sure to score quickly on T20 games */ 
    T20 { 
        @Override public void play() { 
            System.out.printf("In %s, If it's in the V, make sure it goes to tree %n", name()); 
        } 
    }, 

    /* Make a balance between attach and defence in One day */ 
    ONE_DAY { 
        @Override public void play() { 
            System.out.printf("In %s, Push it for Single %n", name()); 
        } 
    }, 

    /* Test match is all about occupying the crease and grinding opposition */ TEST { 
        @Override public void play() { 
            System.out.printf("In %s, Grind them hard %n", name()); 
        } 
    }; 

    public void play() { 
        System.out.printf("In Cricket, Play as per Merit of Ball %n"); 
    } 
}

 ```
 缺陷是：每次添加策略都需要更改enum类，违背了开闭原则。
 参考：[Strategy pattern using enum in java](http://javarevisited.blogspot.com/2014/11/strategy-design-pattern-in-java-using-Enum-Example.html)

3. 用Enum替代String或int常量表示的固定值，如ON/OFF等
4. 实现状态机State Machine 
 参考：[Java secret using enum as state machine](http://vanillajava.blogspot.sg/2011/06/java-secret-using-enum-as-state-machine.html)


### Enum bytecode解释

```java
public class Enum {
    public enum Direction { NORTH, SOUTH, EAST, WEST };

    public static void main(String[] args) {
        Direction d = Direction.EAST;
        System.out.println(d);
    }
}
```
对应的bytecode：

```
public class Enum extends java.lang.Object{
public Enum();
Code:
0: aload_0
1: invokespecial #1; //Method java/lang/Object."":()V
4: return

public static void main(java.lang.String[]);
Code:
0: getstatic #2; //Field Enum$Direction.EAST:LEnum$Direction;
3: astore_1
4: getstatic #3; //Field java/lang/System.out:Ljava/io/PrintStream;
7: aload_1
8: invokevirtual #4; //Method java/io/PrintStream.println:(Ljava/lang/Object;)V
11: return
}
```

注意main函数的第0行bytecode。Direction实际上就是一个类(在本例中是inner class)，并且这个类有一个静态域EAST，而这个EAST还是一个Direction对象。这也解释了再定义一些Enum实例的时候需要用括号传递参数，因为实际上实在传递构造函数的参数！
```java
enum Suit {
    CLUBS(1),
    DIAMONDS(2),
    HEARTS(3),
    SPADES(4)
};
```

Direction的bytecode如下：
```
public final class Enum$Direction extends java.lang.Enum{
public static final Enum$Direction NORTH;

public static final Enum$Direction SOUTH;

public static final Enum$Direction EAST;

public static final Enum$Direction WEST;

public static final Enum$Direction[] values();
Code:
0: getstatic #1; //Field $VALUES:[LEnum$Direction;
3: invokevirtual #2; //Method "[LEnum$Direction;".clone:()Ljava/lang/Object;
6: checkcast #3; //class "[LEnum$Direction;"
9: areturn

public static Enum$Direction valueOf(java.lang.String);
Code:
0: ldc_w #4; //class Enum$Direction
3: aload_0
4: invokestatic #5; //Method java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
7: checkcast #4; //class Enum$Direction
10: areturn

static {};
Code:
0: new #4; //class Enum$Direction
3: dup
4: ldc #7; //String NORTH
6: iconst_0
7: invokespecial #8; //Method "":(Ljava/lang/String;I)V
10: putstatic #9; //Field NORTH:LEnum$Direction;
13: new #4; //class Enum$Direction
16: dup
17: ldc #10; //String SOUTH
19: iconst_1
20: invokespecial #8; //Method "":(Ljava/lang/String;I)V
23: putstatic #11; //Field SOUTH:LEnum$Direction;
26: new #4; //class Enum$Direction
29: dup
30: ldc #12; //String EAST
32: iconst_2
33: invokespecial #8; //Method "":(Ljava/lang/String;I)V
36: putstatic #13; //Field EAST:LEnum$Direction;
39: new #4; //class Enum$Direction
42: dup
43: ldc #14; //String WEST
45: iconst_3
46: invokespecial #8; //Method "":(Ljava/lang/String;I)V
49: putstatic #15; //Field WEST:LEnum$Direction;
52: iconst_4
53: anewarray #4; //class Enum$Direction
56: dup
57: iconst_0
58: getstatic #9; //Field NORTH:LEnum$Direction;
61: aastore
62: dup
63: iconst_1
64: getstatic #11; //Field SOUTH:LEnum$Direction;
67: aastore
68: dup
69: iconst_2
70: getstatic #13; //Field EAST:LEnum$Direction;
73: aastore
74: dup
75: iconst_3
76: getstatic #15; //Field WEST:LEnum$Direction;
79: aastore
80: putstatic #1; //Field $VALUES:[LEnum$Direction;
83: return
}
```

实际上，我们可以自己实现，只是Java帮我们做了这些工作，类似于这样：
```java
public class Direction extends java.lang.Enum {
    public final static Direction NORTH;
    public final static Direction SOUTH;
    public final static Direction EAST;
    public final static Direction WEST;

    static {
        NORTH = new Direction("NORTH", 0);
        SOUTH = new Direction("SOUTH", 1);
        EAST = new Direction("EAST", 2);
        WEST = new Direction("WEST", 3);

        VALUES = new Direction[4];
        VALUES[0] = NORTH;
        VALUES[1] = SOUTH;
        VALUES[2] = EAST;
        VALUES[3] = WEST;
    }
}
```
Direction继承了java.lang.Enum类，从而从这个类中继承了一些方法和域，例如: `toString()`等。

### 参考资料
[Enum examples in java](http://javarevisited.blogspot.com/2011/08/enum-in-java-example-tutorial.html)
[Java 1.5 Explained - Enum](http://boyns.blogspot.nl/2008/03/java-15-explained-enum.html)
[What does EnumSet really mean](http://stackoverflow.com/questions/11825009/what-does-enumset-really-mean)



