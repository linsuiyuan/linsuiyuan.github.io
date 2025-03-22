---
title: "使用 Pytest 编写 SQLAlchemy 异步单元测试"
date: 2025-03-22

categories:
  - 测试
  - Python
tags:
  - Pytest
  - SQLAlchemy
  - 单元测试
  - 异步测试

summary: "本文介绍了如何使用 Pytest 编写 SQLAlchemy 的异步单元测试。首先，介绍了 Pytest 和 SQLAlchemy 的相关依赖包的安装。然后，展示了模型类和 CRUD 类的代码示例。接着，说明了如何进行 pytest 的配置。最后，给出了测试代码的示例。"
---

### 1、引言

Pytest 是 Python 中主流的测试框架，以简洁灵活著称。它支持通过直观的 assert 断言、可复用的 ​fixture 管理测试资源，以及参数化测试减少重复代码。

SQLAlchemy 是 Python 中一个功能强大的开源 ORM（对象关系映射）工具包，提供灵活的高层抽象与底层 SQL 控制。它将数据库表映射为 Python 类，支持通过面向对象方式操作数据库，同时允许直接编写原生 SQL，兼容多种数据库（如 PostgreSQL、MySQL、SQLite 等），兼顾开发效率与性能优化。

SQLAlchemy 支持同步操作，也支持异步操作，本文是讲如何使用 Pytest 来编写 SQLAlchemy 的异步单元测试。

### 2、安装依赖

**1、安装 Pytest 相关依赖包**

关于 Pytest 的相关依赖包，有两个，一个是 `pytest` 包，一个是 `pytest-asyncio`。

pytest-asyncio 是一个用于简化异步代码测试的 pytest 插件，它允许开发者直接编写和运行基于 asyncio 的异步测试用例。

```shell
pip install pytest==8.3.5
pip install pytest-asyncio==0.25.3
```

**2、安装 SQLAlchemy 相关依赖包**

关于 SQLAlchemy 的相关依赖包也有两个，一个是 `sqlalchemy`，另一个是 `greenlet`。

greenlet 是轻量级协程库，SQLAlchemy 的核心功能默认不依赖 greenlet 包，但在使用其异步扩展（如 asyncio 支持）时，需要 greenlet 作为底层协程切换工具。

```shell
pip install sqlalchemy==2.0.39
pip install greenlet==3.1.1
```

**3、安装数据库驱动包**

本项目采用 SQLite 作为数据库，Python 自带了 SQLite 的数据库驱动包 sqlite3，一般情况下无需额外安装依赖包。不过因为本项目是通过异步的方式操作数据库，所以需要额外安装一个异步库 `aiosqlite`。

```shell
pip install aiosqlite==0.21.0
```

### 3、模型类和 CRUD 类代码

为了编写测试代码，需要准备一个模型类和一个 CRUD 类。本项目采用 User 模型类及 UserService 类来作为示例。

User 模型类的代码比较简单，里面只有 id、username、password 三个属性。因为模型类不涉及方法操作，所以无需考虑适配异步操作的问题。User 类在同步和异步操作中都是可以使用的。

User 类位于 `models.py` 文件中，具体代码如下：

```python
from sqlalchemy import Column, String, Integer  
from sqlalchemy.orm import declarative_base  
  
Base = declarative_base()  
  
  
class User(Base):  
    __tablename__ = "user"  
  
    id = Column(Integer, primary_key=True, autoincrement=True)  
    username = Column(String(20), nullable=False, unique=True)  
    password = Column(String(32), nullable=False)
```

UserService 类除了一个异步会话属性 async_session，只有增、删、改、查四个方法，是一个简单的 CRUD 类。因为是异步操作，所以四个方法都使用 async 定义成协程函数。

UserService 类位于 `services.py` 文件中，具体代码如下：

```python
from sqlalchemy import select, update, delete  
from sqlalchemy.ext.asyncio import AsyncSession  
  
from models import User  
  
  
class UserService:  
  
    def __init__(self, async_session: AsyncSession):  
        self.async_session = async_session  
  
    async def get_user(self, user_id: int):  
        query = select(User).where(User.id == user_id)  
        result = await self.async_session.execute(query)  
        return result.scalars().first()  
  
    async def create_user(self, **kwargs):  
        user = User(**kwargs)  
        self.async_session.add(user)  
        await self.async_session.commit()  
        await self.async_session.refresh(user)  
        return user  
  
    async def update_user(self, user_id: int, **kwargs):  
        query = update(User).where(User.id == user_id)  
        result = await self.async_session.execute(query.values(**kwargs))  
        await self.async_session.commit()  
        return result.rowcount  
  
    async def delete_user(self, user_id):  
        query = delete(User).where(User.id == user_id)  
        result = await self.async_session.execute(query)  
        await self.async_session.commit()  
        return result.rowcount
```


### 4、配置文件设置

为了 pytest 能优雅地使用异步测试，可以在配置文件 `pytest.ini` 添加一些设置。内容如下：

```ini
[pytest]  
asyncio_default_fixture_loop_scope = function
asyncio_mode = auto
```

