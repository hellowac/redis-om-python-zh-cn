# 验证

Redis OM 在后台使用 [Pydantic][pydantic-url] 来根据模型的类型注释在运行时验证数据。

## 基本类型验证

基本类型注释（如 `str`）的验证是有效的。因此，给定以下模型：

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
    bio: Optional[str]
```

... Redis OM 将确保 `first_name` 始终是一个字符串。

但每个 Redis OM 模型也是一个 Pydantic 模型，因此您可以使用现有的 Pydantic 验证器，如 `EmailStr`、`Pattern` 和更多其他复杂验证！

## 复杂验证

让我们看看如果尝试使用无效的电子邮件地址创建 `Customer` 对象会发生什么。

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
    bio: Optional[str]


# 如果尝试使用无效的电子邮件地址，我们将收到验证错误！
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

如您所见，创建 `Customer` 对象生成了以下错误：

```
 Traceback:
 pydantic.error_wrappers.ValidationError: 1 validation error for Customer
 email
   value is not a valid email address (type=value_error.email)
```

如果我们将模型实例的字段更改为无效值，然后尝试保存模型，我们也会收到验证错误：

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
    bio: Optional[str]


andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38,
    bio="Python developer, works at Redis, Inc."
)

andrew.email = "Not valid"

try:
    andrew.save()
except ValidationError as e:
    print(e)
    """
    pydantic.error_wrappers.ValidationError: 1 validation error for Customer
     email
       value is not a valid email address (type=value_error.email)
    """
```

再次，我们收到验证错误：

```
 Traceback:
 pydantic.error_wrappers.ValidationError: 1 validation error for Customer
 email
   value is not a valid email address (type=value_error.email)
```

## 约束值

如果您想使用任何约束，Pydantic 包含许多类型注释来为模型字段值引入约束。

“约束”的概念包括许多可能性：

* 始终小写的字符串
* 必须匹配正则表达式的字符串
* 在范围内的整数
* 特定倍数的整数
* 还有更多...

所有这些约束类型都可以与 Redis OM 模型一起使用。阅读 [Pydantic 文档中的约束类型](https://pydantic-docs.helpmanual.io/usage/types/#constrained-types) 以了解更多信息。

[pydantic-url]: https://github.com/samuelcolvin/pydantic