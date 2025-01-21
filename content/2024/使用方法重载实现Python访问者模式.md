---
title: "使用方法重载实现Python访问者模式"
date: 2024-12-13 08:48:34

categories:
  - 设计模式
  - Python
tags:
  - 访问者模式
  - Python
  - 方法重载

summary: "借用 Python 的 singledispatchmethod 装饰器，以“方法重载”的方式来实现设计模式中的访问者模式"
---

Python上的访问者模式，看了一下网上其他人的例子，一般都是类似下面的代码。

```python
from abc import ABC, abstractmethod  
  
# 抽象访问者  
class AnimalVisitor(ABC):  
    @abstractmethod  
    def visit_dog(self, dog: "Dog"):  
        pass  
  
    @abstractmethod  
    def visit_cat(self, cat: "Cat"):  
        pass  
  
# 抽象动物类  
class Animal(ABC):  
    def __init__(self, name: str):  
        self.name = name  
  
    @abstractmethod  
    def accept(self, visitor: AnimalVisitor):  
        pass  
  
# 具体动物类  
class Dog(Animal):  
    def accept(self, visitor):  
        visitor.visit_dog(self)  
  
class Cat(Animal):  
    def accept(self, visitor):  
        visitor.visit_cat(self)  
  
# 具体访问者：喂食访问者  
class FeedingVisitor(AnimalVisitor):  
    def visit_dog(self, dog):  
        print(f"给{dog.name}喂狗粮")  
  
    def visit_cat(self, cat):  
        print(f"给{cat.name}喂猫粮")  
  
# 具体访问者：检查健康访问者  
class HealthCheckVisitor(AnimalVisitor):  
    def visit_dog(self, dog):  
        print(f"检查{dog.name}的疫苗接种情况")  
  
    def visit_cat(self, cat):  
        print(f"检查{cat.name}是否需要洗澡")
  
```

使用示例如下：
```python
def main():  
    # 创建动物  
    dog = Dog("旺财")  
    cat = Cat("咪咪")  
  
    # 创建访问者  
    feeding_visitor = FeedingVisitor()  
    health_visitor = HealthCheckVisitor()  
  
    # 创建动物列表  
    animals = [dog, cat]  
  
    # 执行不同的操作  
    print("=== 喂食时间 ===")  
    for animal in animals:  
        animal.accept(feeding_visitor)  
  
    print("\n=== 健康检查 ===")  
    for animal in animals:  
        animal.accept(health_visitor)
  
  
if __name__ == "__main__":  
    main()
```

以上实现的访问者模式，访问者的接口类（抽象类）一般通过定义不同的方法（带visit前缀的方法）来对不同的被访问者进行访问。有些奇怪，为什么不通过方法重载的方式来实现访问者模式呢？

通过定义不同方法来实现访问者模式，明显违背一些设计原则的。访问者的接口类需要负责访问不同方法，而这些方法之间相关性不大（删除某个方法对其他方法没有影响），这明显违背了`单一职责原则（SRP）`；如果要添加一种动物（被访问者），那么就要修改访问者接口类以添加相应的方法，这是违背`开闭原则（OCP）`的；访问者的接口类的方法还依赖了具体类（比如`Dog`、`Cat`类），这违背了`依赖倒转原则（DIP）`。

使用方法重载实现的访问者模式，则没有上面的问题，而且代码也更简单明了。Python没有传统的方法重载方式，不过在`functools`模块里有个`singledispatchmethod`单分派装饰器，这里可以借用它来实现“方法重载”。

使用方法重载的方式代码如下：

```python
from abc import ABC, abstractmethod  
from functools import singledispatchmethod  
  
# 抽象访问者  
class AnimalVisitor(ABC):  
    @abstractmethod  
    def visit(self, animal: "Animal"):  
        pass  
  
# 抽象动物类  
class Animal(ABC):  
    def __init__(self, name: str):  
        self.name = name  
  
    def accept(self, visitor: AnimalVisitor):  
        visitor.visit(self)  
  
# 具体动物类  
class Dog(Animal): ...  
  
class Cat(Animal): ...  
  
# 具体访问者：喂食访问者  
class FeedingVisitor(AnimalVisitor):  
    @singledispatchmethod  
    def visit(self, animal: Animal):  
        raise NotImplementedError(f"{type(animal)} 未重载 visit方法")  
  
    @visit.register(Dog)  
    def _(self, dog):  
        print(f"给{dog.name}喂狗粮")  
  
    @visit.register(Cat)  
    def _(self, cat):  
        print(f"给{cat.name}喂猫粮")  
  
# 具体访问者：检查健康访问者  
class HealthCheckVisitor(AnimalVisitor):  
  
    @singledispatchmethod  
    def visit(self, animal: Animal):  
        raise NotImplementedError(f"{type(animal)} 未重载 visit方法")  
  
    @visit.register(Dog)  
    def _(self, dog):  
        print(f"检查{dog.name}的疫苗接种情况")  
  
    @visit.register(Cat)  
    def _(self, cat):  
        print(f"检查{cat.name}是否需要洗澡")
```

可以看到，这一种方式访问者抽象类`AnimalVisitor`的方法只有一个，就是`visit`。`AnimalVisitor`只负责一个职责，那就是访问动物，符合`单一职责原则`；添加其他种动物的时候，不用修改抽象类，符合`开闭原则`；`AnimalVisitor`现在只依赖`Animal`这个抽象类，符合`依赖倒转原则`。

有人可能担心，如果访问者抽象类没有把访问动物类的相应方法都列出来，会导致具体访问者类漏实现一些方法重载。这个问题在上面的代码中考虑到了，在`singledispatchmethod`装饰的`visit`方法里使用`NotImplementedError`异常进行防御，如果某个具体动物类没有重载`visit`方法，将抛出异常。