`asyncio_default_fixture_loop_scope` 是 pytest-asyncio 插件中的一个配置选项，它的作用是控制默认的异步事件循环 fixture 的作用域。其实这个配置选项有一个默认值，是 `function`，按道理有默认值是可以不用设置的。不过不设置的话，会有 `PytestDeprecationWarning` 警告输出，比较烦人，所以建议设置。

在 `asyncio_mode` 也是 pytest-asyncio 插件中的一个配置选项，它主要作用是控制 pytest-asyncio 如何处理异步测试函数和 fixtures。

asyncio_mode 的默认模式是 `strict`，这是最严格的模式。在该模式下，所有异步测试函数都必须使用 `@pytest.mark.asyncio` 标记，否则 pytest 会忽略异步测试函数并发出警告。

上面的配置文件里设置 asyncio_mode 为 `auto`，在该模式下，pytest-asyncio 会自动检测测试函数是否为异步函数，并自动应用相应的处理。这样，就不需要显式地使用 @pytest.mark.asyncio 标记。

### 5、测试代码

说点题外话，因为测试的数据库是 SQLite，而 SQLite 可以是文件数据库，也可以是内存数据库。后者非常适合用于测试，速度比较快，又不用考虑测试后的数据清理。故本项目采用内存SQLite数据库。

在编写正式的测试代码之前，可以先定义几个 fixture，以减少重复代码，同时保持代码结构清晰。这几个 fixture 分别是
`async_engine`, `async_session`, `user_service`, `test_user`。async_engine 是异步数据库引擎，用于创建 async_session。async_session 是异步会话类，用于在 asyncio 环境中执行数据库操作。user_service 是 UserService 的实例，便于后面的测试代码调用。test_user 是测试用户，事先插入数据库，便于需要时使用。

**需要注意的一点是**，这几个 fixture 建议使用 pytest-asyncio 插件提供的 `@pytest_asyncio.fixture` 来装饰。如果用 `@pytest.fixture` 来装饰，要确保前面的配置文件 asyncio_mode 设置为 auto 模式。

除了这几个 fixture，剩下的就是正式的测试代码了，没什么需要注意的，这里就不再多提。

测试代码位于 `test_services.py` 文件中，内容如下：

```python
import pytest  
import pytest_asyncio  
from sqlalchemy.exc import IntegrityError  
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession  
from sqlalchemy.orm import sessionmaker  
  
from models import Base, User  
from services import UserService  
  
# 使用内存SQLite数据库  
TEST_SQLALCHEMY_DATABASE_URL = "sqlite+aiosqlite:///:memory:"  
  
  
@pytest_asyncio.fixture  
async def async_engine():  
    # 创建异步引擎  
    engine = create_async_engine(  
        TEST_SQLALCHEMY_DATABASE_URL,  
        echo=False,  
    )  
  
    # 创建所有表  
    async with engine.begin() as conn:  
        await conn.run_sync(Base.metadata.create_all)  
    yield engine  
  
  
@pytest_asyncio.fixture  
async def async_session(async_engine):  
    # 创建异步会话工厂  
    async_session_local = sessionmaker(async_engine,  # noqa  
                                       expire_on_commit=False,  
                                       class_=AsyncSession)  
  
    # 创建新会话  
    async with async_session_local() as session:  
        yield session  
  
  
@pytest_asyncio.fixture  
async def user_service(async_session):  
    yield UserService(async_session)  
  
  
@pytest_asyncio.fixture  
async def test_user(async_session):  
    user = User(username="test_user", password="123456")  
    async_session.add(user)  
    await async_session.commit()  
    yield user  
  
  
async def test_get_user(user_service, test_user):  
    # 测试获取存在的用户  
    user = await user_service.get_user(test_user.id)  
  
    assert user is not None  
    assert user.id == test_user.id  
    assert user.username == test_user.username  
  
  
async def test_get_user_nonexistent(user_service):  
    # 测试获取不存在的用户  
    user = await user_service.get_user(10 ** 10)  
  
    assert user is None  
  
  
async def test_create_user(user_service):  
    # 测试创建新用户  
    username = "create_new_user"  
    new_user = await user_service.create_user(username=username, password="123456")  
  
    assert new_user is not None  
    assert new_user.username == username  
  
  
async def test_create_user_already_exists(user_service, test_user):  
    # 测试创建已存在的用户  
    with pytest.raises(IntegrityError, match="UNIQUE constraint failed"):  
        await user_service.create_user(username=test_user.username, password="123456")  
  
  
async def test_update_user(user_service, test_user):  
    # 测试更新用户信息  
    password = "update_password"  
    assert test_user.password != password  
  
    rowcount = await user_service.update_user(test_user.id, password=password)  
    assert rowcount == 1  
    assert test_user.password == password  
  
  
async def test_delete_user(user_service, test_user):  
    # 测试删除用户  
    uid = test_user.id  
    retrieve_user = await user_service.get_user(uid)  
    assert retrieve_user is not None  
  
    rowcount = await user_service.delete_user(uid)  
    assert rowcount == 1  
  
    retrieve_user = await user_service.get_user(uid)  
    assert retrieve_user is None
```

