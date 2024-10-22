# 模型和字段

Redis OM 对象映射、验证和查询功能的核心是一对声明性模型：`HashModel` 和 `JsonModel`。这两个模型提供大致相同的 API，但它们在 Redis 中存储数据的方式不同。

本页将解释如何通过子类化这些类之一来创建您的 Redis OM 模型。

## HashModel 与 JsonModel

首先，您应该使用哪个？

选择相对简单。如果您想将一个模型嵌入到另一个模型中，比如将一个 `Customer` 模型与一个 `Order` 模型列表关联，那么您需要使用 `JsonModel`。只有 `JsonModel` 支持嵌入模型。

否则，使用 `HashModel`。

## 创建您的模型

您可以通过子类化 `HashModel` 或 `JsonModel` 来创建 Redis OM 模型。例如：

```python
from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
```

## 配置模型

您可以在模型中配置多个特定于 Redis OM 的设置。您可以使用一个称为 _Meta 对象_ 的特殊对象来配置这些设置。

以下是使用 Meta 对象设置全局键前缀的示例：

```python
from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str

    class Meta:
        global_key_prefix = "customer-dashboard"
```

## 抽象模型

您可以通过在 `HashModel` 或 `JsonModel` 之上子类化 `ABC` 来创建抽象 Redis OM 模型。抽象模型仅存在于为子类收集共享配置的目的——您无法实例化它们。

抽象模型的一个用途是配置应用程序中所有模型都将使用的 Redis 键前缀。这是与 Redis 的一个良好最佳实践。以下是使用抽象模型的方法：

```python
from abc import ABC

from redis_om import HashModel


class BaseModel(HashModel, ABC):
    class Meta:
        global_key_prefix = "your-application"
```

### Meta 对象是“特殊的”

Meta 对象有一个特殊的属性：如果您从具有 Meta 对象的基类创建模型子类，Redis OM 会将父类的字段复制到子类的 Meta 对象中。

因此，子类可以在其父类的 Meta 类中重写单个字段，而无需重新定义所有字段。

一个示例将使这一点更加清晰：

```python
from abc import ABC

from redis_om import HashModel, get_redis_connection


redis = get_redis_connection(port=6380)
other_redis = get_redis_connection(port=6381)


class BaseModel(HashModel, ABC):
    class Meta:
        global_key_prefix = "customer-dashboard"
        database = redis


class Customer(BaseModel):
    first_name: str
    last_name: str

    class Meta:
        database = other_redis


print(Customer.global_key_prefix)
# > "customer-dashboard"
```

在这个例子中，我们创建了一个名为 `BaseModel` 的抽象基模型，并为其提供了一个包含数据库连接和全局键前缀的 Meta 对象。

然后我们创建了一个名为 `Customer` 的 `BaseModel` 子类，并为其提供了第二个 Meta 对象，但只定义了 `database`。`Customer` _也获得了 `BaseModel` 定义的全局键前缀_（"customer-dashboard"）。

虽然这与 Python 中通常的对象继承方式不同，但我们认为这有助于使抽象模型更有用，尤其是作为分组共享模型设置的一种方式。

### Meta 对象支持的所有设置

以下是 Meta 对象中可用设置及其控制内容的表格。

| 设置                    | 描述                                                                                                                                                      | 默认值                                                 |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| global_key_prefix       | 应用于模型管理的每个 Redis 键的字符串前缀。这可以是您应用程序的名称。                                                                                     | ""                                                     |
| model_key_prefix        | 应用于表示每个模型的 Redis 键的字符串前缀。例如，HashModel 的 Redis Hash 键。此前缀也会添加到为每个具有索引字段的模型创建的 redisearch 索引中。           | f"{new_class.__module__}.{new_class.__name__}"         |
| primary_key_pattern     | 生成表示此模型的 Redis 键的基础字符串的格式字符串。此字符串应接受一个 "pk" 格式参数。**注意：**这是一个“新风格”的格式字符串，将通过 `.format()` 调用。    | "{pk}"                                                 |
| database                | 模型将用于与 Redis 通信的 redis.asyncio.Redis 或 redis.Redis 客户端实例。                                                                                 | 使用 connections.get_redis_connection() 创建的新实例。 |
| primary_key_creator_cls | 遵循 PrimaryKeyCreator 协议的类，Redis OM 将使用该类为新的模型实例创建主键。                                                                              | UlidPrimaryKey                                         |
| index_name              | 此模型使用的 RediSearch 索引名称。仅在至少一个模型的字段被标记为可索引（`index=True`）时使用。                                                            | "{global_key_prefix}:{model_key_prefix}:index"         |
| embedded                | 此模型是否为“嵌入式”。嵌入式模型不包括在创建和销毁索引的迁移中。相反，它们的索引字段包含在父模型的索引中。**注意**：只有 `JsonModel` 可以拥有嵌入式模型。 | False                                                  |
| encoding                | 用于字符串的默认编码。此编码在连接级别传递给 redis-py。在这两种情况下，Redis OM 将使用您选择的编码解码来自 Redis 的二进制字符串。                         | "utf-8"                                                |

