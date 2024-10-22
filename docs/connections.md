# 连接到 Redis

您可以通过 `REDIS_OM_URL` 环境变量控制 Redis OM 如何连接到 Redis，或者手动构造 Redis 客户端对象。

## 环境变量

默认情况下，Redis OM 尝试连接到您本地主机上的 6379 端口。大多数本地安装方法会导致 Redis 在此位置运行，在这种情况下，您无需为 Redis OM 连接 Redis 做任何特别的设置。

然而，如果您将 Redis 配置为在不同的端口上运行，或者如果您正在使用远程 Redis 服务器，您需要设置 `REDIS_OM_URL` 环境变量。

`REDIS_OM_URL` 环境变量遵循 redis-py URL 格式：

    redis://[[username]:[password]]@localhost:6379/[database number]

**注意：** 方括号表示可选值，并不是 URL 格式的一部分。

默认连接相当于以下 `REDIS_OM_URL` 环境变量：

    redis://localhost:6379

**注意：** 索引仅适用于存储在 Redis 逻辑数据库 0 中的数据。如果在连接到 Redis 时使用不同的数据库编号，您可以期待在运行迁移器时代码会引发 `MigrationError`。

### 密码和用户名

Redis 可以配置为密码保护和“默认”用户，在这种情况下，您可能只需使用密码连接。

您可以像这样使用 Redis OM：

    redis://:your-password@localhost:6379

如果您的 Redis 实例需要用户名和密码，您应在 URL 中包含两者：

    redis://your-username:your-password@localhost:6379

### 数据库编号

Redis 数据库是编号的，默认是 0。您可以省略数据库编号以使用默认数据库，或指定它。

**注意：** 索引仅适用于存储在 Redis 逻辑数据库 0 中的数据。如果在连接到 Redis 时使用不同的数据库编号，您可以期待在运行迁移器时代码会引发 `MigrationError`。

### SSL 连接

使用“rediss”前缀进行 SSL 连接：

    rediss://[[username]:[password]]@localhost:6379/0

### Unix 域套接字

使用“unix”前缀通过 Unix 域套接字连接到 Redis：

    unix://[[username]:[password]]@/path/to/socket.sock?db=0

### 了解更多

要了解更多关于 Redis OM Python 使用的 URL 格式的信息，请查阅 [redis-py URL 文档](https://redis-py.readthedocs.io/en/stable/#redis.Redis.from_url)。

**提示：** 如果您在使用异步或同步模式与 Redis OM（即，为异步导入 `aredis_om` 或为同步导入 `redis_om`），则 URL 格式是相同的。

## 连接对象

除了通过 `REDIS_OM_URL` 环境变量控制连接外，您还可以手动为特定的 OM 模型类构造 Redis 客户端连接。

**注意：** 此方法优先于 `REDIS_OM_URL` 环境变量。

您可以通过将对象分配给模型的 _Meta_ 对象的 *database* 字段来控制特定模型类应使用的连接，如下所示：

```python
from redis_om import HashModel, get_redis_connection


redis = get_redis_connection(port=6378)


class Customer(HashModel):
    first_name: str
    last_name: str
    age: int

    class Meta:
        database = redis
```

`get_redis_connection()` 函数是一个 Redis OM 辅助函数，它将关键字参数传递给 `redis.asyncio.Redis.from_url()` 或 `redis.Redis.from_url()`，具体取决于您是在异步还是同步模式下使用 Redis OM。

您还可以手动构造客户端对象：

```python
from redis import Redis

from redis_om import HashModel


class Customer(HashModel):
    first_name: str
    last_name: str
    age: int

    class Meta:
        database = Redis(port=6378)
```
