---
title: Scrapy 爬虫完全规则化的思考
date: 2025-01-11 21:31:48

categories:
  - 爬虫
  - Python
tags:
  - Python
  - Scrapy

summary: "根据《Python3网络爬虫开发实战》 Scrapy 的规则化爬虫例子，在其基础上添加了 Item 子类和 ItemLoader 子类的规则化，实现了 Scrapy 爬虫的完全规则化。"
---

看了《Python3网络爬虫开发实战（第2版）》，书中15章在讲到`Scrapy`框架时，15.12节谈到了规则化爬虫。

作者提到的规则化思路如下：
> 如果我们可以保留各个站点的`Spider`的公共部分,提取不同的部分进行单独配置(如将爬取规则页面解析方式等抽离出来，做成一个配置文件)，那么我们在新增一个爬虫的时候，只需要实现这些网站的爬取规则和提取规则，而且还可以单独管理和维护这些规则。

书中尝试着实现了规则化爬虫，最后把`Spider`的“设置“、“起止链接“、“`Item`提取方法“等抽取到`json`文件里，实现了可配置化。

然而书中的实现有些不完美的是，没有做到完全的可配置化，或者说没有做到完全的规则化爬虫。比如说要爬取新的站点时，除了需要创建`json`配置文件之外，还需要创建相应的`Item`子类和`ItemLoader`子类，也就说还是需要修改源码。

那么，有没有可能完全实现可配置化呢？想起`Python`的元编程，最简单的实现，貌似可以通过`type`创建类的方式来创建`Item`子类和`ItemLoader`子类？

### 使用type创建类

讲完全规则化的具体实现之前，先来了解一下使用`type`创建类的知识。

学过`Python`的人一般都知道，在`Python`中，可以使用内置的`type`来获取对象的类型。然而一般人不了解的是，`type`是一个元类，它不仅可以用于检查对象类型，还可以用来创建类。

`type`创建类的语法：`type(name, bases, dict)`。
- `name`是类的名称（字符串）；
- `bases`是基类的元组，可以是单个类或多个类（支持多继承）；
- `dict`是包含类属性和方法的字典。

使用type创建类的示例：

```python
Point = type("Point", (), {"x": 1, "y": 2})  
p = Point()  
print(p.x, p.y)
```

### Item子类的规则化

`Item`子类要实现规则化，那么就要根据字符串类名来生成相应`Item`子类。其实`Item`子类的实现挺简单的，实际上只要一行代码。当然了，虽然只有一行代码，最好还是把它封装起来，隐藏具体的实现。

可以用一个函数来封装（如果你喜欢的话，也可以用一个类来封装）：

```python
def item_class_factory(class_name: str, attrs: Iterable[str]):
    """
    Item 类工厂函数
    :param class_name: 类名
    :param attrs: 属性列表
    :return: 
    """
    return type(class_name,
                (Item,),
                {n: Field() for n in attrs})
```

使用的时候，只要传入类名和属性列表就可以了。这两者可以配置到配置文件里，使用的时候可以从配置文件里取出来，因为书中里已经将两者提取到配置文件里，这里便不再举出示例。

`item_class_factory`类的调用代码如下：

```python
item_cls = item_class_factory(cls_name=item_conf["class"], 
                              attrs=item_conf["attrs"].keys())
item = item_cls()
```

### ItemLoader子类的规则化

`ItemLoader`子类的规则化实现逻辑稍微有点复杂。书中并没有提取`ItemLoader`相关信息到配置文件，下面先给出相应的配置文件信息。
#### ItemLoader类配置

```json
{
	...
	"loader": {
	    "class": "MovieItemLoader",
	    "attrs": {
	      "default_output_processor": ["TakeFirst"],
	      "categories_out": ["Identity"],
	      "score_out": ["Compose",["TakeFirst"], ["Strip"]],
	      "drama_out": ["Compose", ["TakeFirst"], ["Strip"]]
	    }
	}
}
```

