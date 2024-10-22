# 入门 Redis OM

## 简介

本教程将引导您安装 Redis OM，创建第一个模型，并使用它来保存和验证数据。

## 先决条件

Redis OM 需要 Python 版本 3.8 或更高版本以及一个 Redis 实例进行连接。

## Python

确保您正在运行 **Python 版本 3.8 或更高版本**：

```
python --version
Python 3.8.0
```

如果您尚未安装 Python，可以从 [Python.org](https://www.python.org/downloads/) 下载，使用 [pyenv](https://github.com/pyenv/pyenv)，或通过操作系统的包管理器安装 Python。

此库需要 [redis-py](https://pypi.org/project/redis) 版本 4.2.0 或更高版本。

## Redis

Redis OM 将数据保存在 Redis 中，因此您需要安装并运行 Redis 以完成本教程。

我们推荐使用 [redis-stack](https://hub.docker.com/r/redis/redis-stack) 镜像，因为它包含此库用来提供额外功能的 Redis 功能。指南后面的部分将提供有关这些功能的更多细节。

您也可以使用官方的 Redis Docker 镜像，该镜像托管在 [Docker Hub](https://hub.docker.com/_/redis) 上。然而，这个镜像不包括存储 JSON 模型和使用 `find` 查询接口所需的搜索和 JSON 模块。

**注意**：当我们在本指南的后面讨论如何实际启动 Redis 时，将谈到使用 Docker 启动 Redis。

### 下载 Redis

Redis 的最新版本可从 [Redis.io](https://redis.io/) 获取。您也可以通过操作系统的包管理器安装 Redis。

**注意**：本教程将指导您如何在本地启动 Redis，但如果 Redis 在远程服务器上运行，这些说明也适用。

### 在 Windows 上安装 Redis

Redis 不能直接在 Windows 上运行，但您可以使用 Windows 子系统 Linux (WSL) 来运行 Redis。有关详细信息，请参阅 [我们在 YouTube 上的视频](https://youtu.be/_nFwPTHOMIY)。

Windows 用户还可以使用前面提到的 Docker 镜像。

## 推荐：RediSearch 和 RedisJSON

Redis OM 依赖 [RediSearch][redisearch-url] 和 [RedisJSON][redis-json-url] Redis 模块，以支持丰富的查询和嵌入模型。

您不需要这些 Redis 模块来使用 Redis OM 的数据建模、验证和持久化功能，但我们推荐您使用它们以充分发挥 Redis OM 的优势。

在本地开发期间运行这些 Redis 模块的最简单方法是使用 [redis-stack](https://hub.docker.com/r/redis/redis-stack) Docker 镜像。

有关其他安装方法，请遵循这两个模块主页上的“快速入门”指南。

## 启动 Redis

在开始使用 Redis OM 之前，请确保启动 Redis。

启动 Redis 的命令将取决于您如何安装它。

### Ubuntu Linux（包括 WSL）

如果您使用 `apt` 安装了 Redis，请使用 `systemctl` 命令启动它：

    $ sudo systemctl restart redis.service

否则，您可以手动启动服务器：

    $ redis-server start

### 使用 Homebrew 的 MacOS

    $ brew services start redis

### Docker

使用 Docker 启动 Redis 的命令取决于您选择使用的镜像。

**提示：** 这些示例中的 `-d` 选项使 Redis 在后台运行，而 `-p 6379:6379` 则使 Redis 可以通过本地主机的 6379 端口访问。

#### 使用 `redismod` 镜像的 Docker（推荐）

    $ docker run -d -p 6379:6379 redislabs/redismod

### 使用 `redis` 镜像的 Docker

    $ docker run -d -p 6379:6379 redis

## 安装 Redis OM

安装 Redis OM 的推荐方法是使用 [Poetry](https://python-poetry.org/docs/)。您可以使用以下命令通过 Poetry 安装 Redis OM：

    $ poetry add redis-om

如果您使用 Pipenv，则命令为：

    $ pipenv install redis-om

最后，您可以通过运行以下命令使用 `pip` 安装 Redis OM：

    $ pip install redis-om

**提示：** 如果您不使用 Poetry 或 Pipenv，而是直接使用 `pip` 安装，建议您在虚拟环境（即 virtualenv）中安装 Redis OM。如果您对这个概念不熟悉，请参阅 [Dan Bader 的视频和文字记录](https://realpython.com/lessons/creating-virtual-environment/)。

## 设置 Redis URL 环境变量

我们快要准备好创建 Redis OM 模型了！但首先，我们需要确保 Redis OM 知道如何连接到 Redis。

默认情况下，Redis OM 尝试连接到您本地主机的 6379 端口。大多数本地安装方法会导致 Redis 运行在这个位置，在这种情况下您无需做任何特别的事情。

然而，如果您将 Redis 配置为在不同的端口上运行，或者您正在使用远程 Redis 服务器，则需要设置 `REDIS_OM_URL` 环境变量。

`REDIS_OM_URL` 环境变量遵循 redis-py URL 格式：

    redis://[[username]:[password]]@localhost:6379/[database number]

默认连接相当于以下 `REDIS_OM_URL` 环境变量：

    redis://@localhost:6379

**提示：** Redis 数据库是编号的，默认值为 0。您可以省略数据库编号以使用默认数据库。

**注意：** 索引仅适用于存储在 Redis 逻辑数据库 0 中的数据。如果在连接到 Redis 时使用不同的数据库编号，您可以预期代码在运行迁移器时会引发 `MigrationError`。

其他支持的前缀包括 "rediss" 用于 SSL 连接和 "unix" 用于 Unix 域套接字：

    rediss://[[username]:[password]]@localhost:6379/0
    unix://[[username]:[password]]@/path/to/socket.sock?db=0

## 定义模型

在本教程中，我们将创建一个 `Customer` 模型，用于验证和保存数据。让我们从模型的基本定义开始。我们将逐步添加功能。

```python
import datetime

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: str
```

有几个细节需要注意：

1. 我们的 `Customer` 模型扩展了 `HashModel` 类。这意味着它将作为哈希保存到 Redis。Redis OM 提供的另一个模型类是 `JsonModel`，稍后我们会讨论。
2. 我们使用 Python 类型注解指定了模型的字段。

让我们更深入地了解一下 `HashModel` 类和类型注解。

### HashModel 类

当您对 `HashModel` 进行子类化时，您的子类既是 Redis OM 模型，具有将数据保存到 Redis 的方法，同时也是 Pydantic 模型。

这意味着您可以在 Redis OM 模型中使用 Pydantic 字段验证，我们将在后面讨论验证时涵盖这一点。但这也意味着您可以在任何需要使用 Pydantic 模型的地方使用 Redis OM 模型，比如在您的 FastAPI 应用程序中。🤯

### 类型注解

您添加到模型字段的类型注解用于几个目的：

* 使用 Pydantic 验证器验证数据
* 序列化数据到 Redis
* 从 Redis 反序列化数据

在本教程的过程中，我们将看到这些用途的示例。

关于 `HashModel` 类的一个重要细节是，它不支持 `list`、`set` 或映射（如 `dict`）类型。这是因为 Redis 哈希不能包含列表、集合或其他哈希。

如果您想建模带有列表、集合或映射类型的字段，或者其他模型，您需要使用 `JsonModel` 类，它可以支持这些类型以及嵌入模型。

## 创建模型

让我们看看创建模型对象的样子：

```python
import datetime

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: str


andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38,
    bio="Python developer, works at Redis, Inc."
)
```

### 可选字段

如果我们省略了其中一个字段，比如 `bio`，会发生什么呢？

```python
import datetime

from redis_om import HashModel
from pydantic import ValidationError


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: str


# 所有字段都是必需的，因为没有任何字段
# 被标记为 `Optional`，所以我们会得到一个验证错误：
try:
    Customer(
        first_name="Andrew",
        last_name="Brookins",
        email="andrew.brookins@example.com",
        join_date=datetime.date.today(),
        age=38  # <- 我们没有传入 bio！
    )
except ValidationError as e:
    print(e)
    """
    ValidationError: 1 validation error for Customer
    bio
      field required (type=value_error.missing)
    """
```

如果我们希望 `bio` 字段是可选的，我们需要将类型注释更改为使用 `Optional`。

```python
import datetime
from typing import Optional

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: Optional[str]  # <- 现在，bio 是一个 Optional[str]
```

现在我们可以创建有或没有 `bio` 字段的 `Customer` 对象。

### 默认值

字段可以有默认值。您可以通过给字段赋值来设置它们。

```python
import datetime
from typing import Optional

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: Optional[str] = "Super dope"  # <- 我们在这里添加了默认值
```

现在，如果我们创建一个没有 `bio` 字段的 `Customer` 对象，它将使用默认值。

```python
import datetime
from typing import Optional

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: Optional[str] = "Super dope"


andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38)  # <- 注意，我们没有提供 bio！

print(andrew.bio)  # <- 所以我们得到了默认值。
# > 'Super Dope'
```

模型将在您下次调用 `save()` 时将此默认值保存到 Redis。

### 自动主键

模型会自动生成一个全球唯一的主键，而无需与 Redis 进行通信。

```python
import datetime
from typing import Optional

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: Optional[str] = "Super dope"


andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38)

print(andrew.pk)
# > '01FJM6PH661HCNNRC884H6K30C'
```

ID 在您保存模型之前就可用。

默认的 ID 生成函数会创建 [ULIDs](https://github.com/ulid/spec)，不过如果您想使用不同类型的主键，可以更改生成模型主键的函数。

## 数据验证

Redis OM 使用 [Pydantic][pydantic-url] 根据您为模型类字段分配的类型注释来验证数据。

这种验证确保像 `first_name` 这样的字段在 `Customer` 模型中标记为 `str` 时始终为字符串。**但每个 Redis OM 模型也是 Pydantic 模型**，因此您可以使用 Pydantic 验证器，如 `EmailStr`、`Pattern` 等进行复杂验证！

例如，我们之前将 `Customer` 模型的 `join_date` 定义为 `datetime.date`。因此，如果我们尝试创建一个 `join_date` 不是日期的模型，我们将得到一个验证错误。

让我们现在尝试一下：

```python
import datetime
from typing import Optional

from redis_om import HashModel
from pydantic import ValidationError


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: Optional[str] = "Super dope"


try:
    Customer(
        first_name="Andrew",
        last_name="Brookins",
        email="a@example.com",
        join_date="not a date!",  # <- 问题所在行！
        age=38
    )
except ValidationError as e:
    print(e)
    """
    pydantic.error_wrappers.ValidationError: 1 validation error for Customer
    join_date
      invalid date format (type=value_error.date)
    """
```

### 模型默认强制转换值

您可能会想，在我们上一个验证示例中，什么算作“日期”。默认情况下，Redis OM 会尝试将输入值强制转换为正确的类型。这意味着我们可以将日期字符串传递给 `join_date`，而不必传递 `date` 对象：

```python
import datetime

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int


andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="a@example.com",
    join_date="2020-01-02",  # <- 我们现在传递的是 YYYY-MM-DD 日期字符串
    age=38
)

print(andrew.join_date)
# > 2021-11-02
type(andrew.join_date)
# > datetime.date  # 模型自动解析了字符串！
```

这种将解析（在此情况下为日期字符串）与验证相结合的能力可以为您节省很多工作。

然而，您可以关闭强制转换——请查看下一节关于使用严格验证的内容。

### 严格验证

您可以开启严格验证，以拒绝字段的值，除非它们与模型的类型注释完全匹配。

您可以通过将字段的类型注释更改为使用 Pydantic 提供的 ["严格" 类型](https://pydantic-docs.helpmanual.io/usage/types/#strict-types) 来实现这一点。

Redis OM 支持 Pydantic 的所有严格类型：`StrictStr`、`StrictBytes`、`StrictInt`、`StrictFloat` 和 `StrictBool`。

如果我们想确保 `age` 字段仅接受整数，并且不尝试解析包含整数的字符串，例如 "1"，我们将使用 `StrictInt` 类。

```python
import datetime
from typing import Optional

from pydantic import StrictInt, ValidationError
from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: StrictInt  # <- 这里使用 StrictInt 而不是 int
    bio: Optional[str]


# 现在如果我们对 `age` 使用字符串而不是整数，
# 我们将得到一个验证错误：
try:
    Customer(
        first_name="Andrew",
        last_name="Brookins",
        email="a@example.com",
        join_date="2020-01-02",
        age="38"  # <- 字符串类型的年龄现在不应该有效！
    )
except ValidationError as e:
    print(e)
    """
    pydantic.error_wrappers.ValidationError: 1 validation error for Customer
    age
      Value is not a valid integer (type=type_error.integer)
    """
```

Pydantic 不包含 `StrictDate` 类，但我们可以自己创建。在这个例子中，我们创建一个 `StrictDate` 类型，用于验证 `join_date` 是一个 `datetime.date` 对象。

```python
import datetime
from typing import Optional

from pydantic import ValidationError
from redis_om import HashModel


class StrictDate(datetime.date):
    @classmethod
    def __get_validators__(cls) -> 'CallableGenerator':
        yield cls.validate

    @classmethod
    def validate(cls, value: datetime.date, **kwargs) -> datetime.date:
        if not isinstance(value, datetime.date):
            raise ValueError("Value must be a datetime.date object")
        return value


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: StrictDate
    age: int
    bio: Optional[str]


# 现在如果我们对 `join_date` 使用字符串而不是日期对象，
# 我们将得到一个验证错误：
try:
    Customer(
        first_name="Andrew",
        last_name="Brookins",
        email="a@example.com",
        join_date="2020-01-02",  # <- 字符串现在不应该有效！
        age="38"
    )
except ValidationError as e:
    print(e)
    """
    pydantic.error_wrappers.ValidationError: 1 validation error for Customer
    join_date
      Value must be a datetime.date object (type=value_error)
    """
```

## 保存模型

我们可以通过调用 `save()` 将模型保存到 Redis：

```python
import datetime

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int


andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38)

andrew.save()
```

## 过期模型

我们可以使用 `expire` 方法让模型实例过期，并传入希望实例在 Redis 中过期的秒数：

```python
# 让 Andrew 在 2 分钟（120 秒）后过期
andrew.expire(120)
```

## 在 Redis 中检查数据

您可以查看存储在 Redis 中的任何 Redis OM 模型的数据。

首先，获取您想要检查的模型实例的键。`key()` 方法将为您提供用于存储模型的确切 Redis 键。

**注意：** 这个方法的命名可能会让人困惑。这不是主键，而是该模型的 Redis 键。因此，方法名称可能会有所更改。

在这个例子中，我们查看我们正在构建的 `Customer` 模型的键：

```python
import datetime
from typing import Optional

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: Optional[str] = "Super dope"


andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38)

andrew.save()
andrew.key()
# > 'mymodel.Customer:01FKGX1DFEV9Z2XKF59WQ6DC9T'
```

有了模型的 Redis 键，您可以启动 `redis-cli` 并检查该键下存储的数据。在这里，我们使用正在运行的 "redis" 容器运行 `JSON.GET` 命令，该容器由此项目的 Docker Compose 文件定义：

```
$ docker compose exec -T redis redis-cli HGETALL mymodel.Customer:01FKGX1DFEV9Z2XKF59WQ6DC9T

 1) "pk"
 2) "01FKGX1DFEV9Z2XKF59WQ6DC9T"
 3) "first_name"
 4) "Andrew"
 5) "last_name"
 6) "Brookins"
 7) "email"
 8) "andrew.brookins@example.com"
 9) "join_date"
10) "2021-11-02"
11) "age"
12) "38"
13) "bio"
14) "Super dope"
```

## 获取模型

如果您有模型的主键，可以在模型类上调用 `get()` 方法来获取模型的数据。

```python
import datetime
from typing import Optional

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: str
    join_date: datetime.date
    age: int
    bio: Optional[str] = "Super dope"


andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38)

andrew.save()

assert Customer.get(andrew.pk) == andrew
```

## 使用表达式查询模型

Redis OM 提供了一种丰富的查询语言，允许您使用 Python 表达式查询 Redis。

为了展示这一点，我们将对之前定义的 `Customer` 模型做一个小改动。我们将添加 `Field(index=True)`，以告诉 Redis OM 我们希望为 `last_name` 和 `age` 字段建立索引：

```python
import datetime
from typing import Optional

from pydantic import EmailStr

from redis_om import (
    Field,
    HashModel,
    Migrator
)


class Customer(HashModel):
    first_name: str
    last_name: str = Field(index=True)
    email: EmailStr
    join_date: datetime.date
    age: int = Field(index=True)
    bio: Optional[str]


# 现在，如果我们在安装了 RediSearch 模块的 Redis 部署中使用此模型，
# 我们可以运行如下查询。

# 在运行查询之前，我们需要运行迁移，以设置 Redis OM 将使用的索引。
# 您也可以使用 `migrate` CLI 工具来实现这一点！
Migrator().run()

# 查找所有姓 "Brookins" 的客户
Customer.find(Customer.last_name == "Brookins").all()

# 查找所有不姓 "Brookins" 的客户
Customer.find(Customer.last_name != "Brookins").all()

# 查找所有姓 "Brookins" 或者年龄为 100 且姓 "Smith" 的客户
Customer.find((Customer.last_name == "Brookins") | (
        Customer.age == 100
) & (Customer.last_name == "Smith")).all()
```

### 保存和查询布尔值

由于历史原因，`HashModels` 不支持保存和查询布尔值。然而，在 JSON 模型中，您可以使用 `==` 语法存储和查询布尔值：

```python
from redis_om import (
    Field,
    JsonModel,
    Migrator
)

class Demo(JsonModel):
    b: bool = Field(index=True)

Migrator().run()
d = Demo(b=True)
d.save()
res = Demo.find(Demo.b == True)
```

## 调用其他 Redis 命令

有时您需要直接运行 Redis 命令。Redis OM 通过模型类上的 `db` 方法支持这一点。它返回一个已连接的 Redis 客户端实例，该实例为每个 Redis 命令提供一个命名函数。例如，让我们执行一些基本的集合操作：

```python
from redis_om import HashModel

class Demo(HashModel):
    some_field: str

redis_conn = Demo.db()

redis_conn.sadd("myset", "a", "b", "c", "d")

# 输出 False
print(redis_conn.sismember("myset", "e"))

# 输出 True
print(redis_conn.sismember("myset", "b"))
```

每个命令函数预期的参数与 [redis.io](https://redis.io/commands/) 上命令页面中记录的参数相同。

如果您不想从模型类获取 Redis 连接，您还可以使用 `get_redis_connection`：

```python
from redis_om import get_redis_connection

redis_conn = get_redis_connection()
redis_conn.set("hello", "world")
```

## 下一步

现在您已经了解了使用 Redis OM 的基础知识，可以在您的项目中开始尝试它！

如果您是 FastAPI 用户，请查看 [如何将 Redis OM 集成到 FastAPI 中](fastapi_integration.md)。

<!-- Links -->

[redisearch-url]: https://redis.io/docs/stack/search/
[redis-json-url]: https://redis.io/docs/stack/json/
[pydantic-url]: https://github.com/samuelcolvin/pydantic
