---
title: "5种创建型设计模式笔记（Python实现）"
date: 2025-03-10T15:26:28+08:00

categories:
  - 设计模式
  - Python
tags:
  - Python
  - 单例模式
  - 工厂方法模式
  - 抽象工厂模式
  - 原型模式
  - 建造者模式

summary: "本文围绕设计模式中的 5 种创建型设计模式展开，详细介绍了单例模式、工厂方法模式、抽象工厂模式、原型模式和建造者模式，涵盖各模式的定义、优缺点、适用场景，并给出相应的 Python 代码示例，可作为学习和复习的笔记参考。"
---


## 引言

本文主要讲设计模式中的5种创建型设计模式，代码采用 Python 实现，以加深对设计模式和 Python 的理解，同时也可以作为笔记以便以后查看。代码实现主要参考张伟老师写的《设计模式的艺术》。

## 单例模式

**定义：**

单例模式是一种创建型设计模式，它确保一个类只有一个实例，并提供一个全局访问点来访问该实例。

**优点：**

- **节省资源：** 由于只有一个实例，可以减少内存占用，特别是在创建实例开销较大的情况下。
- **全局访问：** 提供了一个全局唯一的访问点，方便在程序的任何地方使用。
- **控制共享资源：** 可以有效地控制对共享资源的并发访问。

**缺点：**

- **违背单一职责原则：** 单例类既负责创建自身实例，又负责提供业务功能。
- **难以测试：** 由于全局唯一性，可能导致测试时的依赖关系复杂化。
- **并发问题：** 在多线程环境下，需要考虑线程安全问题。
- **可扩展性差：** 不利于扩展，如果需要修改单例类的行为，可能会影响到所有使用它的地方。

**适用场景：**

- 需要频繁创建和销毁的对象，如线程池、数据库连接池。
- 系统中只需要一个实例的类，如配置管理器、日志记录器。
- 需要控制对共享资源的访问，如计数器、序列生成器。

**代码实现：**

Python 有多种实现单例的方式，这里采用类属性创建单例的方式。在类中定义一个类属性来存储实例，并在 `__new__` 方法中控制实例的创建，同时使用线程锁来保证线程安全，代码如下：

```python
import threading  
  
  
class Singleton:  
    _instance = None  
    _lock = threading.Lock()  
  
    def __new__(cls, *args, **kwargs):  
        if not cls._instance:  
            with cls._lock:  
                if not cls._instance:  
                    cls._instance = super(Singleton, cls).__new__(cls)  
        return cls._instance  
  
  
singleton1 = Singleton()  
singleton2 = Singleton()  
assert singleton1 is singleton2
```

## 工厂方法模式

**定义：**

工厂方法模式是一种创建型设计模式，它定义了一个创建对象的接口，但将实际创建哪个类的决定延迟到子类中。换句话说，父类定义了创建对象的通用方法，而子类决定要创建哪个具体类的实例。

**优点：**

- **符合开闭原则：** 当需要增加新的产品时，只需要增加相应的具体工厂和具体产品，而不需要修改已有的代码。
- **解耦：** 将客户端代码与具体产品类解耦，客户端只需要知道抽象产品和抽象工厂，而不需要知道具体产品的实现细节。
- **提高代码的可扩展性：** 使得系统更容易扩展，可以方便地添加新的产品和工厂。

**缺点：**

- **增加类的数量：** 每增加一个产品，就需要增加一个具体产品类和一个具体工厂类，这会增加系统的复杂性。
- **增加了系统的抽象性和实现的难度：** 需要理解抽象工厂、抽象产品、具体工厂和具体产品之间的关系。

**适用场景：**

- 当一个类不知道它所必须创建的对象的类的时候。
- 当一个类希望由它的子类来指定它所创建的对象的时候。
- 当需要将对象的创建与使用分离时。
- 当需要提供一个产品类的库，而只想显示它们的接口而不是实现时。

**代码实现：**

这里使用日志记录器的例子来说明工厂方法模式。

Logger 接口充当抽象产品，其子类 FileLogger 和 DatabaseLogger 充当具体产品。LoggerFactory 接口充当抽象工厂，其子类 FileLoggerFactory 和 DatabaseLoggerFactory 充当具体工厂。

完整代码如下：

