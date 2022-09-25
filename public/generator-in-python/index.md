# Python中的生成器


## ? 前言

`yield`是Python极其重要的一个关键字，它与Python中的`生成器`密切相关，然而`yield`的用法和`生成器`本身时不时让人困惑...

## ? 生成器和`yield`有什么用

- 大多数情况下，使用`yield`都是为了将函数作为生成器使用(`yield`只能在函数中使用)

- 当有大量数据需要返回时生成器能够节省内存空间

## ? 一些概念

### ? 可迭代对象

> Python中的任意对象，只要它定义了可以返回一个迭代器的`__iter__`方法，或者定义了可以支持下标索引的`__getitem__`方法，那么它就是一个可迭代对象。
>
> 
>
> 简单说，可迭代对象就是能提供迭代器的任意对象

### ? 迭代器

> 任意对象，只要它定义了`next`（Python2）或`__next__`方法，那么它就是一个迭代器

### ? 迭代

> 用简单的话讲，它就是从某个地方（比如一个列表）取出一个元素的过程。当我们使用一个循环来遍历某个东西时，这个过程本身就叫迭代

## ? 生成器

### ? 概念

> 生成器也是一种迭代器，但是你只能对其迭代一次。这是因为它们并没有把所有的值存在内存中，而是在运行时生成值
>
> 你通过遍历来使用它们，要么用一个"for"循环，要么将它们传递给任意可以进行迭代的函数（比如`next()`）或结构
>
> 大多数时候生成器是以函数来实现的（当然，类也可以，但函数较为简便）。然而，它们并不返回一个值，而是`yield`（产生）一个值



### ? 一些例子

**正常思路的斐波那契:**

```python
def fab(max):
    n, a, b = 0, 0, 1
    L = [] # 拿一个列表来存数据，提高复用性
    while n < max:
        L.append(b)
        a, b = b, a + b
        n = n + 1
    return L

for n in fab(5):
    print(n)
```

这么做会使该函数的内存占用随参数`max`增大而增加。



**生成器类构建的斐波那契:**

```python
class Fab:
    def __init__(self, max):
        self.max = max
        self.n, self.a, self.b = 0, 0, 1
    def __iter__(self):
        return self
    def __next__(self):
        if self.n < self.max:
            r = self.b
            self.a, self.b = self.b, self.a + self.b
            self.n = self.n + 1
            return r
        raise StopIteration()
        
for n in Fab(5):
    print(n)
```



**生成器函数构建的斐波那契:**

```python
def fab(n):
    a = b = 1
    for i in range(n):
        yield a
        a, b = b, a + b
```

### ? 一些关于迭代器的内置函数

- `next()`  获取迭代器的下一个值
- `iter()`  以可迭代对象（比如字符串）为参数，返回一个迭代器对象


