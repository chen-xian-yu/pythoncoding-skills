# 神经网络代码规范

当任务涉及 PyTorch、TensorFlow、JAX、训练循环、模型结构、数据集、张量形状、checkpoint、实验日志或可复现性时，使用本文件。

## 核心原则

- 训练代码必须能复现：随机种子、配置、数据版本、模型权重和日志要能追踪
- 模型结构、数据处理、训练循环、评估逻辑要分层清楚
- 张量形状、设备、dtype 和 batch 维度要显式可理解
- 训练脚本不要把所有逻辑写在一个文件里
- 优先写能稳定训练、方便排错的代码，再追求性能优化

## 推荐目录结构

```text
src/
├── data/
│   ├── datasets.py
│   ├── transforms.py
│   └── datamodule.py
├── models/
│   ├── modules.py
│   └── classifier.py
├── training/
│   ├── train_loop.py
│   ├── evaluate.py
│   ├── losses.py
│   └── metrics.py
├── config.py
└── utils/
    ├── checkpoint.py
    ├── seed.py
    └── logging.py
tests/
```

小项目可以减少层级，但仍要保持数据、模型、训练、评估职责分离。

## 命名规范

### 张量命名

张量变量名要表达内容和形状含义。常见缩写可以使用，但不要让读者猜。

```python
# 通过：语义明确
input_ids = batch["input_ids"]
attention_mask = batch["attention_mask"]
image_batch = batch["images"]
target_labels = batch["labels"]
logits = model(input_ids=input_ids, attention_mask=attention_mask)
loss = loss_fn(logits, target_labels)

# 可以接受：深度学习中常见的局部变量
x = self.backbone(images)
x = self.pooling(x)
logits = self.classifier(x)

# 不通过：无法看出含义
a = batch["input_ids"]
b = model(a)
c = criterion(b, y)
```

### 形状说明

复杂张量处理必须在注释或变量名中说明关键维度。

```python
# images: [batch_size, channels, height, width]
# logits: [batch_size, num_classes]
logits = model(images)

# token_embeddings: [batch_size, sequence_length, hidden_size]
token_embeddings = encoder(input_ids, attention_mask)
```

## 模型定义

模型类只负责网络结构和 forward 计算，不要在模型里写训练循环、读取文件或保存 checkpoint。

```python
import torch
from torch import nn


class ImageClassifier(nn.Module):
    def __init__(self, backbone: nn.Module, hidden_size: int, num_classes: int) -> None:
        super().__init__()
        self.backbone = backbone
        self.classifier = nn.Linear(hidden_size, num_classes)

    def forward(self, images: torch.Tensor) -> torch.Tensor:
        features = self.backbone(images)
        return self.classifier(features)
```

避免在 `forward` 中制造隐式副作用。

```python
# 不通过：forward 中保存文件，副作用太重
def forward(self, images: torch.Tensor) -> torch.Tensor:
    logits = self.classifier(self.backbone(images))
    torch.save(logits, "debug.pt")
    return logits
```

## Dataset 与 DataLoader

`Dataset` 负责读取和返回单条样本，`DataLoader` 负责 batch、shuffle 和多进程加载。不要让 `Dataset` 承担训练状态管理。

```python
from pathlib import Path
from torch.utils.data import Dataset


class ImageDataset(Dataset):
    def __init__(self, image_paths: list[Path], labels: list[int], transform=None) -> None:
        if len(image_paths) != len(labels):
            raise ValueError("image_paths and labels must have the same length")
        self.image_paths = image_paths
        self.labels = labels
        self.transform = transform

    def __len__(self) -> int:
        return len(self.image_paths)

    def __getitem__(self, index: int) -> dict:
        image = load_image(self.image_paths[index])
        if self.transform is not None:
            image = self.transform(image)
        return {"image": image, "label": self.labels[index]}
```

## 训练循环

训练循环要显式包含：模式切换、设备迁移、清梯度、前向、loss、反向、梯度裁剪、优化器更新、日志。

```python
import torch
from torch import nn
from torch.utils.data import DataLoader


def train_one_epoch(
    model: nn.Module,
    dataloader: DataLoader,
    optimizer: torch.optim.Optimizer,
    loss_fn: nn.Module,
    device: torch.device,
    max_grad_norm: float | None = None,
) -> float:
    model.train()
    total_loss = 0.0

    for batch in dataloader:
        images = batch["image"].to(device)
        labels = batch["label"].to(device)

        optimizer.zero_grad(set_to_none=True)
        logits = model(images)
        loss = loss_fn(logits, labels)
        loss.backward()

        if max_grad_norm is not None:
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_grad_norm)

        optimizer.step()
        total_loss += loss.item()

    return total_loss / len(dataloader)
```

## 验证与推理

验证和推理必须使用 `model.eval()`，并关闭梯度计算。

