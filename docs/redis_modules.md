# Redis 模块

Redis OM 的一些高级功能，如丰富的查询表达式和以 JSON 格式保存数据，依赖于两个可用的 Redis 模块的核心功能：**RediSearch** 和 **RedisJSON**。

这些模块是幕后“魔法”的源泉：

* RediSearch 为 Redis 添加了查询、索引和全文搜索功能
* RedisJSON 为 Redis 添加了 JSON 数据类型

## 为什么这很重要

如果没有安装 RediSearch 或 RedisJSON，您仍然可以使用 Redis OM 创建由 Redis 支持的声明式模型。

我们将您的模型数据作为 Hash 存储在 Redis 中，您可以使用主键检索模型。您还将获得 Pydantic 的所有验证功能。

那么，没有这些模块，什么将无法工作？

1. 没有 RedisJSON，您将无法将模型嵌套在一起，例如我们在 `Customer` 模型中嵌入 `Address` 的示例模型。
2. 没有 RediSearch，您将无法使用我们的表达式查询找到模型——只能使用主键。

## 那么如何获取 RediSearch 和 RedisJSON？

您可以在自托管的 Redis 部署中使用 RediSearch 和 RedisJSON。只需按照它们的快速入门指南中的说明安装模块的二进制版本：

- [RedisJSON 快速入门 - 运行二进制文件](https://redis.io/docs/stack/json/)
- [RediSearch 快速入门 - 运行二进制文件](https://redis.io/docs/stack/search/quick_start/)

**注意**：这两个模块的快速入门指南中也有关于如何使用 Docker 在 Redis 中运行这些模块的说明。

不想自己运行 Redis？RediSearch 和 RedisJSON 也可以在 Redis Cloud 上使用。 [在这里开始。](https://redis.com/try-free/)