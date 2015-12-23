title: Python decorator(译)
date: 2015-12-22 14:28:50
tags: [python, decorator, language]
categories: python
---

### Python函数就是对象

首先要理解python中函数就是object。一个例子：
```python
def shout(word="yes"):
    return word.capitalize()+"!"
print shout()
# outputs : 'Yes!'

# 因为function是对象，因此可以被赋值给变量。
# 注意这里没有用双括号，所以不是调用函数
scream = shout

# 用变量srceam调用函数
print scream()
# outputs : 'Yes!'

# 删除原来的函数名，同样可以利用scream调用函数
del shout
try:
    print shout()
except NameError, e:
    print e
    #outputs: "name 'shout' is not defined"

print scream()
# outputs: 'Yes!'
```

<!--more-->

>从上面可以看出，函数是对象，无论是定义时的名字shout，还是之后赋值的名字scream，都是变量，变量指向内存中的函数对象而已。

其次注意，可以在一个函数中定义另一函数。例子：
```python
def talk():
    # 在函数中定义另一个函数， 并且立刻使用它
    def whisper(word="yes"):
        return word.lower()+"..."

    print whisper()

# 当你调用talk的时候，whisper就会被在talk函数中调用
talk()
# outputs: "yes..."

#  但是在外部无法访问whisper函数
try:
    print whisper()
except NameError, e:
    print e
    #outputs : "name 'whisper' is not defined"*
    #Python's functions are objects
```

**总结：**
- 函数是对象，可以被赋值给变量
- 函数可以被定义在另一个函数中


### 函数引用
根据part-1中的论述，意味着：**函数可以返回另一个函数（的引用）**。例子：
```python
def getTalk(kind="shout"):
    
    # 同样在函数内部定义一些函数
    def shout(word="yes"):
        return word.capitalize()+"!"

    def whisper(word="yes") :
        return word.lower()+"...";

    # 试图返回函数，不用双括号，因为要返回的时函数对象
    if kind == "shout":
        return shout  
    else:
        return whisper

# 调用getTalk（注意，是调用！），获取返回值，并赋值给变量
talk = getTalk()      

# talk是一个function object，并且这个对象是getTalk返回的其中一个对象
print talk
#outputs : <function shout at 0xb7ea817c>
print talk()
#outputs : Yes!

# 当然你也可以不赋值，直接使用。
print getTalk("whisper")()
#outputs : yes...
``` 

Further, 你可以将函数作为参数传递。例子
```python
def doSomethingBefore(func): 
    print "I do something before then I call the function you gave me"
    print func()

doSomethingBefore(scream)
#outputs: 
#I do something before then I call the function you gave me
#Yes!
```

### Handcrafted decorators(自己实现decorator)

**什么是decorator**
一种包装器（wrapper），使得你不用修改函数本身就可以在函数执行前或执行后添加额外的代码。

```python
# decorator是一个函数，它将另一个函数作为它的参数
def my_shiny_new_decorator(a_function_to_decorate):

    # 在decorator内部，它动态定义一个函数（wrapper）。这个函数包装在作为参数的函数外面，从而可以在原有函数执行前后执行一些额外的代码。
    def the_wrapper_around_the_original_function():

        # 函数前执行的代码
        print "Before the function runs"

        # 调用被装饰的函数
        a_function_to_decorate()

        # 函数后执行的代码
        print "After the function runs"

    # 注意，被修饰的函数至今没有执行过！
    # 返回包装函数，其中包含了实际函数以及包装在前后的额外代码
    return the_wrapper_around_the_original_function

# 假设你定义了一个你之后不能修改的函数
def a_stand_alone_function():
    print "I am a stand alone function, don't you dare modify me"

a_stand_alone_function() 
#outputs: I am a stand alone function, don't you dare modify me

# 但是你想扩充一下上面函数的功能，如何做到？使用decorator包装一下。
a_stand_alone_function_decorated = my_shiny_new_decorator(a_stand_alone_function)
# 调用返回的包装函数即可，在包装函数内部会执行原有函数
a_stand_alone_function_decorated()
#outputs:
#Before the function runs
#I am a stand alone function, don't you dare modify me
#After the function runs
```

