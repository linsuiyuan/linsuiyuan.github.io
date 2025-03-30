---
title: "Python单例模式中那些蛋疼的问题"
date: 2025-01-01T22:57:39+08:00

categories:
  - 设计模式
  - Python
tags:
  - 单例模式
  - Python

summary: "讲述 Python 中各种形式的单例模式，以及使用这些单例模式遇到的一些问题，并提出解决方法，最后讨论各种单例的优劣。"
---

本文中讨论的单例模式都是线程安全的。

## 一、装饰器形式的单例模式

首先先给出`Python`中装饰器的单例模式：

```python
import threading  
  
def singleton(cls):  
    _instances = {}  
    _lock = threading.Lock()  
  
    def get_instance(*args, **kwargs):  
        if cls not in _instances:  
            with _lock:  
                if cls not in _instances:  
                    _instances[cls] = cls(*args, **kwargs)  
        return _instances[cls]  
  
    return get_instance
```

那么装饰器形式的单例模式会出现什么问题呢？

### 装饰器单例问题1、无法使用内置函数`isinstance()`来判断类型

使用`isinstance()`来判断单例类型的示例：

```python
@singleton  
class MyClass:...  
  
a1 = MyClass()  
a2 = MyClass()  
assert a1 is a2  
assert isinstance(a1, MyClass)
```

上面的示例执行时会触发异常：

```sh
TypeError: isinstance() arg 2 must be a type, a tuple of types, or a union
```

出错的原因是“`isinstance()` 的第二个参数必须是一个`type`、`type`元组或`union`。” 

示例中传给`isinstance()`的第二个参数是`MyClass`，这是一个类，而在`Python`中，类的类型是`type`，怎么还报错呢？

打印看一下`type(MyClass)`，输出是`function`。也就是说，加了`@singleton`之后，`MyClass`变成一个函数(`function`)了，而函数是无法作为`isinstance()`的第二个参数的。

那么有解决办法吗？一个解决办法就是，使用`functions.wraps`将`cls`包装起来，然后单例类型使用`__wrapped__`属性获取原类型。

```python
import threading  
import functools

def singleton(cls):  
    _instances = {}  
    _lock = threading.Lock()  
  
    @functools.wraps(cls)  
    def get_instance(*args, **kwargs):  
        if cls not in _instances:  
            with _lock:  
                if cls not in _instances:  
                    _instances[cls] = cls(*args, **kwargs)  
        return _instances[cls]  
  
    return get_instance

@singleton  
class MyClass:...  
  
a1 = MyClass()  
a2 = MyClass()  
assert a1 is a2  
assert isinstance(a1, MyClass.__wrapped__)
```

这样执行的时候就不会报错了。

使用`functions.wraps`没什么问题，使用`__wrapped__`属性就太不优雅了，还容易出错。`IDE`可不会提示你应该使用`__wrapped__`，你自己也无法时刻记住某个类是不是使用了单例模式。

### 装饰器单例问题2、无法使用"`|`"符号与其他类型组合成联合类型

使用"`|`"符号来表示联合类型是 `Python3.10` 推出的功能。

"`|`"和单例模式一起使用的示例：

```python
@singleton  
class MyClass:...  
  
a1: MyClass | None = None
```

示例执行时报错：

```sh
TypeError: unsupported operand type(s) for |: 'function' and 'NoneType'
```

报错原因是“`'function'和'NoneType'之间不支持 ｜ 操作`”，还是`'function'`的锅。

使用了装饰器单例模式的类，就不能使用`|`符号来组合类型了，蛋疼。

当然，也不是没有解决之道，可以使用`typing`模块的功能。

```python
a1: Optional[MyClass] = None
# 或者
a1: Union[MyClass, None] = None
```

但是这样的话，会造成风格不统一（有的使用`typing.Union`来组合类型，有的使用`|`符号）。或者要风格统一的话（都用`typing`模块），就不能使用`|`符号的新功能。

## 二、元类形式的单例模式

以上两个单例问题之所以存在，是因为装饰器将类包装成了一个函数，而函数的类型是`function`，`function`无法使用`type`的一些功能。

那么不使用装饰器，使用其他形式（比如元类）的单例模式，是不是就没有以上的问题呢？确实是。

元类形式的单例模式如下：