```python
from abc import ABC, abstractmethod  
  
  
# 日志记录器接口：抽象产品  
class Logger(ABC):  
    @abstractmethod  
    def log(self):  
        pass  
  
  
# 日志记录器工厂接口：抽象工厂  
class LoggerFactory(ABC):  
    @abstractmethod  
    def create_logger(self) -> Logger:  
        pass  
  
  
# 数据库日志记录器：具体产品  
class DatabaseLogger(Logger):  
    def log(self):  
        print("数据库日志记录。")  
  
  
# 数据库日志记录器工厂类：具体工厂  
class DatabaseLoggerFactory(LoggerFactory):  
    def create_logger(self) -> Logger:  
        # 连接数据库，代码省略  
        dblogger = DatabaseLogger()  
        # 初始化日志记录器  
        ...  
        return dblogger  
  
  
# 文件日志记录器：具体产品  
class FileLogger(Logger):  
    def log(self):  
        print("文件日志记录。")  
  
  
# 文件日志记录器工厂类：具体工厂  
class FileLoggerFactory(LoggerFactory):  
    def create_logger(self) -> Logger:  
        flogger = FileLogger()  
        # 初始化日志记录器  
        ...  
        return flogger  
  
  
if __name__ == "__main__":  
    factory = FileLoggerFactory()  
    logger = factory.create_logger()  
    logger.log()
```

## 抽象工厂模式

**定义：**

抽象工厂模式是一种创建型设计模式，它提供了一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。简单来说，抽象工厂模式可以创建一系列产品族，每个产品族包含多个相关产品。

**优点：**

- **隔离具体类：** 客户端代码与具体产品类解耦，只需与抽象工厂和抽象产品交互。
- **易于替换产品族：** 可以轻松替换整个产品族，而无需修改客户端代码。
- **保证产品一致性：** 确保同一产品族的产品一起使用。

**缺点：**

- **增加系统复杂性：** 需要定义多个抽象类和具体类，增加了系统的复杂性。
- **难以支持新种类的产品：** 如果需要添加新的产品种类，可能需要修改抽象工厂接口，影响所有具体工厂。

**适用场景：**

- 当系统需要独立于其产品的创建、组合和表示时。
- 当系统需要支持多个产品族时。
- 当需要保证同一产品族的产品一起使用时。
- 当需要在运行时切换产品族时。

**代码实现：**

这里使用界面皮肤库的例子来实现抽象工厂模式。

抽象类 SkinFactory 充当抽象工厂，其子类 SpringSkinFactory 和 SummerSkinFactory 充当具体工厂。抽象类 Button、TextField 和 ComboBox 充当抽象产品，其子类 SpringButton、SpringTextField 和 SummerButton、SummerTextField 充当具体产品。

完整代码如下:

```python
from abc import ABC, abstractmethod  
  
class Button(ABC):  
    @abstractmethod  
    def display(self):  
        pass  
  
class SpringButton(Button):  
    def display(self):  
        print("显示浅绿色按钮。")  
  
class SummerButton(Button):  
    def display(self):  
        print("显示浅蓝色按钮。")  
  
class TextField(ABC):  
    @abstractmethod  
    def display(self):  
        pass  
  
class SpringTextField(TextField):  
    def display(self):  
        print("显示绿色边框文本框。")  
  
class SummerTextField(TextField):  
    def display(self):  
        print("显示蓝色边框文本框。")  
  
class SkinFactory(ABC):  
    @abstractmethod  
    def create_button(self) -> Button:  
        pass  
  
    @abstractmethod  
    def create_text_field(self) -> TextField:  
        pass  
  
class SpringSkinFactory(SkinFactory):  
    def create_button(self) -> Button:  
        return SpringButton()  
  
    def create_text_field(self) -> TextField:  
        return SpringTextField()  
  
class SummerSkingFactory(SkinFactory):  
    def create_button(self) -> Button:  
        return SummerButton()  
  
    def create_text_field(self) -> TextField:  
        return SummerTextField()  
  
if __name__ == "__main__":  
    factory = SpringSkinFactory()  
    bt = factory.create_button()  
    tf = factory.create_text_field()  
  
    bt.display()  
    tf.display()
```

## 原型模式

**定义：**