但是你想让它更机智一点：每次我调用`a_stand_alone_function`的时候，就会自行调用`a_stand_alone_function_decorated`，而不是我需要自己记住不去调用`a_stand_alone_function`。可以！覆盖`a_stand_alone_function`这个变量名就ok。
```python
a_stand_alone_function = my_shiny_new_decorator(a_stand_alone_function)
a_stand_alone_function()
#outputs:
#Before the function runs
#I am a stand alone function, don't you dare modify me
#After the function runs
```
以上就是decorator所做的事情了。简单吧！？


### Decorators demystified(更简便地使用decorator)

使用`@`操作符。例子：
```python
@my_shiny_new_decorator
def another_stand_alone_function():
    print "Leave me alone"

another_stand_alone_function()  
#outputs:  
#Before the function runs
#Leave me alone
#After the function runs
```
实际上，`@decorator`是`another_stand_alone_function = my_shiny_new_decorator(another_stand_alone_function)
`的简写形式而已。

decorator只是包装设计模式的一种pyhon表达而已，python中其实还嵌入了其他设计模式，例如迭代器。

当然，你可以嵌套decorator：
```python
def bread(func):
    def wrapper():
        print "</''''''\>"
        func()
        print "<\______/>"
    return wrapper

def ingredients(func):
    def wrapper():
        print "#tomatoes#"
        func()
        print "~salad~"
    return wrapper

def sandwich(food="--ham--"):
    print food

sandwich()
#outputs: --ham--
sandwich = bread(ingredients(sandwich))
sandwich()
#outputs:
#</''''''\>
# #tomatoes#
# --ham--
# ~salad~
#<\______/>
```

用python的`@decorator`语法等价于：
```python
@bread
@ingredients
def sandwich(food="--ham--"):
    print food

sandwich()
#outputs:
#</''''''\>
# #tomatoes#
# --ham--
# ~salad~
#<\______/>
```

需要注意的时decorator的次序，最先声明的decorator被包装在最外面。
```python
@ingredients
@bread
def strange_sandwich(food="--ham--"):
    print food

strange_sandwich()
#outputs:
##tomatoes#
#</''''''\>
# --ham--
#<\______/>
# ~salad~
```

### 继续深入：传递参数给被包装函数
```python
# 很简单，你只需要让decorator中定义的包装函数传递参数即可
def a_decorator_passing_arguments(function_to_decorate):
    def a_wrapper_accepting_arguments(arg1, arg2):
        print "I got args! Look:", arg1, arg2
        function_to_decorate(arg1, arg2)
    return a_wrapper_accepting_arguments

# 当你调用返回的函数对象时，实际上是调用wrapper。
# 传递给wrapper的参数会被自动传递给里面被修饰的函数
@a_decorator_passing_arguments
def print_full_name(first_name, last_name):
    print "My name is", first_name, last_name
print_full_name("Peter", "Venkman")
# outputs:
#I got args! Look: Peter Venkman
#My name is Peter Venkman
```

### Decorating methods（包装类的方法）

方法和一般函数一样处理，要注意一点：考虑self引用。

```python
def method_friendly_decorator(method_to_decorate):
    def wrapper(self, lie):
        lie = lie - 3 # very friendly, decrease age even more :-)
        return method_to_decorate(self, lie)
    return wrapper

class Lucy(object):
    def __init__(self):
        self.age = 32

    @method_friendly_decorator
    def sayYourAge(self, lie):
        print "I am %s, what did you think?" % (self.age + lie)

l = Lucy()
l.sayYourAge(-3)
#outputs: I am 26, what did you think?
```

### 定义通用的generator

使用`*args`, `**kwargs`。
```python
def a_decorator_passing_arbitrary_arguments(function_to_decorate):
    # wrapper可以接受任何参数
    def a_wrapper_accepting_arbitrary_arguments(*args, **kwargs):
        print "Do I have args?:"
        print args
        print kwargs
    
        function_to_decorate(*args, **kwargs)
    return a_wrapper_accepting_arbitrary_arguments

@a_decorator_passing_arbitrary_arguments
def function_with_no_argument():
    print "Python is cool, no argument here."

function_with_no_argument()
#outputs
#Do I have args?:
#()
#{}
#Python is cool, no argument here.

@a_decorator_passing_arbitrary_arguments
def function_with_arguments(a, b, c):
    print a, b, c

function_with_arguments(1,2,3)
#outputs
#Do I have args?:
#(1, 2, 3)
#{}
#1 2 3 

@a_decorator_passing_arbitrary_arguments
def function_with_named_arguments(a, b, c, platypus="Why not ?"):
    print "Do %s, %s and %s like platypus? %s" %\
    (a, b, c, platypus)

function_with_named_arguments("Bill", "Linus", "Steve", platypus="Indeed!")
#outputs
#Do I have args ? :
#('Bill', 'Linus', 'Steve')
#{'platypus': 'Indeed!'}
#Do Bill, Linus and Steve like platypus? Indeed!

class Mary(object):
    def __init__(self):
        self.age = 31

    @a_decorator_passing_arbitrary_arguments
    def sayYourAge(self, lie=-3): # You can now add a default value
        print "I am %s, what did you think ?" % (self.age + lie)

m = Mary()
m.sayYourAge()
#outputs
# Do I have args?:
#(<__main__.Mary object at 0xb7d303ac>,)
#{}
#I am 28, what did you think?
```