## 配置 Pydantic

每个 Redis OM 模型也是一个 Pydantic 模型，因此除了使用 Meta 对象配置 Redis OM 行为外，您还可以通过模型类中的 Config 对象控制 Pydantic 配置。

有关此对象的工作方式和可用设置的详细信息，请参阅 [Pydantic 文档](https://pydantic-docs.helpmanual.io/usage/model_config/)。

Redis OM 为模型设置的默认 Pydantic 配置相当于以下内容（在实际模型上演示）：

```python
from redis_om import HashModel


class Customer(HashModel):
    # ... 字段 ...

    model_config = ConfigDict(
        from_attributes=True,
        arbitrary_types_allowed=True,
        extra="allow",
    )
```

如果您更改这些设置，某些功能可能无法正常工作。

## 字段

您可以使用 Python _类型注解_ 在 Redis OM 模型上定义字段。如果您对类型注解不熟悉，可以查看这个 [教程](https://towardsdatascience.com/type-annotations-in-python-d90990b172dc)。

这与 Pydantic 的工作方式完全相同。有关字段类型的指导，请查看 [Pydantic 文档](https://pydantic-docs.helpmanual.io/usage/types/)。

### 使用 HashModel

`HashModel` 将数据存储在 Redis Hash 中，结构是扁平的。这意味着 Redis Hash 不能包含 Redis Set、List 或 Hash。由于这个要求，`HashModel` 当前也不支持容器类型，例如：

* 集合
* 列表
* 字典和其他“映射”类型
* 其他 Redis OM 模型
* Pydantic 模型

**注意**：将来，我们可能会将这些值序列化为 JSON 字符串，就像我们对 `JsonModel` 所做的那样。不同之处在于，在 `HashModel` 的情况下，您将无法对这些字段进行索引，只能获取和保存它们。而在 `JsonModel` 中，您可以对列表字段和嵌入的 `JsonModel` 进行索引。

因此，简而言之，如果您想使用容器类型，请使用 `JsonModel`。

### 使用 JsonModel

好消息！`JsonModel` 支持容器类型。

我们将使用 Pydantic 的 JSON 序列化和编码来序列化您的 `JsonModel` 并将其保存在 Redis 中。

### 默认值

字段可以具有默认值。您通过为字段赋值来设置它们。

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
    bio: Optional[str] = "Super dope"  # <- 在这里添加了默认值
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

## 标记字段为索引

如果您在 Redis 实例中使用 RediSearch 模块，可以将字段标记为“索引”。一旦您将模型中的任何字段标记为索引，Redis OM 会自动为该模型创建和管理一个二级索引，使您能够对任何索引字段进行查询。

要将字段标记为索引，您需要使用 Redis OM 的 `Field()` 助手，如下所示：

```python
from redis_om import (
    Field,
    HashModel,
)


class Customer(HashModel):
    first_name: str
    last_name: str = Field(index=True)
```

在这个例子中，我们将 `Customer.last_name` 标记为索引。

要为具有索引字段的任何模型创建索引，请使用 Redis OM 安装在您的 Python 环境中的 `migrate` CLI 命令。

该命令会检测您项目中的任何 `JsonModel` 或 `HashModel` 实例，并对每个不是抽象或嵌入的模型执行以下操作：

* 如果模型尚未存在索引：
  * 迁移工具创建一个索引
  * 迁移工具存储索引定义的哈希
* 如果模型存在索引：
  * 迁移工具检查存储的索引哈希是否过时
  * 如果存储的哈希过时，迁移工具会删除索引（不会删除您的数据！）并使用新的索引定义重建它

您也可以通过代码自行运行 `Migrator`：

```python
from redis_om import (
    get_redis_connection,
    Migrator
)

redis = get_redis_connection()
Migrator().run()
```
