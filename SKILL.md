---
name: pythoncoding-standards
description: Python 通用代码规范 skill，覆盖命名、格式、类型标注、函数设计、文件组织和常用质量工具。涉及 PyTorch、TensorFlow、训练循环、模型结构、张量形状、实验可复现性或神经网络工程规范时，读取 references/neural-network.md。只有用户明确要求检测、审查、测试、查异常处理或查代码异味时，才读取 references/python-quality-review.md。
---

# Python 代码规范

适用于 Python 项目的通用代码规范。这个 skill 只放 Python 基础规则；神经网络、深度学习训练和模型工程相关规则放在 `references/neural-network.md`。

## 何时启用

- 编写、审查或重构 Python 代码
- 配置 Python 项目的 lint、format、type-check 或测试规则
- 统一模块、函数、变量、类和异常处理风格
- 提升代码可读性、可维护性和可测试性


## 何时读取神经网络专项文件

当任务涉及以下内容时，先读取 `references/neural-network.md`：
- PyTorch或深度学习训练代码

## 何时读取质量检测专项文件

只有用户明确要求“检测、审查、review、测试、查异常处理、查日志、查代码异味、找问题、质量检查”时，才读取 `references/python-quality-review.md`。

不要在普通编写、解释、改写 Python 代码时主动做检测式审查。

## 范围边界

本 skill 适合处理：
- Python 命名与格式
- 类型标注与数据结构
- 函数设计、类设计和模块拆分
- 文件组织、注释、配置和常用工具

质量检测专项文件适合处理：
- 异常处理与日志检查
- 测试规范
- 代码异味识别
- Python 代码审查清单


## 基础原则

### 1. 可读性优先
- 代码要让人快速理解意图
- 优先使用清晰命名，而不是短缩写
- 简单直接的实现优于炫技写法
- 注释解释“为什么”，代码本身表达“做什么”

### 2. 保持Pythonic
- 遵循 PEP 8 的基本风格
- 使用标准库和成熟生态工具，不重复造简单轮子（待确定）
- 优先使用列表推导、生成器、上下文管理器等 Python 常见表达
- 不为了“聪明”牺牲可读性

### 3. 函数
- 一个函数只做一件清晰的事
- 函数过长时，按校验、转换、执行、输出拆分

### 4. 显式优于隐式
- 输入、输出、异常和副作用要尽量明确
- 公共函数建议写类型标注
- 重要默认值要用命名常量或配置项表达

## 命名规范

### 变量命名

变量名要表达含义。不要使用无意义单字母、随意拼音缩写或只有自己知道的缩写。循环下标、数学公式、坐标轴等短生命周期变量可以例外。

```python
# 通过：名称能直接说明含义，尽量符合中文阅读习惯


# 可以接受：简短的中文拼音代替复杂不常用的英语名称


# 可以接受：短生命周期、语义明确
for i, sample in enumerate(samples):
    print(i, sample)

# 不通过：名称含义不清
flag = True
x = 1000

# 不通过：拼音缩写不利于协作
ddje = 1000
yhzt = "active"
```

### 函数命名

优先使用动词或动宾结构，表达动作和结果。

```python
# 通过：动作明确
def load_user_profile(user_id: str) -> dict:
    ...

def calculate_order_total(items: list[dict]) -> int:
    ...

def is_valid_email(email: str) -> bool:
    ...

# 不通过：名称过短或像名词，动作不明确
def user(id):
    ...

def total(items):
    ...

def email(e):
    ...
```

### 类、常量和私有成员

```python
# 类名使用 PascalCase
class OrderRepository:
    ...

# 常量使用 UPPER_SNAKE_CASE
MAX_RETRY_COUNT = 3
DEFAULT_TIMEOUT_SECONDS = 10

# 模块内部使用的函数或属性，可用单下划线前缀
def _normalize_phone_number(phone: str) -> str:
    ...
```

### 文件命名

一般是主要作用+版本号+更新功能，使用小写字母和下划线分隔。


## jupyter代码整体布局
如果没有特殊说明，就按照以下顺序布局代码：
- 导入库和模块
- 定义常量和配置项
- 定义函数和类
- 主执行逻辑（如果是脚本或 notebook）
- 其他（如测试代码、调试代码等按需求额外添加的）


## 格式与工具

### 推荐工具

- `ruff`：lint 和部分自动修复
- `black`：统一格式化
- `mypy` 或 `pyright`：类型检查
- `pytest`：测试
- `pre-commit`：提交前自动检查

项目允许时，优先在 `pyproject.toml` 中集中配置。

```toml
[tool.black]
line-length = 88

[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM"]

[tool.mypy]
python_version = "3.14.5"
warn_unused_ignores = true
warn_return_any = true
disallow_untyped_defs = true
```

