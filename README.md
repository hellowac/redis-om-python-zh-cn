<div align="center">
  <br/>
  <br/>
  <img width="360" src="https://raw.githubusercontent.com/redis-developer/redis-om-python/main/images/logo.svg?token=AAAXXHUYL6RHPESRRAMBJOLBSVQXE" alt="Redis OM" />
  <br/>
  <br/>
</div>

<p align="center">
    <p align="center">
        对象映射, 以及更多, 基于 Redis 和 Python
    </p>
</p>

---

[![Version][version-svg]][package-url]
[![License][license-image]][license-url]
[![Build Status][ci-svg]][ci-url]

**Redis OM Python** 使在 Python 应用程序中建模 Redis 数据变得简单。

[Redis OM .NET](https://github.com/redis/redis-om-dotnet) | [Redis OM Node.js](https://github.com/redis/redis-om-node) | [Redis OM Spring](https://github.com/redis/redis-om-spring) | **Redis OM Python**

<details>
  <summary><strong>目录</strong></summary>

span

<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [💡 为什么选择 Redis OM？](#-为什么选择-redis-om)
- [💻 安装](#-安装)
- [🏁 快速开始](#-快速开始)
    - [启动 Redis](#启动-redis)
- [📇 数据建模](#-数据建模)
- [✓ 使用模型验证数据](#-使用模型验证数据)
- [🔎 丰富的查询和嵌入式模型](#-丰富的查询和嵌入式模型)
    - [查询](#查询)
    - [嵌入式模型](#嵌入式模型)
- [调用其他 Redis 命令](#调用其他-redis-命令)
- [📚 文档](#-文档)
- [⛏️ 故障排除](#️-故障排除)
- [✨ 如何获取 RediSearch 和 RedisJSON？](#-如何获取-redisearch-和-redisjson)
- [❤️ 贡献](#️-贡献)
- [📝 许可证](#-许可证)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

</details>

## 💡 为什么选择 Redis OM？

Redis OM 提供了高级抽象，使得在现代 Python 应用程序中建模和查询 Redis 数据变得简单。

此 **预览** 版本包含以下功能：

* 对 Redis 对象的声明式对象映射
* 声明式二级索引生成
* 用于查询 Redis 的流畅 API

## 💻 安装

使用 `pip`、Poetry 或 Pipenv 安装非常简单。

```sh
# 使用 pip
$ pip install redis-om

# 或使用 Poetry
$ poetry add redis-om
```

## 🏁 快速开始

### 启动 Redis

在编写任何代码之前，您需要一个带有适当 Redis 模块的 Redis 实例！最快的方法是使用 Docker：

```sh
docker run -p 6379:6379 -p 8001:8001 redis/redis-stack
```

这将启动 [redis-stack](https://redis.io/docs/stack/)，它是 Redis 的扩展，为 Redis 添加了各种现代数据结构。您还会注意到，如果打开 http://localhost:8001，您将可以访问 redis-insight GUI，这是一个可视化和操作 Redis 中数据的图形界面。

## 📇 数据建模

Redis OM 包含强大的声明式模型，提供数据验证、序列化和持久化到 Redis。

查看这个使用 Redis OM 建模客户数据的示例。首先，我们创建一个 `Customer` 模型：

```python
import datetime
from typing import Optional

from pydantic import EmailStr

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: EmailStr
    join_date: datetime.date
    age: int
    bio: Optional[str] = None
```

现在我们有了 `Customer` 模型，接下来用它将客户数据保存到 Redis。

```python
import datetime
from typing import Optional

from pydantic import EmailStr

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: EmailStr
    join_date: datetime.date
    age: int
    bio: Optional[str] = None


# 首先，我们创建一个新的 `Customer` 对象：
andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38,
    bio="Python developer, works at Redis, Inc."
)

# 该模型会自动生成一个全球唯一的主键
# 无需与 Redis 通信。
print(andrew.pk)
# > "01FJM6PH661HCNNRC884H6K30C"

# 我们可以通过调用 `save()` 将模型保存到 Redis：
andrew.save()

# 设置模型在 2 分钟（120 秒）后过期
andrew.expire(120)

# 要使用主键检索此客户，我们使用 `Customer.get()`：
assert Customer.get(andrew.pk) == andrew
```

**准备好了解更多？** 请查看 [快速入门](docs/getting_started.md) 指南。

或者，继续阅读以了解 Redis OM 如何简化数据验证。

## ✓ 使用模型验证数据

Redis OM 使用 [Pydantic][pydantic-url] 根据您在模型类中分配给字段的类型注释来验证数据。

此验证确保像 `first_name` 这样的字段（在 `Customer` 模型中标记为 `str`）始终是字符串。**但是每个 Redis OM 模型也是一个 Pydantic 模型**，因此您可以使用 Pydantic 验证器，如 `EmailStr`、`Pattern` 等，进行复杂的验证！

例如，由于我们对 `email` 字段使用了 `EmailStr` 类型，如果我们尝试创建一个无效电子邮件地址的 `Customer`，将会得到验证错误：

```python
import datetime
from typing import Optional

from pydantic import EmailStr, ValidationError

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    email: EmailStr
    join_date: datetime.date
    age: int
    bio: Optional[str] = None


try:
    Customer(
        first_name="Andrew",
        last_name="Brookins",
        email="Not an email address!",
        join_date=datetime.date.today(),
        age=38,
        bio="Python developer, works at Redis, Inc."
    )
except ValidationError as e:
    print(e)
    """
    pydantic.error_wrappers.ValidationError: 1 validation error for Customer
     email
       value is not a valid email address (type=value_error.email)
    """
```

**任何现有的 Pydantic 验证器都应该可以**作为 Redis OM 模型的即插即用类型注释。您也可以编写任意复杂的自定义验证！

要了解更多信息，请查看 [数据验证文档](docs/validation.md)。

## 🔎 丰富的查询和嵌入式模型

数据建模、验证和将模型保存到 Redis 的所有功能都可以在任何 Redis 运行方式下使用。

接下来，我们将展示当您的 Redis 部署安装了 [RediSearch][redisearch-url] 和 [RedisJSON][redis-json-url] 模块时，Redis OM 提供的 **丰富查询表达式** 和 **嵌入式模型**。

**提示**：*等一下，Redis 模块是什么？* 如果您不熟悉 Redis 模块，请查看本 README 的 [那么，如何获取 RediSearch 和 RedisJSON？](#-so-how-do-you-get-redisearch-and-redisjson) 部分。

### 查询

Redis OM 提供了丰富的查询语言，允许您使用 Python 表达式查询 Redis。

为了展示其工作原理，我们将对之前定义的 `Customer` 模型进行小改动。我们将添加 `Field(index=True)`，以告知 Redis OM 我们希望为 `last_name` 和 `age` 字段建立索引：

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
    bio: Optional[str] = None


# 现在，如果我们在安装了 RediSearch 模块的 Redis 部署中使用这个模型，
# 我们可以运行如下查询。

# 在运行查询之前，我们需要运行迁移以设置 Redis OM 将使用的索引。
# 您也可以使用 `migrate` CLI 工具来完成此操作！
Migrator().run()

# 找到所有姓 "Brookins" 的客户
Customer.find(Customer.last_name == "Brookins").all()

# 找到所有不姓 "Brookins" 的客户
Customer.find(Customer.last_name != "Brookins").all()

# 找到所有姓 "Brookins" 的客户，或者年龄为 100 且姓 "Smith" 的客户
Customer.find((Customer.last_name == "Brookins") | (
        Customer.age == 100
) & (Customer.last_name == "Smith")).all()
```

这些查询——以及更多——之所以可能，是因为 **Redis OM 自动为您管理索引**。

使用此索引进行查询具有丰富的表达语法，灵感来自 Django ORM、SQLAlchemy 和 Peewee。我们认为您会喜欢它！

**注意：** 索引仅适用于存储在 Redis 逻辑数据库 0 中的数据。如果在连接到 Redis 时使用了不同的数据库编号，您可以预期在运行迁移器时代码会引发 `MigrationError`。

### 嵌入式模型

Redis OM 可以像任何文档数据库一样存储和查询 **嵌套模型**，同时享受 Redis 带来的速度和强大功能。让我们看看这如何实现。

在下一个示例中，我们将定义一个新的 `Address` 模型，并将其嵌入到 `Customer` 模型中。

```python
import datetime
from typing import Optional

from redis_om import (
    EmbeddedJsonModel,
    JsonModel,
    Field,
    Migrator,
)


class Address(EmbeddedJsonModel):
    address_line_1: str
    address_line_2: Optional[str] = None
    city: str = Field(index=True)
    state: str = Field(index=True)
    country: str
    postal_code: str = Field(index=True)


class Customer(JsonModel):
    first_name: str = Field(index=True)
    last_name: str = Field(index=True)
    email: str = Field(index=True)
    join_date: datetime.date
    age: int = Field(index=True)
    bio: Optional[str] = Field(index=True, full_text_search=True,
                               default="")

    # 创建嵌入式模型。
    address: Address


# 有了这两个模型以及安装了 RedisJSON 模块的 Redis 部署，我们可以运行如下查询。

# 在运行查询之前，我们需要运行迁移以设置 Redis OM 将使用的索引。
# 您也可以使用 `migrate` CLI 工具来完成此操作！
Migrator().run()

# 找到所有居住在圣安东尼奥，德克萨斯州的客户
Customer.find(Customer.address.city == "San Antonio",
              Customer.address.state == "TX")
```

## 调用其他 Redis 命令

有时您需要直接运行 Redis 命令。Redis OM 通过模型类上的 `db` 方法支持这一点。它返回一个连接的 Redis 客户端实例，该实例公开了一个以每个 Redis 命令命名的函数。例如，我们来执行一些基本的集合操作：

```python
from redis_om import HashModel

class Demo(HashModel):
    some_field: str

redis_conn = Demo.db()

redis_conn.sadd("myset", "a", "b", "c", "d")

# 打印 False
print(redis_conn.sismember("myset", "e"))

# 打印 True
print(redis_conn.sismember("myset", "b"))
```

每个命令函数所期望的参数与 [redis.io](https://redis.io/commands/) 上命令页面的文档相同。

如果您不想从模型类获取 Redis 连接，也可以使用 `get_redis_connection`：

```python
from redis_om import get_redis_connection

redis_conn = get_redis_connection()
redis_conn.set("hello", "world")
```

## 📚 文档

Redis OM 文档可以在 [此处](docs/index.md) 找到。

## ⛏️ 故障排除

如果您遇到问题或有任何疑问，我们随时为您提供帮助！

请在 [Redis Discord 服务器](http://discord.gg/redis) 上联系我们，或 [在 GitHub 上打开一个问题](https://github.com/redis-developer/redis-om-python/issues/new)。

## ✨ 如何获取 RediSearch 和 RedisJSON？

Redis OM 的一些高级功能依赖于两个开源 Redis 模块的核心功能：[RediSearch][redisearch-url] 和 [RedisJSON][redis-json-url]。

您可以在自托管的 Redis 部署中运行这些模块，或者使用 [Redis Enterprise][redis-enterprise-url]，该服务包含这两个模块。

要了解更多信息，请阅读 [我们的文档](docs/redis_modules.md)。

## ❤️ 贡献

我们欢迎您的贡献！

**错误报告** 在项目的这个阶段特别有帮助。 [您可以在 GitHub 上打开一个错误报告](https://github.com/redis/redis-om-python/issues/new)。

您也可以 **贡献文档**——或者告诉我们是否需要更详细的信息。 [在 GitHub 上打开一个问题](https://github.com/redis/redis-om-python/issues/new) 以开始。

## 📝 许可证

Redis OM 使用 [MIT 许可证][license-url]。

<!-- Badges -->

[version-svg]: https://img.shields.io/pypi/v/redis-om?style=flat-square
[package-url]: https://pypi.org/project/redis-om/
[ci-svg]: https://github.com/redis/redis-om-python/actions/workflows/ci.yml/badge.svg
[ci-url]: https://github.com/redis/redis-om-python/actions/workflows/ci.yml
[license-image]: https://img.shields.io/badge/license-mit-green.svg?style=flat-square
[license-url]: LICENSE
<!-- Links -->

[redis-om-website]: https://developer.redis.com
[redis-om-js]: https://github.com/redis-om/redis-om-js
[redis-om-dotnet]: https://github.com/redis-om/redis-om-dotnet
[redis-om-spring]: https://github.com/redis-om/redis-om-spring
[redisearch-url]: https://redis.io/docs/stack/search/
[redis-json-url]: https://redis.io/docs/stack/json/
[pydantic-url]: https://github.com/samuelcolvin/pydantic
[ulid-url]: https://github.com/ulid/spec
[redis-enterprise-url]: https://redis.com/try-free/
