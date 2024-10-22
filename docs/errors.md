# 错误

本页列出了 Redis OM 在使用过程中可能产生的错误，并提供了更多的错误上下文信息。

## E1

> 为了对列表字段进行查询，您必须使用类型注解定义列表的内容，例如：orders: List[Order]。

如果您尝试在不是列表的字段上使用 "IN" 查询，例如 `await TarotWitch.find(TarotWitch.tarot_cards << "death").all()`，您将看到此错误。

在这个例子中，`TarotWitch.tarot_cards` 是一个列表，因此查询有效：

```python
from typing import List

from redis_om import JsonModel, Field

class TarotWitch(JsonModel):
    tarot_cards: List[str] = Field(index=True)
```

但如果 `tarot_cards` 不是列表，尝试使用 `<<` 查询将导致此错误。

## E2

> 您尝试按 {field_name} 排序，但 {self.model} 并未将该字段定义为可排序。

您尝试按一个不可排序的字段对查询结果进行排序。以下是如何将字段标记为可排序：

```python
from typing import List

from redis_om import JsonModel, Field

class Member(JsonModel):
    age: int = Field(index=True, sortable=True)
```

**注意：** 只有已索引字段才能是可排序的。

## E3

> 您尝试在字段 '{field.name}' 上进行全文搜索，但该字段未被索引以进行全文搜索。请使用 full_text_search=True 选项。

您可以使用模块 (`%`) 运算符进行全文搜索。这样的查询看起来像这样：

```python
from redis_om import JsonModel, Field

class Member(JsonModel):
    bio: str = Field(index=True, full_text_search=True, default="")

Member.find(Member.bio % "beaches").all()
```

如果您看到此错误，意味着您查询的字段（示例中的 `bio`）未被索引以进行全文搜索。确保同时将字段标记为 `index=True` 和 `full_text_search=True`，如示例所示。

## E4

> 仅支持列表和元组作为多值字段。

这意味着您将一个字段标记为 `index=True`，但该字段不是 Redis OM 实际可以索引的类型。

具体而言，您可能使用了下标注释，例如 `Dict[str, str]`。Redis OM 只能索引的下标类型是 `List` 和 `Tuple`。

## E5

> 仅支持等于 (=)、不等于 (!=) 和 like() 比较用于 TEXT 字段。

您正在查询一个您标记为全文搜索的已索引字段。您只能使用相等（==）、不相等（!=）和 like `(%)` 运算符查询这些字段。

```python
from redis_om import JsonModel, Field

class Member(JsonModel):
    bio: str = Field(index=True, full_text_search=True, default="")

# 等于
Member.find(Member.bio == "Programmer").all()

# 不等于
Member.find(Member.bio != "Programmer").all()

# Like（全文搜索）。这将 "programming"
# 词干提取，以找到任何匹配的相同词干的术语，
# "program"。
Member.find(Member.bio % "programming").all()
```

## E6

> 您尝试查询一个未被索引的字段 ({field_name})。

您编写了一个查询，使用了一个未被标记为索引的模型字段。您只能查询已索引的字段。以下示例代码将生成此错误：

```python
from redis_om import JsonModel, Field

class Member(JsonModel):
    first_name: str 
    bio: str = Field(index=True, full_text_search=True, default="")

# 因为我们没有将 `first_name` 标记为索引，
# 所以引发了 QueryNotSupportedError！
Member.find(Member.first_name == "Andrew").all()
```

通过将字段标记为索引来修复此问题：

```python
from redis_om import JsonModel, Field

class Member(JsonModel):
    first_name: str = Field(index=True)
    bio: str = Field(index=True, full_text_search=True, default="")

# 因为我们没有将 `first_name` 标记为索引，
# 所以引发了 QueryNotSupportedError！
Member.find(Member.first_name == "Andrew").all()
```

## E7

> 查询表达式应以字段或括号内的表达式开头。

我们在尝试解析您的查询表达式时感到困惑。这不是您的问题，是我们的！一些代码示例可能会有所帮助……

```python
from redis_om import JsonModel, Field

class Member(JsonModel):
    first_name: str = Field(index=True)
    last_name: str = Field(index=True)

# 只有一个运算符的查询通常很简单：
Member.find(Member.first_name == "Andrew").all()

# 如果您想添加多个条件，可以将它们用 AND 结合在一起，
# 通过将条件一个接一个地作为参数包含。
Member.find(Member.first_name=="Andrew",
            Member.last_name=="Brookins").all()

# 或者，您可以用括号分隔条件，并使用显式的 AND。
Member.find(
    (Member.first_name == "Andrew") & ~(Member.last_name == "Brookins")
).all()

# 您不能使用 `!` 表示 NOT。相反，请使用 `~`。
Member.find(
    (Member.first_name == "Andrew") & 
    ~(Member.last_name == "Brookins")  # <- 请注意，这个现在是 NOT！
).all()

# 括号是构建更复杂查询的关键，
# 像这样。
Member.find(
    ~(Member.first_name == "Andrew")
    & ((Member.last_name == "Brookins") | (Member.last_name == "Smith"))
).all()

# 如果您对 Redis OM 如何解释查询感到困惑，
# 可以使用 `tree()` 方法可视化 `FindQuery` 的表达式树。
query = Member.find(
    ~(Member.first_name == "Andrew")
    & ((Member.last_name == "Brookins") | (Member.last_name == "Smith"))
)
print(query.expression.tree)
"""
           ┌first_name
    ┌NOT EQ┤
    |      └Andrew
 AND┤
    |     ┌last_name
    |  ┌EQ┤
    |  |  └Brookins
    └OR┤
       |  ┌last_name
       └EQ┤
          └Smith
"""
```

## E8

> 您只能用 AND (&) 或 OR (|) 组合两个查询表达式。

您可以在查询中组合表达式的唯一两个运算符是 `&` 和 `|`。您可能意外使用了其他运算符，或者 Redis OM 可能感到困惑。确保您使用括号来组织查询表达式。

如果您试图使用 "NOT"，可以通过在查询前加上 `~` 运算符来实现，例如：

```python
from redis_om import JsonModel, Field

class Member(JsonModel):
    first_name: str = Field(index=True)
    last_name: str = Field(index=True)

# 查找不叫 Andrew 的人。
Member.find(~(Member.first_name == "Andrew")).all()
```

请注意，这种形式要求在您要 "否定" 的表达式周围加上括号。当然，这个例子用 `!=` 也更合适：

```python
from redis_om import JsonModel, Field

class Member(JsonModel):
    first_name: str = Field(index=True)
    last_name: str = Field(index=True)

# 查找不叫 Andrew 的人。
Member.find(Member.first_name != "Andrew").all()
```

不过，`~` 在否定括号包围的表达式组时非常有用。