```python
@torch.no_grad()
def evaluate(
    model: nn.Module,
    dataloader: DataLoader,
    loss_fn: nn.Module,
    device: torch.device,
) -> dict[str, float]:
    model.eval()
    total_loss = 0.0
    correct_count = 0
    sample_count = 0

    for batch in dataloader:
        images = batch["image"].to(device)
        labels = batch["label"].to(device)

        logits = model(images)
        loss = loss_fn(logits, labels)
        predictions = logits.argmax(dim=-1)

        total_loss += loss.item()
        correct_count += (predictions == labels).sum().item()
        sample_count += labels.numel()

    return {
        "loss": total_loss / len(dataloader),
        "accuracy": correct_count / sample_count,
    }
```

## 设备与 dtype 管理

- 不要在代码各处散落 `"cuda"` 字符串，统一由配置或入口决定设备
- 输入张量、模型和 loss 相关张量必须在同一个 device
- 混合精度训练要集中管理，不要局部随意转换 dtype

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model.to(device)

for batch in dataloader:
    images = batch["image"].to(device)
    labels = batch["label"].to(device)
```

## 配置管理

训练参数必须可记录、可复现。不要把关键参数散落在代码里。

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class TrainConfig:
    learning_rate: float
    batch_size: int
    num_epochs: int
    seed: int
    output_dir: str
    max_grad_norm: float | None = None
```

配置至少应覆盖：
- 数据路径和数据版本
- batch size、epoch、学习率、优化器、scheduler
- 模型名称、隐藏层大小、类别数
- 随机种子、输出目录、checkpoint 策略

## 可复现性

训练入口必须设置随机种子，并记录环境和关键配置。

```python
import random
import numpy as np
import torch


def set_seed(seed: int) -> None:
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
```

完全确定性可能牺牲性能。需要严格复现时，再启用确定性算法，并在说明中写清楚代价。

```python
torch.use_deterministic_algorithms(True)
```

## Checkpoint

checkpoint 必须保存足够恢复训练的信息，而不只是模型权重。

```python
def save_checkpoint(
    path: str,
    model: nn.Module,
    optimizer: torch.optim.Optimizer,
    epoch: int,
    metrics: dict[str, float],
) -> None:
    torch.save(
        {
            "model_state_dict": model.state_dict(),
            "optimizer_state_dict": optimizer.state_dict(),
            "epoch": epoch,
            "metrics": metrics,
        },
        path,
    )
```

加载 checkpoint 时，要明确处理 device 映射。

```python
checkpoint = torch.load(checkpoint_path, map_location=device)
model.load_state_dict(checkpoint["model_state_dict"])
```

## 实验日志

训练过程至少记录：
- train loss、validation loss
- 主要业务指标，例如 accuracy、F1、AUC、BLEU 等
- 学习率
- epoch、step、耗时
- checkpoint 路径
- 配置快照

日志里不要只输出“训练完成”。要让后续的人能判断训练是否正常。

## 性能与内存

优先保证正确性，再做性能优化。优化前先定位瓶颈。

常见检查项：
- DataLoader 是否需要设置 `num_workers`、`pin_memory`
- 是否在验证阶段忘记 `torch.no_grad()`
- 是否在循环里保留了不必要的计算图
- 是否频繁把张量在 CPU 和 GPU 之间来回搬运
- 是否在每个 step 都做昂贵的同步或保存操作

```python
# 不通过：把带计算图的 loss 长期保存，可能导致显存增长
losses.append(loss)

# 通过：只保存普通数值
losses.append(loss.item())
```

## 测试神经网络代码

不要只依赖完整训练来验证代码。应增加快速单元测试：
- Dataset 返回字段和类型是否正确
- collate 后 batch 形状是否正确
- 模型 forward 能否跑通
- loss 是否是标量
- 一个极小 batch 能否完成一次训练 step
- checkpoint 能否保存和加载

```python
def test_model_forward_returns_logits_with_expected_shape() -> None:
    model = ImageClassifier(backbone=FakeBackbone(), hidden_size=32, num_classes=10)
    images = torch.randn(4, 3, 224, 224)

    logits = model(images)

    assert logits.shape == (4, 10)
```

## 审查清单

审查神经网络代码时，优先检查：
- 数据、模型、训练、验证、配置是否职责分离
- 张量名称和形状是否清楚
- 训练和验证是否正确切换 `train()` / `eval()`
- 验证和推理是否关闭梯度
- device 和 dtype 是否统一管理
- 随机种子、配置、checkpoint 和日志是否支持复现
- loss、metrics、scheduler、gradient clipping 是否位置正确
- 是否有最小单元测试覆盖 forward、batch shape 和单步训练

**记住**：神经网络代码最怕“能跑但不可复现、能训但不可解释、出错但不知道错在哪里”。规范的目标是让训练过程可追踪、可复现、可调试。