### 传递参数给decorator

前面我们讨论了如何传递参数给被包装的函数，那么如何传递参数给decorator呢？
这有点困难，因为decorator需要一个函数对象作为参数，所以你不能直接把被包装函数的参数传递给decorator本身。

先讨论一个相关的问题：
```python
# decorator是普通函数！
def my_decorator(func):
    print "I am an ordinary function"
    def wrapper():
        print "I am function returned by the decorator"
        func()
    return wrapper


def lazy_function():
    print "zzzzzzzz"

# my_decorator被调用了，因此其中的print被执行。
# 注意wrapper没有被执行！所以没有输出里面的内容以及lazy_function的内容
decorated_function = my_decorator(lazy_function)
#outputs: I am an ordinary function

# 同样，以`@`方式包装也会输出I am ...
@my_decorator
def lazy_function():
    print "zzzzzzzz"

#outputs: I am an ordinary function
```
所以，当你使用`@decorator`的时候，你实际上在让python调用`decorator`这个变量所标示的函数。而这个label可以直接指向decorator，也可以不。

请看：
```python
def decorator_maker():

    print "I make decorators! I am executed only once: "+\
          "when you make me create a decorator."

    def my_decorator(func):

        print "I am a decorator! I am executed only when you decorate a function."

        def wrapped():
            print ("I am the wrapper around the decorated function. "
                  "I am called when you call the decorated function. "
                  "As the wrapper, I return the RESULT of the decorated function.")
            return func()

        print "As the decorator, I return the wrapped function."

        return wrapped

    print "As a decorator maker, I return a decorator"
    return my_decorator

# 通过decorator_maker创建一个decorator
new_decorator = decorator_maker()       
#outputs:
#I make decorators! I am executed only once: when you make me create a decorator.
#As a decorator maker, I return a decorator

# 然后我们利用decorator包装函数
def decorated_function():
    print "I am the decorated function."
decorated_function = new_decorator(decorated_function)
#outputs:
#I am a decorator! I am executed only when you decorate a function.
#As the decorator, I return the wrapped function

# 调用包装后的函数
decorated_function()
#outputs:
#I am the wrapper around the decorated function. I am called when you call the decorated function.
#As the wrapper, I return the RESULT of the decorated function.
#I am the decorated function.
```
我们继续简化我们的代码：
```python
def decorated_function():
    print "I am the decorated function."
decorated_function = decorator_maker()(decorated_function)
#outputs:
#I make decorators! I am executed only once: when you make me create a decorator.
#As a decorator maker, I return a decorator
#I am a decorator! I am executed only when you decorate a function.
#As the decorator, I return the wrapped function.

# 调用包装后的函数
decorated_function()    
#outputs:
#I am the wrapper around the decorated function. I am called when you call the decorated function.
#As the wrapper, I return the RESULT of the decorated function.
#I am the decorated function.
```

在进一步简短些：
```python
# @语法是一次函数调用+包装，不是简单的直接包装
@decorator_maker()
def decorated_function():
    print "I am the decorated function."
#outputs:
#I make decorators! I am executed only once: when you make me create a decorator.
#As a decorator maker, I return a decorator
#I am a decorator! I am executed only when you decorate a function.
#As the decorator, I return the wrapped function.

#Eventually: 
decorated_function()    
#outputs:
#I am the wrapper around the decorated function. I am called when you call the decorated function.
#As the wrapper, I return the RESULT of the decorated function.
#I am the decorated function.
```

我们利用`@`语法进行了一次函数调用，而不仅仅是像原来一样直接包装一个函数。而`@`进行函数调用返回后的decorator才是用来包装原有函数的！！！

