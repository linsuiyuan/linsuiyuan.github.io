---
title: "在 FastAPI 中自定义 Pydantic 的校验错误消息"
date: 2025-03-19T22:56:54+08:00

categories:
  - Web开发
  - Python
tags:
  - FastAPI
  - Pydantic
  - Python
  - 数据校验

summary: "本文探讨了在 FastAPI 中如何自定义 Pydantic 的校验错误消息。通过示例代码，展示了使用 @field_validator 装饰器自定义错误消息的方法，并介绍了如何通过自定义异常处理器来优化错误消息的显示，使其对用户更加友好。"
---
### 引言

FastAPI 是一个基于 Python 的高性能 Web 框架，专注于 API 开发，使用 Python 类型注解提供自动数据验证、序列化和高效的异步支持（基于 Starlette 和 Pydantic）。Pydantic 是一个数据验证和设置管理库，主要用于解析 JSON、校验数据类型。

Pydantic 添加校验条件是比较简单和自然的，然而如果要自定义错误消息则显得有些繁琐，下面来简单讨论一下。

### 依赖包说明

在开始讨论之前，先提一下执行本文代码所依赖的包。为方便计，安装依赖包使用的是傻瓜式的安装方式。

安装命令如下：

```
pip install "fastapi[all]"
```

通过上面的安装命令，会安装许多和 fastapi 相关的依赖包，和本文中直接相关的两个包分别是 fastapi 和 pydantic。其中 fastapi 的版本是 `fastapi==0.115.11`, pydantic 的版本是 `pydantic==2.10.6`

本文使用的 Python 版本是 `Python 3.9.6`

### Pydantic 自定义错误消息

Pydantic 为数据添加校验最简单的方式是使用 Field 给模型类的字段添加校验条件，如下所示：

```python
from pydantic import BaseModel, Field  
  
class User(BaseModel):  
    username: str = Field(..., min_length=3)  
    age: int = Field(..., ge=0)
```

上面的代码看上去和许多库添加数据校验条件的方式差不多，不过这个 Field 有些迷惑性，它实际上是一个工厂函数，不像其他库那样是描述符。Field 工厂函数返回的是 FieldInfo 实例，FieldInfo 实现了类似描述符的功能，能对字段进行校验。

当提供的值通不过 FieldInfo 中定义的校验条件时，Pydantic 会抛出异常并给出错误消息。在上面的代码后面添加以下代码：

```python
try:  
    user = User(username="ab", age=16)  
except ValidationError as e:  
    print(e.json())
```

运行上面的代码将输出：

```json
[{"type":"string_too_short","loc":["username"],"msg":"String should have at least 3 characters","input":"ab","ctx":{"min_length":3},"url":"https://errors.pydantic.dev/2.10/v/string_too_short"}]
```

这些消息是 Pydantic 包装好的，用户无法自定义。也就是说，如果要自定义错误消息，就不能使用 Field 工厂函数。

那么可以使用什么办法呢？可以 **@field_validator** 装饰器给字段添加自定义错误消息，代码如下所示：

```python
from pydantic import BaseModel, field_validator, ValidationError  
  
  
class User(BaseModel):  
    username: str  
    age: int  
  
    @field_validator("username")  
    def validate_username(cls, value):  
        if len(value) < 3:  
            raise ValueError("用户名长度必须大于等于 3 个字符")  
        return value  
  
    @field_validator("age")  
    def validate_age(cls, value):  
        if value < 0:  
            raise ValueError("年龄必须大于或等于 0 岁")  
        return value  
  
try:  
    user = User(username="ab", age=16)  
except ValidationError as e:  
    print(e.json())
```

运行上面的代码，将输出：

```json
[{"type":"value_error","loc":["username"],"msg":"Value error, 用户名长度必须大于等于 3 个字符","input":"ab","ctx":{"error":"用户名长度必须大于等于 3 个字符"},"url":"https://errors.pydantic.dev/2.10/v/value_error"}]
```

可以看出，**msg** 字段对应的值已是用户自定义的消息了。

**注意**要将之前在字段上添加的 Field 删除，否则输出的仍然是 Pydantic 封装的错误消息。

### 在 FastAPI 中自定义校验错误消息

一般在 FastAPI 中用来添加数据校验的库就是 Pydantic，所以可以直接将上面的代码直接集成到 FastAPI 中。

集成后的代码示例如下：

```python
from fastapi import FastAPI  
from pydantic import BaseModel, field_validator  
  
app = FastAPI()  
  
  
class User(BaseModel):  
    username: str  
    age: int  
  
    @field_validator("username")  
    def validate_username(cls, value):  
        if len(value) < 3:  
            raise ValueError("用户名长度必须大于等于 3 个字符")  
        return value  
  
    @field_validator("age")  
    def validate_age(cls, value):  
        if value < 0:  
            raise ValueError("年龄必须大于或等于 0 岁")  
        return value  
  
  
@app.post("/users/")  
async def create_user(user: User):  
    return {"message": "用户创建成功", "user": user}  
  
  
if __name__ == '__main__':  
    import uvicorn  
  
    uvicorn.run("main:app")
```

运行上面的代码，使用 API 接口调试工具，发送 post 请求，返回结果如下：

```json
{
    "detail": [
        {
            "type": "value_error",
            "loc": [
                "body",
                "username"
            ],
            "msg": "Value error, 用户名长度必须大于等于 3 个字符",
            "input": "ab",
            "ctx": {
                "error": {}
            }
        }
    ]
}
```

可以看出，虽然上面的结果包含了 Pydantic 自定义的错误消息，但是也包含了许多不必要的内容，对用户不够友好。

为了显示更友好的消息提示，可以在 User 模型类下面的位置添加一个自定义的异常处理器，如下所示：

```python
# 其他代码...

@app.exception_handler(RequestValidationError)  
async def validation_exception_handler(request: Request, exc: RequestValidationError):  
    errors = exc.errors()  
    custom_errors = [  
        {"field": error["loc"][-1], "message": error["msg"]}  
        for error in errors  
    ]  
    return JSONResponse(status_code=422, content={"errors": custom_errors})

  
@app.post("/users/")
# 其他代码....
```

重新运行代码，发送 post 请求，返回结果如下所示：

```json
{
    "errors": [
        {
            "field": "username",
            "message": "Value error, 用户名长度必须大于等于 3 个字符"
        }
    ]
}
```

可以看出，现在的结果就比较简单明了了，只包含了必要的消息，校验出错的字段以及对应的错误消息。

### 总结

综上，可以看出，在 FastAPI 中自定义 Pydantic 的校验错误消息并不难，只是有些繁琐。Pydantic 是一个很强大的库，希望在自定义错误消息方面能做一些改进，以方便用户使用。