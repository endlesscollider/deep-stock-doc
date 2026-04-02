# 深度学习股票预测 (train/deep_stock)

## 模块概述

深度学习股票预测模块使用神经网络模型预测股票未来收益。

**核心功能：**

- 🧠 多种神经网络架构（LSTM、GRU、Transformer、CNN1D）
- 📊 分组特征归一化
- 🏭 行业嵌入支持
- 📈 多步预测
- 🎯 概率预测（高斯分布）

---

## 模块结构

```
train/deep_stock/
├── train.py              # 训练脚本
├── dataset.py            # 数据集定义
├── preprocessor.py       # 数据预处理
├── model_factory.py      # 模型工厂
├── model_composer.py     # 模型组合器
├── training_utils.py     # 训练工具函数
├── layers/               # 自定义层
│   ├── hybrid_loss.py    # 混合损失函数
│   ├── output_layer.py   # 输出层
│   └── industry_embedding.py  # 行业嵌入
└── models/               # 模型定义
```

---

## 支持的模型

| 模型 | 说明 | 适用场景 |
|------|------|---------|
| lstm | LSTM 网络 | 时间序列预测 |
| gru | GRU 网络 | 更快的 RNN |
| transformer | Transformer | 长序列建模 |
| cnn1d | 1D CNN | 局部模式识别 |
| lstm_attention | LSTM + Attention | 可解释性预测 |

---

## 训练配置

### TrainConfig 参数

```python
@dataclass
class TrainConfig:
    # 数据配置
    sample_method: str = "zz1000"  # 采样方法
    start_date: str = None         # 开始日期
    end_date: str = None           # 结束日期
    
    # 特征配置
    feature_cols: List[str] = None  # 自定义特征列
    target_col: str = "close"       # 目标列
    
    # 预处理配置
    normalize_method: str = "grouped"  # 归一化方法
    handle_missing: str = "ffill"      # 缺失值处理
    
    # 数据集配置
    window_size: int = 60     # 回看窗口
    predict_days: int = 7     # 预测天数
    return_kind: str = "simple"  # 收益类型
    
    # 模型配置
    model_type: str = "lstm"  # 模型类型
    hidden_size: int = 256    # 隐藏层大小
    num_layers: int = 2       # 层数
    dropout: float = 0.2      # Dropout
    
    # 训练配置
    batch_size: int = 512     # 批大小
    learning_rate: float = 1e-3  # 学习率
    num_epochs: int = 100     # 训练轮数
```

---

## 快速开始

### 1. 训练模型

```bash
# 使用默认配置训练
python -m src.train.deep_stock.train

# 使用特定股票池
python -m src.train.deep_stock.train --sample-method hs300

# 自定义参数
python -m src.train.deep_stock.train \
    --model-type lstm \
    --window-size 60 \
    --predict-days 7 \
    --batch-size 512 \
    --num-epochs 100
```

### 2. Python API

```python
from src.train.deep_stock.train import TrainConfig, train_model

# 创建配置
config = TrainConfig(
    sample_method="hs300",
    model_type="lstm",
    window_size=60,
    predict_days=7,
    num_epochs=100
)

# 训练模型
model, scaler = train_model(config)
```

---

## 数据预处理

### 分组归一化

不同类型的特征使用不同的归一化方法：

| 特征组 | 归一化方法 | 说明 |
|--------|-----------|------|
| price | standard | 价格特征 |
| volume | robust | 成交量（对异常值鲁棒）|
| market_value | robust | 市值特征 |
| ratio | robust | 比率类特征 |
| fina | robust | 财务指标 |

```python
from src.train.deep_stock.preprocessor import StockPreprocessor

preprocessor = StockPreprocessor(
    normalize_method="grouped",
    handle_missing="ffill"
)

preprocessor.fit(train_df)
train_dataset = preprocessor.transform(train_df)
```

### 数据集构建

```python
from src.train.deep_stock.dataset import StockDLDataset

dataset = StockDLDataset(
    df=train_df,
    window_size=60,
    predict_days=7,
    preprocessor=preprocessor
)

# 输出格式
# X: (batch, window_size, n_features)
# y: (batch,) 或 (batch, predict_days) 多步预测
```

---

## 模型架构

### LSTM 模型

```
输入 (batch, window_size, n_features)
    ↓
LSTM 层 × num_layers
    ↓
Dropout
    ↓
全连接层
    ↓
输出 (batch, 1) 或 (batch, predict_days)
```

### Transformer 模型

```
输入 (batch, window_size, n_features)
    ↓
位置编码
    ↓
Transformer Encoder 层
    ↓
全局池化
    ↓
全连接层
    ↓
输出
```

---

## 损失函数

### 混合损失 (HybridLoss)

组合多种损失函数：

```python
from src.train.deep_stock.layers.hybrid_loss import HybridLoss

loss_fn = HybridLoss(
    mse_weight=1.0,      # MSE 损失权重
    mae_weight=0.5,      # MAE 损失权重
    huber_weight=0.3,    # Huber 损失权重
    direction_weight=0.2 # 方向准确率权重
)
```

### 概率预测

使用高斯分布输出：

```python
# 模型输出均值和方差
mu, sigma = model(x)

# 使用 NLL 损失
loss = -torch.distributions.Normal(mu, sigma).log_prob(y).mean()
```

---

## 训练监控

### TensorBoard

```bash
tensorboard --logdir runs/deep_stock/
```

**监控指标：**

- `train/loss` - 训练损失
- `train/mse` - MSE 损失
- `val/loss` - 验证损失
- `val/mse` - 验证 MSE
- `val/direction_acc` - 方向准确率

### 学习率调度

```python
# 余弦退火
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
    optimizer,
    T_max=num_epochs,
    eta_min=1e-6
)
```

---

## 输出文件

```
runs/deep_stock/experiment_xxx/
├── model.pt              # 模型权重
├── model_config.json     # 模型配置
├── scaler.pkl            # 归一化参数
├── train_config.json     # 训练配置
├── logs/                 # TensorBoard 日志
│   └── events.out.tfevents.*
└── checkpoints/          # 训练检查点
    └── checkpoint_epoch_*.pt
```

---

## 高级用法

### 多步预测

```python
config = TrainConfig(
    multi_step=True,
    predict_days=7  # 预测未来7天每天
)

# 输出形状: (batch, 7)
```

### 行业嵌入

```python
from src.train.deep_stock.layers.industry_embedding import IndustryEmbedding

# 为不同行业学习独立嵌入
embedding = IndustryEmbedding(
    num_industries=30,
    embedding_dim=16
)
```

### 自定义模型

```python
from src.train.deep_stock.model_factory import ModelFactory

# 创建自定义模型
model = ModelFactory.create_model(
    model_type="custom",
    input_size=n_features,
    hidden_size=256,
    num_layers=3,
    dropout=0.3
)
```

---

## 最佳实践

1. **数据质量**：确保数据完整，缺失值处理得当
2. **特征工程**：选择有预测能力的特征
3. **防止过拟合**：使用 Dropout、早停
4. **学习率**：从 1e-3 开始，根据收敛情况调整
5. **批大小**：根据 GPU 内存调整

---

## 相关文档

- [RL股票交易](../rl/rl_training_overview.md)
- [新闻ETF预测](news_etf.md)
- [预测模块](../predict/overview.md)
