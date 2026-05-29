# Python 质量检测与审查

只有用户明确要求检测、审查、review、测试、查异常处理、查日志、查代码异味、找问题或质量检查时，才使用本文件。

不要在普通编写、解释、改写 Python 代码时主动套用本文件做检测式审查。

## 异常处理与日志

### 捕获具体异常

```python
from pathlib import Path
import json
import logging


logger = logging.getLogger(__name__)


def load_config(path: Path) -> dict:
    try:
        return json.loads(path.read_text(encoding="utf-8"))
    except FileNotFoundError:
        logger.error("配置文件不存在: %s", path)
        raise
    except json.JSONDecodeError as error:
        raise ValueError(f"配置文件不是合法 JSON: {path}") from error
```

不要裸 `except`，不要吞掉异常后继续执行。

```python
# 不通过：异常被吞掉，调用方不知道失败了
try:
    sync_orders()
except Exception:
    pass
```

### 日志不要泄露敏感信息

日志可以记录上下文，但不要输出密码、token、身份证号、手机号全量等敏感数据。

```python
# 通过：记录可定位问题的上下文
logger.info("订单同步完成: order_id=%s item_count=%s", order_id, item_count)

# 不通过：泄露敏感字段
logger.info("用户登录: phone=%s password=%s", phone, password)
```

## 测试规范

### 使用 AAA 结构

测试可以按 Arrange、Act、Assert 组织：先准备数据，再执行行为，最后断言结果。

```python
def test_calculate_order_total_returns_sum_of_all_items() -> None:
    # Arrange
    items = [
        OrderItem(product_id="sku_1", unit_price_cents=1000, quantity=2),
        OrderItem(product_id="sku_2", unit_price_cents=500, quantity=1),
    ]

    # Act
    total = calculate_order_total(items)

    # Assert
    assert total == 2500
```

### 测试命名

测试名要说明场景和期望结果。

```python
# 通过
def test_find_user_email_returns_none_when_user_missing() -> None:
    ...

def test_load_config_raises_error_for_invalid_json() -> None:
    ...

# 不通过
def test_works() -> None:
    ...

def test_user() -> None:
    ...
```

### 测试覆盖重点

检测测试质量时，优先看：
- 正常路径是否覆盖
- 边界条件是否覆盖
- 失败路径是否覆盖
- 外部依赖是否被隔离或替换
- 测试是否能稳定重复运行

## 代码异味识别

### 函数过长

```python
# 不通过：校验、转换、保存全混在一起
def process_order(payload: dict) -> None:
    ...


# 通过：拆分成独立步骤
def process_order(payload: dict) -> None:
    order = validate_order_payload(payload)
    normalized_order = normalize_order(order)
    save_order(normalized_order)
```

### 参数过多

参数过多通常说明函数承担了太多职责，或缺少配置对象/数据对象。

```python
# 不通过：调用方很难记住每个参数的含义
def export_report(data, path, encoding, delimiter, include_header, retry_count):
    ...


# 通过：把相关参数收成配置对象
def export_report(data, path, options: ExportOptions):
    ...
```

### 嵌套过深

```python
# 不通过：主路径被多层条件包住
if user is not None:
    if user.is_active:
        if order is not None:
            if order.can_be_paid:
                pay_order(order)

# 通过：提前返回
if user is None:
    return
if not user.is_active:
    return
if order is None:
    return
if not order.can_be_paid:
    return

pay_order(order)
```

### 魔法数字和魔法字符串

```python
# 不通过：读者不知道数字含义
if retry_count > 3:
    ...

# 通过：使用命名常量表达含义
MAX_RETRY_COUNT = 3

if retry_count > MAX_RETRY_COUNT:
    ...
```

## 审查清单

审查 Python 代码时，优先检查：
- 命名是否清晰，是否存在无意义缩写或拼音缩写
- 函数是否过长、参数是否过多、嵌套是否过深
- 类型标注是否能帮助调用方理解边界
- 是否存在可变默认参数
- 异常是否被吞掉，日志是否泄露敏感信息
- 测试是否覆盖正常路径、边界条件和失败路径
- 是否能通过 formatter、linter、type checker 和测试

检测输出应优先列出具体问题、文件位置、风险和建议改法；不要泛泛而谈。