**因此：如果我们可以定义一个能够动态生成decorator的函数，那么就可以通过这个函数传递参数给decorator了。** 所以这里需要三层的嵌套，比之前的handcrafted decorator多了一层。这多出来的一层正是为了传递参数用的！

```python
def decorator_maker_with_arguments(decorator_arg1, decorator_arg2):

    print "I make decorators! And I accept arguments:", decorator_arg1, decorator_arg2

    def my_decorator(func):
        # 传递参数的能力源于闭包这个概念，请参阅：http://stackoverflow.com/questions/13857/can-you-explain-closures-as-they-relate-to-python
        print "I am the decorator. Somehow you passed me arguments:", decorator_arg1, decorator_arg2

        # 不要混淆decorator参数和函数参数，function_arg1/2是函数的参数
        # 作用是传递参数给真正的函数
        def wrapped(function_arg1, function_arg2) :
            print ("I am the wrapper around the decorated function.\n"
                  "I can access all the variables\n"
                  "\t- from the decorator: {0} {1}\n"
                  "\t- from the function call: {2} {3}\n"
                  "Then I can pass them to the decorated function"
                  .format(decorator_arg1, decorator_arg2,
                          function_arg1, function_arg2))
            return func(function_arg1, function_arg2)

        return wrapped

    return my_decorator

@decorator_maker_with_arguments("Leonard", "Sheldon")
def decorated_function_with_arguments(function_arg1, function_arg2):
    print ("I am the decorated function and only knows about my arguments: {0}"
           " {1}".format(function_arg1, function_arg2))

decorated_function_with_arguments("Rajesh", "Howard")
#outputs:
#I make decorators! And I accept arguments: Leonard Sheldon
#I am the decorator. Somehow you passed me arguments: Leonard Sheldon
#I am the wrapper around the decorated function. 
#I can access all the variables 
#   - from the decorator: Leonard Sheldon 
#   - from the function call: Rajesh Howard 
#Then I can pass them to the decorated function
#I am the decorated function and only knows about my arguments: Rajesh Howard
```

当然你也可以同样用`*args`,和`**kwargs`来定义一个通用的decorator_maker。但是需要记住：
decorator只会被调用一次，一旦你用decorator装饰了一个函数之后，就不能再动态设定decorator的参数了。也就是，当你`import`的时候，函数就已经decoratored了。（？？？）


### Decorating a decorator
Bonus, 让任何decorator接受任何参数（跟之前的方式一样）

```python
def decorator_with_args(decorator_to_enhance):
    """ 
    这个函数用作decorator，它修饰另一个作为decorator的函数
    这种方式使得任何decorator可以接受任意的参数了
    """

    # 我们用同样的手段传递参数
    def decorator_maker(*args, **kwargs):

        def decorator_wrapper(func):
            # 我们返回原来的decorator，它只是一个普通的函数，只是有一点需要注意，返回的decorator必须限制为这样的签名，否则不会奏效。
            return decorator_to_enhance(func, *args, **kwargs)

        return decorator_wrapper

    return decorator_maker
```

我们可以这样调用它：
```python
# 定义作为decorator的函数，并且用另一个decorator包装它。
# 记住，函数的签名必须是 'decorator(func, *args, **kwargs)'
@decorator_with_args 
def decorated_decorator(func, *args, **kwargs): 
    def wrapper(function_arg1, function_arg2):
        print "Decorated with", args, kwargs
        return func(function_arg1, function_arg2)
    return wrapper

# 然后你就可以用你刚刚定义的decorated decorator去包装一个函数了
@decorated_decorator(42, 404, 1024)
def decorated_function(function_arg1, function_arg2):
    print "Hello", function_arg1, function_arg2

decorated_function("Universe and", "everything")
#outputs:
#Decorated with (42, 404, 1024) {}
#Hello Universe and everything

# Whoooot!
```

### Decorator最佳实践

几点关于decorator的注意：
1. > python 2.4
2. decorator会减慢函数调用的速度，不要滥用
3. 不能un-decorate一个函数，一旦包装了之后，就不能解除了（即使可以，一般也不这么做）
4. decorator使得函数的调试变得困难


