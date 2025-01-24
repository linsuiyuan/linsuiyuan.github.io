---
title: "Python中的函数“重载”"
date: 2025-01-24

categories:
  - 编程语言
  - Python
tags:
  - Python
  - 重载
  - 函数重载

summary: "介绍了 Python 实现重载的方法，包括 typing.overload、singledispatch、multipledispatch 和 pyoverload，并分析了各自的优缺点，最终推荐了更优雅且易用的 pyoverload 库作为 Python 函数重载的解决方案。"
---


## 其他语言的重载

**重载**是指在同一个作用域内定义多个同名但参数不同的函数或方法。如在 Java 中，重载是通过函数名和参数类型或数量的不同来实现的。

Java 重载例子：

```java
class OverloadExample {  
  
    public int add(int a, int b) {  
        return a + b;  
    }  
  
    public double add(double a, double b) {  
        return a + b;  
    }  
  
    public static void main(String[] args) {  
        OverloadExample example = new OverloadExample();  
  
        // 调用不同的add方法  
        System.out.println("两个整数的和: " + example.add(5, 10));  
        System.out.println("两个浮点数的和: " + example.add(5.5, 10.5));  
    }  
}
```

不像 Java，Python 中后面定义的函数或方法会覆盖前面的定义，因此 Python 是不支持 Java 那种形式的重载的。不过，在 Python 中可以通过一些方法来实现重载的行为。下面就来谈谈这些方法。

## Python 中的“重载”

Python 实现重载的行为，可以使用 Python 内置的一些装饰器或者使用第三方库。当然，也可以自己定义装饰器来实现，但除非是为了学习或者有特殊需求，否则没必要自己实现。

### 和重载不搭边的“重载”

第一种“重载”方法是利用 `typing.overload` 装饰器：

```python
from typing import overload  
  
@overload  
def add(a: int, b: int):...  
  
@overload  
def add(a: float, b: float):...  
  
def add(a, b):  
    if type(a) == type(b) == int:  
        print("int add")  
        return a + b  
    elif type(a) == type(b) == float:  
        print("float add")  
        return a + b  
    else:  
        return a + b  
  
print(add(1, 2))  
print(add(1.5, 2.5))
```

这种方法除了调用函数时有提示和方便类型检查，其实和重载搭不上边。真正的重载，不同参数类型的函数体的实现是在不同的函数中，而上面这个方式，实现全部在最后的一个函数中。实现时，还需要手动判断类型，重载的意义何在？

实际上，去掉前面使用 `overload` 装饰的两个函数定义，代码也是能正常运行的，输出也是一样的。`overload` 装饰器完全是为了类型检查而存在的，除此之外没一点用处。Python 是动态语言，过度追求类型检查，显得不伦不类。如果真的要追求完全的类型检查，还不如像 JavaScript 那样搞一个 TypeScript，比如搞一个 “TypePython”。

### 和重载搭半边的“重载”

第二种重载方法是使用 `singledispatch`，Python 提供了一个 `singledispatch` 装饰器来实现单分派。单分派是指根据第一个参数的类型来选择函数的实现。

`singledispatch` 例子：

```python
from functools import singledispatch  
  
@singledispatch  
def process(value):  
    print("处理未知类型", value)  
  
@process.register(int)  
def _(value):  
    print(f"处理整数: {value}")  
  
@process.register(str)  
def _(value):  
    print(f"处理字符串: {value}")  
  
process(10)  # 输出: 处理整数: 10  
process("hello")  # 输出: 处理字符串: hello
```

要注意的是，如果要实现类中的方法的单分派，则需要使用 `singledispatchmethod` 装饰器：

```python
class MyClass:  
    @singledispatchmethod  
    def process(self, value):  
        print("处理未知类型", value)  
  
    @process.register(int)  
    def _(self, value):  
        print(f"处理整数: {value}")  
  
    @process.register(str)  
    def _(self, value):  
        print(f"处理字符串: {value}")  
  
myclass = MyClass()  
myclass.process(10)  
myclass.process("hello")
```

一些读者已经注意到了，单分派无法分派（重载）多个参数的函数。

没错，官方给了一个和重载不搭边的 `overload` 装饰器，然后又给了一个只和重载“搭半边“的 `singledispatch` 装饰器。甚至这个“残缺”的装饰器，还无法用在类中的方法上，类的方法需要使用  `singledispatchmethod` 装饰器！

