title: 关于Java局部类不能访问外围的非final局部变量的探索
date: 2016-07-04 21:04:47
tags: [java, inner-class]
categories: java
---

### 实验
关于java中的内部类，有很多坑，其中条就是：
>对于定义在函数中的内部类而言，在内部类中可以访问外部函数的局部变量，但这些局部变量必须被申明为final。

<!--more-->

为了清晰，首先用例子探索一下：

```java
import java.util.Date;

/**
 * Created by paranoidq on 16/7/4.
 */
public class LocalInnerClassDemo {

    public void demo() {
        int counter = 0;
        String str = "test";

        Date[] dates = new Date[100];
        for (int i = 0; i < dates.length; i++) {
            dates[i] = new Date() {
                @Override
                public int compareTo(Date anotherDate) {
                    System.out.println(counter);    // case1: int不修改 -> 通过
                    System.out.println(counter++);  // case2: int修改 -> compiler error
                    System.out.println(str);        // case3: string不修改 -> 通过
                    System.out.println(str + "t");  // case4: string为不可变对象 -> 通过

                    str = "aaa";
                    System.out.println(str);        // case5: 修改了string -> 不通过
                    // Error:
                    // Variable str is accessed from within inner class,
                    // need to be final or effectively final
                    return super.compareTo(anotherDate);
                }
            };
        }
    }
}
```
### 分析
我们发现，其实编译器足够智能，对于case1和case3而言，虽然访问了非final局部变量，但是还是通过编译了，而只有在case2、case5中修改了局部变量时，才报错。而对于case4而言，涉及到string对象不可变的另一个知识点，这里略过。

分析报错的提示，可以知道，实际上对于局部类访问外部变量的规则，相对比较宽松，只要是`final or effectively final`即可，所谓`effectively final`其实也就是在局部类内没有做出实质性的修改动作，这一类情况编译器也是让过的。

### 原因
为什么在局部类内不能访问外部的非final局部变量呢？参考[这个帖子](http://android.blog.51cto.com/268543/384844)，写的很到位。引用如下

这是一个编译器设计的问题，如果你了解java的编译原理的话很容易理解。  
首先，内部类被编译的时候会生成一个单独的内部类的.class文件，这个文件并不与外部类在同一class文件中。  
当外部类传的参数被内部类调用时，从java程序的角度来看是直接的调用例如： 

```java
public void dosome(final String a,final int b){  
  class Dosome{public void dosome(){System.out.println(a+b)}};  
  Dosome some=new Dosome();  
  some.dosome();  
}  
```

从代码来看好像是那个内部类直接调用的a参数和b参数，但是实际上不是，在java编译器编译以后实际的操作代码是:
```
class Outer$Dosome{  
    public Dosome(final String a,final int b){  
        this.Dosome$a=a;  
        this.Dosome$b=b;  
    }  
    public void dosome(){  
        System.out.println(this.Dosome$a+this.Dosome$b);  
    }  
} 
``` 
从以上代码看来，内部类并不是直接调用方法传进来的参数，而是内部类将传进来的参数通过自己的构造器备份到了自己的内部，自己内部的方法调用的实际是自己的属性而不是外部类方法的参数。  

这样理解就很容易得出为什么要用final了，因为两者从外表看起来是同一个东西，实际上却不是这样，如果内部类改掉了这些参数的值也不可能影响到原参数，然而这样却失去了参数的一致性，因为从编程人员的角度来看他们是同一个东西，如果编程人员在程序设计的时候在内部类中改掉参数的值，但是外部调用的时候又发现值其实没有被改掉，这就让人非常的难以理解和接受，为了避免这种尴尬的问题存在，所以编译器设计人员把内部类能够使用的参数设定为必须是final来规避这种莫名其妙错误的存在。”

(简单理解就是，拷贝引用，为了避免引用值发生改变，例如被外部类的方法修改等，而导致内部类得到的值不一致，于是用final来让该引用不可改变)


### 参考
内部类的原理分析：[http://android.blog.51cto.com/268543/384809](http://android.blog.51cto.com/268543/384809)