`functools.wraps()`（在python2.5版本引入）,它拷贝被包装函数的名字、模块和docstring到包装器中。
```python
# For debugging, the stacktrace prints you the function __name__
def foo():
    print "foo"

print foo.__name__
#outputs: foo

# decorator会隐藏实际的函数名   
def bar(func):
    def wrapper():
        print "bar"
        return func()
    return wrapper

@bar
def foo():
    print "foo"

# 返回的时wrapper。但是明明实际的函数是foo啊。
print foo.__name__
#outputs: wrapper

# functools可以帮我们解决这个问题

import functools

def bar(func):
    # We say that "wrapper", is wrapping "func"
    # and the magic begins
    @functools.wraps(func)
    def wrapper():
        print "bar"
        return func()
    return wrapper

@bar
def foo():
    print "foo"

print foo.__name__
#outputs: foo

```

### decorator到底有什么用处？

典型的应用：
1. 为一个你不能修改的lib添加一些功能
2. 用于调试

```python
def benchmark(func):
    """
    A decorator that prints the time a function takes
    to execute.
    """
    import time
    def wrapper(*args, **kwargs):
        t = time.clock()
        res = func(*args, **kwargs)
        print func.__name__, time.clock()-t
        return res
    return wrapper


def logging(func):
    """
    A decorator that logs the activity of the script.
    (it actually just prints it, but it could be logging!)
    """
    def wrapper(*args, **kwargs):
        res = func(*args, **kwargs)
        print func.__name__, args, kwargs
        return res
    return wrapper


def counter(func):
    """
    A decorator that counts and prints the number of times a function has been executed
    """
    def wrapper(*args, **kwargs):
        wrapper.count = wrapper.count + 1
        res = func(*args, **kwargs)
        print "{0} has been used: {1}x".format(func.__name__, wrapper.count)
        return res
    wrapper.count = 0
    return wrapper

@counter
@benchmark
@logging
def reverse_string(string):
    return str(reversed(string))

print reverse_string("Able was I ere I saw Elba")
print reverse_string("A man, a plan, a canoe, pasta, heros, rajahs, a coloratura, maps, snipe, percale, macaroni, a gag, a banana bag, a tan, a tag, a banana bag again (or a camel), a crepe, pins, Spam, a rut, a Rolo, cash, a jar, sore hats, a peon, a canal: Panama!")

#outputs:
#reverse_string ('Able was I ere I saw Elba',) {}
#wrapper 0.0
#wrapper has been used: 1x 
#ablE was I ere I saw elbA
#reverse_string ('A man, a plan, a canoe, pasta, heros, rajahs, a coloratura, maps, snipe, percale, macaroni, a gag, a banana bag, a tan, a tag, a banana bag again (or a camel), a crepe, pins, Spam, a rut, a Rolo, cash, a jar, sore hats, a peon, a canal: Panama!',) {}
#wrapper 0.0
#wrapper has been used: 2x
#!amanaP :lanac a ,noep a ,stah eros ,raj a ,hsac ,oloR a ,tur a ,mapS ,snip ,eperc a ,)lemac a ro( niaga gab ananab a ,gat a ,nat a ,gab ananab a ,gag a ,inoracam ,elacrep ,epins ,spam ,arutaroloc a ,shajar ,soreh ,atsap ,eonac a ,nalp a ,nam A
```

```python
@counter
@benchmark
@logging
def get_random_futurama_quote():
    from urllib import urlopen
    result = urlopen("http://subfusion.net/cgi-bin/quote.pl?quote=futurama").read()
    try:
        value = result.split("<br><b><hr><br>")[1].split("<br><br><hr>")[0]
        return value.strip()
    except:
        return "No, I'm ... doesn't!"


print get_random_futurama_quote()
print get_random_futurama_quote()

#outputs:
#get_random_futurama_quote () {}
#wrapper 0.02
#wrapper has been used: 1x
#The laws of science be a harsh mistress.
#get_random_futurama_quote () {}
#wrapper 0.01
#wrapper has been used: 2x
#Curse you, merciful Poseidon!
```

另外，python自己就提供了一些decorators，例如：property, statismethod等
- Django利用decorators来管理caching和view的权限
- Twisted to fake inlining asynchronous functions calls.（不懂==）

decorator的应用值得好好进一步探索。

### 参考资料：
[StackOverflow Answer](http://stackoverflow.com/questions/739654/how-can-i-make-a-chain-of-function-decorators-in-python/1594484#1594484)
[函数的参数](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001431752945034eb82ac80a3e64b9bb4929b16eeed1eb9000)