在配置文件里配置`ItemLoader`的类名和属性信息。

每一个属性的键值对代表`ItemLoader`子类的属性和`itemloaders.processors`的类型。类型是一个列表，列表的第一个值为类型名，后续值为参数（如果有参数的话）。

#### Processor

`Item`类，其属性的初始化都是`Field()`，所以不用另外设置。而`ItemLoader`类的属性不同，其初始化是`processors`的类，`processors`有不同的类，那么就要进行另外的设置，所以在配置文件里要填上相应的`processors`类名。

将配置文件里的`processors`类名转为类实例，自然也需要封装一个函数，其实现如下：

```python
def processor_fatory(proc_list: list[Any]):
    """
    Processor 实例工厂函数
    :param proc_list: Processor类型和参数列表
    :return:
    """

    # 第一个是类型，后续的是参数，因为参数有可能为空，所以这里不使用python的解包功能
    proc_name = proc_list[0]
    args = proc_list[1:]
    match proc_name:
        case x if x in ("Identity", "TakeFirst"):
            return getattr(processors, proc_name)()

        case "Join":
            if not args:
                return getattr(processors, proc_name)()
            else:
                return getattr(processors, proc_name)(*args)

        case "SelectJmes":
            return getattr(processors, proc_name)(*args)

        case x if x in ("Compose", "MapCompose"):
            # 使用递归进行组合
            compose = getattr(processors, proc_name)
            return compose(*(processor_fatory(a) for a in args))

        # 自定义的类型
        case "Strip":
            return getattr(processors, proc_name)()

        case _:
            raise TypeError(f"不支持的Processor：{proc_name}")
```

`Processor` 工厂函数的实现有几点值得注意的，

- 这不是创建类而是创建实例，故返回`Processor`时需要实例化。
- 对于需要自定义的`processor`，最好仿照`processors`模块里的类，封装成相应功能的`Processor`类，这样比较统一和规范。

上面代码中的`Strip`类就是自定义的`processor`，其实现如下：

```python
class Strip:
    """
    去除首尾空格，是对 str.strip 的包装
    """

    def __call__(self, value: str) -> str:
        return value.strip()
```

书中自定义的processor，没有进行封装，不够优雅。书中相关的`MovieItemLoader`类代码如下：

```python
class MovieItemLoader(ItemLoader):  
    default_output_processor = TakeFirst()  
    categories_out = Identity()  
    score_out = Compose(TakeFirst(), str.strip)  
    drama_out = Compose(TakeFirst(), str.strip)
```

代码中，直接将`str.strip`作为第二个参数传给`Compose`，而`Compose`的前一个参数是`TakeFirst()`。两者并不统一，封装了`Strip`类之后就可以这样传参：`Compose(TakeFirst(), Strip())`。这样就优雅多了。

#### ItemLoader子类的规则化

讲完了前面的，终于可以讲重点“ItemLoader子类的规则化”了。

有了`processor_fatory`函数的辅助，`ItemLoader`子类的规则化函数其实也挺简单的，其实现如下：

```python
def loader_class_factory(cls_name, attrs):
    """
    ItemLoder 类工厂函数
    :param cls_name: 类名
    :param attrs: 属性序列
    :return:
    """
    attr_dict = {key: processor_fatory(value)
                 for key, value in attrs.items()
                 }

    return type(cls_name,
                (ItemLoader,),
                attr_dict)
```

调用代码如下：

```python
loader_cls = loader_class_factory(loader_conf["class"], 
								  loader_conf["attrs"])
loader = loader_cls(item, selector=sel)
```
### 总结

至此，就可以说是实现了真正意义上的爬虫完全规则化了。这样要爬取新的网站时，只要添加新的站点json配置文件，完全不需要修改源码。

本文的源码存放在GitHub上，地址：[scrapy_universal_demo](https://github.com/linsuiyuan/scrapy_universal_demo)
