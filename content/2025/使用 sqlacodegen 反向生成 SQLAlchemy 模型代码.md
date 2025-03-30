---
title: "使用 sqlacodegen 反向生成 SQLAlchemy 模型代码"
date: 2025-03-30T16:58:52+08:00

categories:
  - 开发工具
  - 数据库
  - Python
tags:
  - SQLAlchemy
  - 数据库
  - Python
  - 开发工具

summary: "sqlacodegen 是一个强大的工具，它可以读取现有数据库结构，并自动生成相应的 SQLAlchemy 模型代码。本文介绍了 sqlacodegen 的安装、基本用法、通用选项以及各种生成器的介绍和选项。"
---


## 介绍

sqlacodegen 是一款能够读取现有数据库结构并生成相应的 SQLAlchemy 模型代码的工具。目前支持 SQLAlchemy 2.x，能识别关系类型（包括多对多、一对一），能自动检测表继承等。




## 安装

sqlacodegen 的安装命令如下：
```sh
pip install sqlacodegen
```

截至目前， sqlacodegen 的版本是 3.0.0。



## 示例

使用 sqlacodegen 命令，最基本的是要传给它一个数据库 URL。

示例：

```shell
sqlacodegen postgresql:///some_local_db
sqlacodegen --generator tables mysql+pymysql://user:password@localhost/dbname
sqlacodegen --generator dataclasses sqlite:///database.db
```

下面使用一个简单的 user 表来演示 sqlacodegen 命令的使用。user 表的数据库是 MySQL，有三个字段，分别是 id、username 和 password。其中 id 是主键字段，username 和 password 是用户名和密码，且都不能为 null，同时 username 具有唯一索引。

user 表结构如下所示：

| **名**   | **类型** | **长度** | 不是 null | **键** | 注释   |
| -------- | -------- | -------- | --------- | ------ | ------ |
| id       | int      |          | ☑         | 🔑      | 主键   |
| username | varchar  | 20       | ☑         |        | 用户名 |
| password | varchar  | 32       | ☑         |        | 密码   |

使用 sqlacodegen 命令前，需要安装 pymysql 这个 MySQL 数据库驱动包，命令如下：

```shell
pip install pymysql
```

目前 pymysql 的版本是 1.1.1。

接下来就可以使用 sqlacodegen 命令反向生成 ORM 模型类了，命令如下：

```shell
sqlacodegen mysql+pymysql://user:password@localhost/test --outfile user.py
```

生成的 user.py 文件的代码如下：

```python
from sqlalchemy import Index, Integer
from sqlalchemy.dialects.mysql import VARCHAR
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = 'user'
    __table_args__ = (
        Index('username', 'username', unique=True),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, comment='主键')
    username: Mapped[str] = mapped_column(VARCHAR(20), comment='用户名')
    password: Mapped[str] = mapped_column(VARCHAR(32), comment='密码')
```



## 通用选项

使用 `sqlacodegen --help` 命令可以查看 sqlacodegen 命令的通用选项，其输出如下所示：

```shell
usage: sqlacodegen [-h] [--options OPTIONS] [--version] [--schemas SCHEMAS] [--generator {dataclasses,declarative,sqlmodels,tables}]
                   [--tables TABLES] [--noviews] [--outfile OUTFILE]
                   [url]

Generates SQLAlchemy model code from an existing database.

positional arguments:
  url                   SQLAlchemy url to the database

options:
  -h, --help            show this help message and exit
  --options OPTIONS     options (comma-delimited) passed to the generator class
  --version             print the version number and exit
  --schemas SCHEMAS     load tables from the given schemas (comma-delimited)
  --generator {dataclasses,declarative,sqlmodels,tables}
                        generator class to use
  --tables TABLES       tables to process (comma-delimited, default: all)
  --noviews             ignore views (always true for sqlmodels generator)
  --outfile OUTFILE     file to write output to (default: stdout)
```

相应的中文翻译如下所示：

```shell
用法: sqlacodegen [-h] [--options OPTIONS] [--version] [--schemas SCHEMAS] [--generator {dataclasses,declarative,sqlmodels,tables}]
                   [--tables TABLES] [--noviews] [--outfile OUTFILE]
                   [url]

从现有数据库生成 SQLAlchemy 模型代码。

位置参数:
  url                   数据库URL

选项:
  -h, --help            显示帮助信息
  --options OPTIONS     传递给生成器类的选项（逗号分隔）
  --version             输出版本号
  --schemas SCHEMAS     从指定模式加载表（逗号分隔）
  --generator {dataclasses,declarative,sqlmodels,tables}
                        使用的生成器
  --tables TABLES       要处理的表（逗号分隔，默认：全部）
  --noviews             忽略视图（sqlmodels 生成器始终忽略视图）
  --outfile OUTFILE     输出到文件（默认：标准输出）
```



## 生成器(generator)

### 生成器(generator)介绍

生成器是  sqlacodegen  的核心选项，用于指定代码生成逻辑的类型。不同的生成器会输出不同风格的 SQLAlchemy 模型代码，适用于不同的开发场景。通过 `--generator` 参数选择生成器。

sqlacodegen 内置的生成器包括:

- `tables` (仅生成 `Table` 对象)
- `declarative` (默认，生成继承自 `declarative_base()` 的类)
- `dataclasses` (生成基于数据类的模型，仅限 v1.4+)
- `sqlmodels` (生成 SQLModel 模型类)



### 生成器专属选项

通过 `--options` 指定(多个值用逗号分隔):

- tables

  - noconstraints: 忽略约束（外键、唯一约束等）

  - nocomments: 忽略注释

  - noindexes: 忽略索引


- declarative

  - 继承 tables 所有选项

  - use_inflect: 自动将复数表名转换为单数类名（如 `users` → `User`）。

  - nojoined: 禁用表继承检测

  - nobidi: 仅生成单向关系（不生成反向关系属性）。


- dataclasses
  - 继承 declarative 所有选项


- sqlmodels
  - 继承 declarative 所有选项