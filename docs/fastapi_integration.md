# FastAPI 集成

## 介绍

好消息：Redis OM 是专门设计为与 FastAPI 集成的！

本节包含一个完整的示例，展示如何将 Redis OM 与 FastAPI 集成。

## 概念

### 每个 Redis OM 模型也是一个 Pydantic 模型

每个 Redis OM 模型也是一个 Pydantic 模型，因此您可以定义一个模型，然后在 FastAPI 期望 Pydantic 模型的任何地方使用该模型类。

这意味着几个事情：

1. Redis OM 模型可以用于请求体验证
2. Redis OM 模型会出现在自动生成的 API 文档中

### 缓存与数据

Redis 既可以作为持久数据存储，也可以作为缓存，但这两种用例的最佳 Redis 配置通常不同。

在进行缓存时，您几乎总是希望使用经过缓存优化的 Redis 实例，而在存储应用状态时，使用经过数据持久性优化的独立 Redis 实例。

本示例展示了如何在同一个应用程序中管理这两种 Redis 使用。该应用使用 FastAPI 缓存框架和专用的缓存 Redis 实例进行缓存，并使用一个经过持久性优化的独立 Redis 实例来处理 Redis OM 模型。

## 示例应用代码

让我们看看一个使用 Redis OM 的 FastAPI 应用示例。

**注意**：此示例代码需要依赖项才能运行。要安装依赖项，首先从 GitHub 克隆 [redis-om-fastapi](https://github.com/redis-developer/redis-om-fastapi) 仓库。然后按照本文档后面的安装步骤或该仓库的 README.md 文件中的步骤进行操作。

```python
import datetime
from typing import Optional

import redis

from fastapi import FastAPI, HTTPException
from starlette.requests import Request
from starlette.responses import Response

from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend
from fastapi_cache.decorator import cache

from pydantic import EmailStr

from redis_om import HashModel, NotFoundError
from redis_om import get_redis_connection

# 这个 Redis 实例经过持久化优化。
REDIS_DATA_URL = "redis://localhost:6380"

# 这个 Redis 实例经过缓存性能优化。
REDIS_CACHE_URL = "redis://localhost:6381"


class Customer(HashModel):
    first_name: str
    last_name: str
    email: EmailStr
    join_date: datetime.date
    age: int
    bio: Optional[str]


app = FastAPI()


@app.post("/customer")
async def save_customer(customer: Customer):
    # 我们可以通过调用 `save()` 将模型保存到 Redis：
    return customer.save()


@app.get("/customers")
async def list_customers(request: Request, response: Response):
    # 要通过主键检索此客户，我们使用 `Customer.get()`：
    return {"customers": Customer.all_pks()}


@app.get("/customer/{pk}")
@cache(expire=10)
async def get_customer(pk: str, request: Request, response: Response):
    # 要通过主键检索此客户，我们使用 `Customer.get()`：
    try:
        return Customer.get(pk)
    except NotFoundError:
        raise HTTPException(status_code=404, detail="客户未找到")


@app.on_event("startup")
async def startup():
    r = redis.asyncio.from_url(REDIS_CACHE_URL, encoding="utf8",
                          decode_responses=True)
    FastAPICache.init(RedisBackend(r), prefix="fastapi-cache")

    # 您可以通过 REDIS_OM_URL 环境变量设置 Redis OM URL，
    # 或者通过使用模型的 Meta 对象手动创建连接。
    Customer.Meta.database = get_redis_connection(url=REDIS_DATA_URL,
                                                  decode_responses=True)
```

## 测试应用

您应该先安装应用的依赖项。该应用使用 Poetry，因此您需要确保先安装 Poetry：

    $ pip install poetry

然后安装依赖项：

    $ poetry install

接下来，启动服务器：

    $ poetry run uvicorn --reload main:app

然后，在另一个终端中，创建一个客户：
```
    $ curl -X POST "http://localhost:8000/customer" -H 'Content-Type: application/json' -d '{"first_name":"Andrew","last_name":"Brookins","email":"a@example.com","age":"38","join_date":"2020-01-02"}'
    {"pk":"01FM2G8EP38AVMH7PMTAJ123TA","first_name":"Andrew","last_name":"Brookins","email":"a@example.com","join_date":"2020-01-02","age":38,"bio":""}
```

获取 "pk" 的值，这是模型的主键，并发出另一个请求以获取该客户：

    $ curl "http://localhost:8000/customer/01FM2G8EP38AVMH7PMTAJ123TA"
    {"pk":"01FM2G8EP38AVMH7PMTAJ123TA","first_name":"Andrew","last_name":"Brookins","email":"a@example.com","join_date":"2020-01-02","age":38,"bio":""}

您还可以获取所有客户主键的列表：

    $ curl "http://localhost:8000/customers"
    {"customers":["01FM2G8EP38AVMH7PMTAJ123TA"]}

## Redis OM 与 Asyncio

Redis OM 设计为与 asyncio 一起使用，因此您可以在 FastAPI 应用程序中异步使用 Redis OM 模型。

唯一的区别是您从 `aredis_om` 模块导入 Redis OM 模型，而不是 `redis_om` 模块。

以下是之前的 FastAPI 应用，但使用与 asyncio 兼容的 Redis OM 代码：

```python
import datetime
from typing import Optional

import redis

from fastapi import FastAPI, HTTPException
from starlette.requests import Request
from starlette.responses import Response

from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend
from fastapi_cache.decorator import cache

from pydantic import EmailStr

from aredis_om import HashModel, NotFoundError  # <- 注意，我们从 aredis_om 导入
from aredis_om import get_redis_connection

# 这个 Redis 实例经过持久化优化。
REDIS_DATA_URL = "redis://localhost:6380"

# 这个 Redis 实例经过缓存性能优化。
REDIS_CACHE_URL = "redis://localhost:6381"


class Customer(HashModel):
    first_name: str
    last_name: str
    email: EmailStr
    join_date: datetime.date
    age: int
    bio: Optional[str]


app = FastAPI()


@app.post("/customer")
async def save_customer(customer: Customer):
    # 我们可以通过调用 `save()` 将模型保存到 Redis：
    return await customer.save()  # <- 我们在这里使用 await


@app.get("/customers")
async def list_customers(request: Request, response: Response):
    # 要通过主键检索此客户，我们使用 `Customer.get()`：
    return {"customers": await Customer.all_pks()}  # <- 我们在这里也使用 await


@app.get("/customer/{pk}")
@cache(expire=10)
async def get_customer(pk: str, request: Request, response: Response):
    # 要通过主键检索此客户，我们使用 `Customer.get()`：
    try:
        return await Customer.get(pk)  # <- 最后，再次使用 await！
    except NotFoundError:
        raise HTTPException(status_code=404, detail="客户未找到")


@app.on_event("startup")
async def startup():
    r = redis.asyncio.from_url(REDIS_CACHE_URL, encoding="utf8",
                          decode_responses=True)
    FastAPICache.init(RedisBackend(r), prefix="fastapi-cache")

    # 您可以通过 REDIS_OM_URL 环境变量设置 Redis OM URL，
    # 或者通过使用模型的 Meta 对象手动创建连接。
    Customer.Meta.database = get_redis_connection(url=REDIS_DATA_URL,
                                                  decode_responses=True)
```

**注意：** 模块 `redis_om` 和 `aredis_om` 在几乎所有方面都是相同的。唯一的区别是 `aredis_om` 返回协程，您必须 `await`。
