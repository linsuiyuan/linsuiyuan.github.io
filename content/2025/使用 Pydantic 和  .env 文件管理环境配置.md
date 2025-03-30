---
title: "使用 Pydantic 和 .env 文件管理环境配置"
date: 2025-03-16T16:38:28+08:00

categories:
  - 开发技术
  - Python
tags:
  - Pydantic
  - 环境变量
  - 配置管理

summary: "本文介绍如何使用 Pydantic 库结合 .env 文件管理项目配置，实现环境变量的自动加载和类型验证，支持跨环境配置切换，避免硬编码敏感信息。"
---


Pydantic 是一个 Python 第三方包，用于数据验证和设置管理。它通过类型注解自动验证和解析数据，确保数据符合预期格式，并且能够生成清晰的错误信息。

这里主要介绍使用 Pydantic 和 .env 文件来管理环境配置。

### **1、安装依赖库**

Pydantic 第一个版本时，安装 Pydantic 库就可以了，而第二个版本之后，settings 相关的功能被拆分到独立的包了。独立出来的包叫 `pydantic-settings` ，目前最新版本为 2.8.1。

现在只需要安装 pydantic-settings 包，安装命令如下：

```sh
pip install pydantic-settings==2.8.1
```

安装 pydantic-settings 包时，会自动安装其他 pydantic、python-dotenv 等依赖包。
### **2、创建 .env 文件**

创建一个 .env 文件，根据具体项目需要填入相应的内容。例如：

```ini
DATABASE_URL=your_database_url
DATABASE_USERNAME=your_username
DATABASE_PASSWORD=your_password
```

### **3、创建 config.py 文件**

创建一个 config.py 文件，在里面创建一个 Settings 类，该类继承自 pydantic_settings 的 BaseSettings 类。Settings 类添加和 .env 文件里的环境变量相同的类属性，并创建一个 model_config 类属性，用 SettingsConfigDict 初始化该类属性 。

代码如下：

```python
from pydantic_settings import BaseSettings, SettingsConfigDict  
  
  
class Settings(BaseSettings):  
    DATABASE_URL: str  
    DATABASE_USERNAME: str  
    DATABASE_PASSWORD: str  
  
    model_config = SettingsConfigDict(  
        env_file=".env"  
    )
```

为了方便获取配置，还可以在 config.py 文件里添加一个 `get_settings()` 函数，如下所示：

```python
def get_settings():  
    return Settings()
```

测试代码如下：

```python
if __name__ == '__main__':  
    settings = get_settings()  
    assert settings.DATABASE_URL == "your_database_url"  
    assert settings.DATABASE_USERNAME == "your_username"  
    assert settings.DATABASE_PASSWORD == "your_password"
```

### **4、为环境变量添加前缀以区别不同的环境**

使用 .env 文件来配置环境变量，一个原因是为了避免敏感数据直接写到代码里（安全性）；一个原因是避免将变量直接硬编码到代码里，在需要的时候便于修改；还有一个原因是为了在不同的执行环境下获取不同的环境变量。

在上面的代码里，每个环境变量只设置了一个值。那么如何给每个变量设置不同的值呢？

可以给环境变量添加前缀来区分不同的执行环境。比如生产环境使用 `PROD_` 前缀，开发环境使用 `DEV_` 前缀。

初始化 model_config 类属性时，SettingsConfigDict 类添加 `env_prefix` 参数，值为 `os.getenv("ENVIRONMENT_PREFIX", "DEV_")` 。表示会先从环境变量 `ENVIRONMENT_PREFIX` 读取前缀，如果读取不到，则默认使用 `DEV_` 前缀。

同时 SettingsConfigDict 类添加 extra 参数，其值为 "allow"。因为 Pydantic 默认会忽略未在模型中定义的字段（即 `extra="ignore"`），这些字段不会被校验或存储。当设置为 `extra="ignore"` 后，模型会 ​**接受并保留** 未定义的字段，将它们作为动态属性存储，但不会进行类型校验。

**注意：添加了  `env_prefix` 属性之后，要记得根据需要设置一下 `ENVIRONMENT_PREFIX` 这个环境变量。**

代码如下：

```python
	# 其他代码...
	model_config = SettingsConfigDict(  
	    env_prefix=os.getenv("ENVIRONMENT_PREFIX", "DEV_"),  
	    env_file=".env",  
	    extra="allow"  
	)
```

### **5、最后的完整代码**

.env 文件的完整代码示例：

```ini
DEV_DATABASE_URL=your_database_url  
DEV_DATABASE_USERNAME=your_username  
DEV_DATABASE_PASSWORD=your_password  
  
PROD_DATABASE_URL=prod_your_database_url  
PROD_DATABASE_USERNAME=prod_your_username  
PROD_DATABASE_PASSWORD=prod_your_password
```

config.py 的完整代码示例：

```python
import os  
  
from pydantic_settings import BaseSettings, SettingsConfigDict  
  
  
class Settings(BaseSettings):  
    DATABASE_URL: str  
    DATABASE_USERNAME: str  
    DATABASE_PASSWORD: str  
  
    model_config = SettingsConfigDict(  
        env_prefix=os.getenv("ENVIRONMENT_PREFIX", "DEV_"),  
        env_file=".env",  
        extra="allow"  
    )  
  
  
def get_settings():  
    return Settings()  
  
  
if __name__ == '__main__':  
    settings = get_settings()  
    prefix = os.getenv("ENVIRONMENT_PREFIX", "DEV_")  
    if prefix == "DEV_":  
        assert settings.DATABASE_URL == "your_database_url"  
        assert settings.DATABASE_USERNAME == "your_username"  
        assert settings.DATABASE_PASSWORD == "your_password"  
    elif prefix == "PROD_":  
        assert settings.DATABASE_URL == "prod_your_database_url"  
        assert settings.DATABASE_USERNAME == "prod_your_username"  
        assert settings.DATABASE_PASSWORD == "prod_your_password"
```