### 第三方库的“重载”

既然单分派装饰器无法分派多个参数的函数，那么使用多分派是不是就可以分派多个参数的函数了？

确实如此。然而气人的是，Python 官方除了提供一个没用的装饰器和一个残缺的单分派装饰器，并不提供多分派装饰器！

Python 本身不提供，第三方库倒是有提供的。

#### multipledispatch 库

`multipledispatch` 的特点：

- **多分派**：可以根据多个参数的类型来选择函数实现，而不仅仅是第一个参数。
- **简洁的语法**：使用 `@dispatch` 装饰器可以轻松定义不同参数类型的函数实现。
- **动态类型**：支持动态类型，可以在运行时根据参数类型选择合适的实现。

`multipledispatch` 的安装:

```sh
pip install multipledispatch
```

`multipledispatch` 的例子：

```python
from multipledispatch import dispatch

@dispatch(int, int)
def process(a, b):
    print(f"处理整数: {a} 和 {b}")

@dispatch(str)
def process(value):
    print(f"处理字符串: {value}")

process(10, 20)  # 输出: 处理整数: 10 和 20
process("hello")  # 输出: 处理字符串: hello
```

使用 `multipledispatch` 来实现重载函数和重载方法，都是使用 `dispatch` 装饰器，不用像官方那样使用不同的装饰器。

这里介绍 `multipledispatch` 是因为这个库使用的人比较多，其实笔者是不喜欢这个库的。因为在使用 `dispatch` 装饰器的时候，必须将类型注解传给它，否则就会触发运行时异常。比如下面这样的代码，会触发异常。

```python 
from multipledispatch import dispatch

# 这样会触发运行时异常 NotImplementedError
@dispatch
def process(a: int, b: int):  
    print(f"处理整数: {a} 和 {b}")
```

这样有种脱裤子放屁的感觉。Python 的变量都是带有类型注解的（如果没有设置，那么就是默认的 Any 类型）。既然变量已经带有类型注解，那么直接从变量上获取就好了，根本就没必要手动传给装饰器。

将类型注解设置在装饰器上，是不如直接将类型设置在变量后面直观的。`dispatch` 虽然支持装饰器上的类型注解和参数上的类型注解同时存在，就像下面这样：

```python
from multipledispatch import dispatch

@dispatch(int, int)
def process(a: int, b: int):  
    print(f"处理整数: {a} 和 {b}")
```

这样虽然看起来直观了，不过代码显得重复了，违背 `DRY` 原则。而且，也容易出错，比如说修改了其中一处的类型却忘了修改另一处相应的类型。

#### pyoverload 库

`pyoverload` 的特点：

- **函数重载**：允许定义多个同名函数，根据不同的参数类型和数量来选择合适的实现。
- **类型检查**：在调用函数时，`pyoverload` 会根据传入参数的类型进行检查，并选择匹配的函数实现。
- **简洁的语法**：使用 `@overload` 装饰器可以轻松定义不同参数类型的函数实现。

`pyoverload` 的安装：

```sh
pip install pyoverload
```

`pyoverload` 的例子：

```python
from pyoverload import overload

@overload
def add(x: int, y: int) -> int:
    return x + y

@overload
def add(x: str, y: str) -> str:
    return x + " " + y

print(add(1, 2))          # 输出: 3
print(add("Hello", "World"))  # 输出: Hello World

```

从例子中可以看到，`pyoverload` 的使用更简单，只需在相应的函数添加上 `overload` 装饰器。除此之外，`overload` 装饰器对用户来说是透明的，无需做其他事情。

不知道为什么，`pyoverload` 没有 `multipledispatch` 的缺点，使用人数反而没有 `multipledispatch` 的多。

## 总结

不管是 Python 官方还是第三方库，实现函数的重载功能都不约而同地选择了装饰器，可见装饰器几乎是实现 Python函数重载的最优解。然而并不是每个实现都能做到优雅，一个优雅的重载函数装饰器，一是要满足最基本的重载需求（官方显然没有做到），二是要对用户透明（`multipledispatch` 没有做到）。`pyoverload` 库两方面都做到了，是一个值得推荐的库。