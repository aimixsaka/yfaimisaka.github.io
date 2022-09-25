# Python中的装饰器


## ? 前言

Python中的一切皆对象。同样函数也是如此。

装饰器（形如@staticmethod, @overload等）其实是一个`装饰器函数或一个装饰器类`对`装饰器装饰的函数`的`对象`的调用再赋值的过程, 表面上体现为以函数对象作为参数。

### ? 实质

{{< admonition tip "理解装饰器">}}

```python
@decorator

function_name()

# 等价于：

function_name = decorator(function_name)

#（其中，有这么一个装饰器函数：

def decorator(f):

		@wraps(f)

		def xxx(xx, xx...):

				... # 调用前

				return f()  # 或者直接f()来调用函数

				...  # 调用后		return xxx
```

{{< /admonition >}}



## ? 普通装饰器

(直接拿菜鸟教程的例子)

- 示例一：

```python
def a_new_decorator(a_func):
    def wrap_the_function():
        print("I am doing some boring work before executing a_func()")
        a_func()
        print("I am doing some boring work after executing a_func()")
    return warp_the_function

def a_function_requiring_decoration():
    print("I am the function which needs some decoration to remove my foul smell")
   
a_function_requiring_decoration()
# outputs: "I am the function which needs some decoration to remove my foul smell"
a_function_requiring_decoration = a_new_decorator(a_function_requiring_decoration)
#now a_function_requiring_decoration is wrapped by wrap_the_function()

a_function_requiring_decration()
#outputs:I am doing some boring work before executing a_func()
#        I am the function which needs some decoration to remove my foul smell
#        I am doing some boring work after executing a_func()

```



- 用装饰器写

  ```python
  @a_new_decorator
  def a_function_requring_decoration():
      print("I am a function which needs some decoration to"
            "remove my foul smell")
      
  a_function_requiring_decoration()
  # outputs: I am doing some boring work before executing a_func()
  #         I am the function which needs some decoration to remove my foul smell
  #         I am doing some boring work after executing a_func()
   
  #the @a_new_decorator is just a short way of saying:
  a_function_requiring_decoration = a_new_decorator(a_function_requiring_decoration)
  ```

  

  - 特殊点：

    ```python
    print(a_function_requiring_decoration.__name__)
    # Output: wrap_the_function
    ```

    {{< admonition >}}

    并没有输出"a_function_requiring_decoration"。而被"wrap_the_function"替代了。

    {{< /admonition >}}h
    
    - 解决方案
    
      ```python
      from functools import wraps
      
      def a_new_decorator(a_func):
          @wrap(a_func)
          def wrap_the_function():
              print("I am doing some boring work before execting a_func")
              a_func()
              print("I am doing some boring work after exexting a_func()")
          return wrap_the_function
      @a_new_decorator
      def a_function_requring_decoration():
          print("I am the function which needs some decoration to "
                "remove my foul smell")
   
      print(a_function_requiring_decoration.__name__)
      # Output: a_function_requiring_decoration
      ```
    
      

## ？带参数的装饰器

- 思路： 创建一个包裹函数。在装饰器函数外面包一层。
  - 例：

```python
from functools import wraps

def logit(logfile="out.log"):
    def logging_decrator(func):
        @wraps(func)
        def wrapped_function(*args, **kwargs):
            log_string = func.__name__ + "was called"
            print(log_string)
            # 打开lgofile，并写入内容
            with open(logfile, "a") as f:
                f.write(log_string + "\n")
            return func(*args, **kargs)
        return wrapped_function
    return logging_decorator

@logit()
def myfunc1():
    pass
 
myfunc1()
# Output: myfunc1 was called
# 现在一个叫做 out.log 的文件出现了，里面的内容就是上面的字符串
 
@logit(logfile='func2.log')
def myfunc2():
    pass
 
myfunc2()
# Output: myfunc2 was called
# 现在一个叫做 func2.log 的文件出现了，里面的内容就是上面的字符串
```

## ？应用场景

{{< admonition tip >}}

授权

日志

{{< /admonition >}}

## ？装饰器类

- 当你想利用装饰器执行不止一个操作时，可以用来类来包装。

例：

```python 
from functools import wraps

class Logit(object):
    def __init__(self, logfile='out.log'):
        self.logfile = logfile
        
    def __call__(self, func):
        @wraps(func)
        def wrapped_function(*args, **kargs):
            log_string = func.__name__ + "was called"
            print(log_string)
            # 打开logfile并写入
            with open(self.logfile, 'a') as f:
                f.write(log_string + '\n')
            # 现在，发送一个通知
            self.notify()
            return func(*args, **kargs)
        return wrapped_function
    
    def notify(self):
        # 该类为logit类，什么也不做，交个子类
        pass
    
    
    
class EmailLogit(Logit):
    """
    Logit子类,实现日志记录的同时发邮件
    """
    def __init__(self, email="admin@myproject.com", *args, **kargs):
        self.email = email
        super(EmailLogit, self).__init__(*args, **kargs)
        
    def notify(self):
        # 具体实现
        ...
       
# 调用时与装饰器函数类似。使用@Logit
```