```python
class SingletonMeta(type):  
    _instances = {}  
    _lock = threading.Lock()  
  
    def __call__(cls, *args, **kwargs):  
        if cls not in cls._instances:  
            with cls._lock:  
                if cls not in cls._instances:  
                    cls._instances[cls] = super().__call__(*args, **kwargs)  
        return cls._instances[cls]
```

测试`isinstance()`：

```python
class MyClass(metaclass=SingletonMeta):...  
  
a1 = MyClass()  
a2 = MyClass()  
assert a1 is a2  
assert isinstance(a1, MyClass)
```

没问题。

再测试`|`符号：

```python
class MyClass(metaclass=SingletonMeta):...  

a1: MyClass | None = None
```

也没有问题。

元类形式的单例模式，似乎挺完美的，因为它能解决装饰器单例模式的缺陷。

它真的完美吗？并不。

### 元类单例问题、可能无法继承或实现同样使用了元类的类或接口

元类形式的单例模式，如果想继承或实现另外一个同样使用了元类的类或接口，就会出现问题。

```python
from abc import ABC, abstractmethod  

class Flyable(ABC):  
    @abstractmethod  
    def fly(self):...  
  
class MyClass(Flyable, metaclass=SingletonMeta):  
    def fly(self):  
        print("fly")
```

以上代码执行的时候报错：

```sh
TypeError: metaclass conflict: the metaclass of a derived class must be a (non-strict) subclass of the metaclasses of all its bases
```

出错原因是：“元类冲突：派生类的 元类 必须是其所有基类的 元类 的（非严格）子类。”

`abc`模块中的`ABC`类使用了元类`ABCMeta`，`MyClass`使用了元类`SingletonMeta`，`SingletonMeta`并不是`ABCMeta`的子类，所以出现了元类冲突。

有什么解决办法吗？有的，那就是让`SingletonMeta`成为`ABCMeta`的子类。

修改后的代码如下：

```python
import threading
from abc import ABC, abstractmethod, ABCMeta  

class _SingletonMeta(type):  
    _instances = {}  
    _lock = threading.Lock()  
  
    def __call__(cls, *args, **kwargs):  
        if cls not in cls._instances:  
            with cls._lock:  
                if cls not in cls._instances:  
                    cls._instances[cls] = super().__call__(*args, **kwargs)  
        return cls._instances[cls]  

# 让`SingletonMeta`成为`ABCMeta`的子类
class SingletonMeta(_SingletonMeta, ABCMeta): ...  
  
class Flyable(ABC):  
    @abstractmethod  
    def fly(self):...  
  
class MyClass(Flyable, metaclass=SingletonMeta):  
    def fly(self):  
        print("fly")

a1 = MyClass()  
a2 = MyClass()  
assert a1 is a2
```

可见元类形式的单例模式，也不是完美的。好在这种打补丁的方法对用户是透明的，不需要修改客户端的代码。

元类形式的单例模式，目前就发现这一个问题。如果有其他问题，等发现了再来补充。

## 三、模块级单例模式和类属性单例

在 `Python` 中，模块本身是单例，可以将单例对象定义在模块级别，这样在导入模块时，就会得到同一个实例。

```python
# singleton.py
class Singleton:...

singleton_instance = Singleton()

```

单例模式还可以通过类属性来实现，可以在类中定义一个类属性来存储实例，并在 `__new__` 方法中控制实例的创建。

```python

class Singleton:
    _instance = None
    _lock = threading.Lock() 

    def __new__(cls, *args, **kwargs):
        if not cls._instance:
	        with cls._lock:
		        if not cls._instance:
		            cls._instance = super(Singleton, cls).__new__(cls)
        return cls._instance

# 使用示例
singleton1 = Singleton()
singleton2 = Singleton()
assert singleton1 is singleton2

```

问题在于，这两种形式的单例模式，都是无法通用的，即要针对每一种类型都进行单独的实现。

## 四、总结

Python的单例模式，似乎没有一个完美的实现形式，只能在“矮子里拔将军”。

不能通用的单例模式不必再说。能通用的形式，装饰器单例也不太好，因为会改变原类型，容易影响客户端的代码实现。也就元类形式的单例能看一看了，虽然可能需要打补丁，但至少对用户透明，不会影响客户端的代码。

## 五、环境说明

- **操作系统**: macOS 15.2
- **编程语言**: Python 3.10
- **开发工具**: 
	- PyCharm 2024.3.1
	- Jupyter Notebook 7.3.2