原型模式是一种创建型设计模式，它通过复制现有对象（原型）来创建新对象，而无需知道其具体类。简单来说，就是“克隆”对象。

**优点：**

- **性能提升：** 对于创建复杂或耗时的对象，克隆现有实例比重新创建更高效。
- **简化创建：** 可以避免复杂的对象初始化过程。
- **动态创建：** 可以在运行时动态创建对象，无需预先知道其具体类型。
- **隐藏实现细节:**  客户端无需知道对象创建的具体细节。

**缺点：**

- **克隆复杂对象可能复杂：** 对于包含循环引用或复杂引用的对象，深拷贝可能比较困难。
- **需要实现克隆方法：** 每个需要克隆的类都需要实现克隆方法，这可能会增加代码量。

**适用场景：**

- **创建复杂对象开销较大时：** 例如，从数据库读取数据或进行复杂计算后创建的对象。
- **需要大量相似对象时：** 例如，游戏中的角色或图形编辑器中的形状。
- **需要隐藏对象创建细节时：** 客户端只需要知道如何克隆对象，而不需要知道其具体实现。
- **需要动态指定对象类型时：** 在运行时添加或删除对象。

**代码实现：**

Python中有个 copy 模块，可以直接用来实现原型模式。浅拷贝使用 copy.copy() 方法，深拷贝使用 copy.deepcopy() 方法，如果需要自定义拷贝逻辑，可以定义一个 clone() 方法。

因为比较简单，这里没有给出相关的代码例子。

## 建造者模式

**定义：**

建造者模式是一种创建型设计模式，它将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。简单来说，就是将复杂对象的创建过程分解为多个简单的步骤，并由一个建造者来负责这些步骤的执行。

**优点：**

- **分离构建与表示：** 使得同样的构建过程可以创建不同的表示，提高了代码的灵活性。
- **更好的控制构建过程：** 可以精细地控制对象的创建过程，避免了复杂的构造函数。
- **易于扩展：** 可以方便地添加新的建造者，从而创建新的对象表示。
- **代码清晰：** 将复杂对象的构建步骤分解，使得代码更易于理解和维护。

**缺点：**

- **增加类的数量：** 需要定义抽象建造者、具体建造者和指挥者等多个类，增加了系统的复杂性。
- **需要理解多个类之间的关系：** 需要理解抽象建造者、具体建造者，指挥者和产品之间的关系。

**适用场景：**

- 创建复杂对象，且对象的构建过程比较复杂时。
- 需要创建多种不同表示的对象，且这些对象具有相似的构建过程时。
- 需要隐藏对象的构建细节时。
- 当一个类中有很多的参数，而很多的参数都具有默认值，为了避免写过多的构造函数的时候。

**代码实现：**

这里使用游戏角色的创建来实现建造者模式。

ActorController 充当指挥者，ActorBuilder 充当抽象建造者，HeroBuilder、AngelBuilder 充当具体建造者，Actor 充当复杂产品。

```python
from abc import ABC, abstractmethod  
from dataclasses import dataclass  
  
@dataclass  
class Actor:  
    type: str = None  
    sex: str = None  
    costume: str = None  
  
class ActorBuilder(ABC):  
    actor: Actor = Actor()  
  
    @abstractmethod  
    def build_type(self): ...  
  
    @abstractmethod  
    def build_sex(self): ...  
  
    @abstractmethod  
    def build_costume(self): ...  
  
    def build(self):  
        return self.actor  
  
class HeroBuilder(ActorBuilder):  
    def build_type(self):  
        self.actor.type = "英雄"  
  
    def build_sex(self):  
        self.actor.sex = "男"  
  
    def build_costume(self):  
        self.actor.costume = "盔甲"  
  
class AngelBuilder(ActorBuilder):  
    def build_type(self):  
        self.actor.type = "天使"  
  
    def build_sex(self):  
        self.actor.sex = "女"  
  
    def build_costume(self):  
        self.actor.costume = "白裙"  
  
class ActorController:  
    def construct(self, builder: ActorBuilder) -> Actor:  
        builder.build_type()  
        builder.build_sex()  
        builder.build_costume()  
        return builder.build()  
  
if __name__ == "__main__":  
    builder = HeroBuilder()  
    actor = ActorController().construct(builder)  
    print(actor)
```