### 导入顺序

导入分三组：标准库、第三方库、本项目模块。每组之间空一行。

```python
# 标准库
from pathlib import Path
import json
# 第三方库
import pandas as pd
from pydantic import BaseModel
# 本项目模块
from app.services.order_service import calculate_order_total
```

## 类型标注

公共函数、跨模块函数和复杂数据结构必须写类型标注。内部很短的小函数可以适度放宽，但不要让调用方猜输入和输出。

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class OrderItem:
    product_id: str
    unit_price_cents: int
    quantity: int


def calculate_order_total(items: list[OrderItem]) -> int:
    return sum(item.unit_price_cents * item.quantity for item in items)
```

避免滥用 `Any`。如果确实无法静态表达，必须让边界尽量小，并说明原因。

```python
from typing import Any


# 可以接受：第三方 SDK 返回结构不稳定，把 Any 限制在边界处
def parse_sdk_payload(payload: dict[str, Any]) -> str:
    raw_name = payload.get("name")
    if not isinstance(raw_name, str):
        raise ValueError("name must be a string")
    return raw_name
```

## 函数设计

### 参数不要过多

参数过多时，优先使用 dataclass、配置对象或明确的请求对象。

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ExportOptions:
    include_header: bool
    delimiter: str
    encoding: str


def export_orders(orders: list[dict], output_path: str, options: ExportOptions) -> None:
    ...
```

### 避免可变默认参数

```python
# 不通过：默认 list 会在多次调用之间共享
def add_tag(tag: str, tags: list[str] = []) -> list[str]:
    tags.append(tag)
    return tags


# 通过：使用 None 表示未传入
def add_tag(tag: str, tags: list[str] | None = None) -> list[str]:
    if tags is None:
        tags = []
    return [*tags, tag]
```

### 明确返回值

不要混合返回多种不相关类型。失败时优先抛出明确异常，或返回清晰的结果对象。

```python
def find_user_email(user_id: str) -> str | None:
    user = load_user(user_id)
    if user is None:
        return None
    return user.email
```

## 文件组织
暂无
### 推荐项目结构
### 模块职责


## 注释规范

### 通用注释
注释解释原因、约束和非显然选择，可调参数要说明含义。
每一行代码都要在右端进行注释，或者在上方进行注释一整个部分代码。
注释中出现不常见英文单词、专业术语或多个英文单词组成的概念时，应在英文后用括号补充简短中文解释；常见库名、变量名、函数名不用强行翻译。

```python
# 通过：说明参数的含义，在右端进行注释
batch_size = min(len(items), 100) # 第三方接口最多允许一次提交 100 条，超过会返回 413
batch_size = 100  # 设置批次大小

# 通过：英文术语后补充中文解释
# DeepONet query grid（DeepONet 在预测输出函数时，要查询的坐标点集合）
query_grid = build_query_grid(x_min, x_max, num_points)

# 通过：在上方进行注释一整个部分代码作用
# 左端 Neumann：第一行第二列与其他不同为-2*F
A[0, 0] = 1 + 2*F - dt*lam[0]
A[0, 1] = -2*F

# 不通过：复述代码
batch_size = 100 # 把 batch_size 设置为 100

# 不通过：复杂英文术语没有中文解释
# DeepONet query grid
query_grid = build_query_grid(x_min, x_max, num_points)
```

### 函数注释
公共函数、复杂函数和对外 API 建议写 docstring，说明参数、返回值、异常和使用示例。

```python
def calculate_discounted_price(price_cents: int, discount_rate: float) -> int:
    """
    作用：计算折扣后的价格，返回单位为分的整数金额。
    输入参数:
        price_cents: 原始价格，单位为分。
        discount_rate: 折扣率，范围在 0 到 1 之间。
    返回:
        折扣后的价格，单位为分。
    Raises:
        ValueError: 当 discount_rate 不在有效范围内时抛出。
    """

    if not 0 <= discount_rate <= 1:
        raise ValueError("discount_rate must be between 0 and 1")
    return round(price_cents * (1 - discount_rate))
```

### 文件注释

文件顶部首先要注释该版本的整体作用、主要类和函数、重要依赖和配置项。
其次记录每次更新迭代的内容，版本号+日期+更新内容。


## 质量检测与审查

异常处理与日志、测试规范、代码异味识别和审查清单已拆到 `references/python-quality-review.md`。

只有用户明确要求检测、审查、测试、查异常处理、查日志或查代码异味时，才读取该文件；普通编码任务不要主动执行检测式审查。

**记住**：Python 代码可以简洁，但不能含糊。好的 Python 代码应当让读者快速看懂数据从哪里来、经过什么处理、失败时会发生什么。
