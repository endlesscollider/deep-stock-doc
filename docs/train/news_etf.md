# 新闻驱动 ETF 预测 (train/news_etf)

## 模块概述

新闻驱动 ETF 预测模块使用新闻文本数据预测 ETF 未来收益。

**核心功能：**

- 📰 多种文本编码器（TF-IDF、BERT、BGE-M3）
- 📊 新闻特征与行情特征融合
- 🧠 LSTM/Transformer 预测模型
- 📈 多日收益预测

---

## 模块结构

```
train/news_etf/
├── train_news_etf.py     # 训练脚本
├── config.py             # 配置定义
├── dataset.py            # 数据集定义
├── preprocessor.py       # 数据预处理
├── text_encoder.py       # 文本编码器
├── models/               # 模型定义
└── tests/                # 测试用例
```

---

## 文本编码器

### TF-IDF

```python
from src.train.news_etf.text_encoder import TextEncoder

encoder = TextEncoder(
    encoder_type="tfidf",
    max_features=5000
)

vectors = encoder.encode(texts)
```

### BERT

```python
encoder = TextEncoder(
    encoder_type="bert",
    model_name="bert-base-chinese"
)

vectors = encoder.encode(texts)
```

### BGE-M3（推荐）

```python
encoder = TextEncoder(
    encoder_type="bge_m3",
    model_name="BAAI/bge-m3"
)

vectors = encoder.encode(texts)
```

---

## 训练配置

```python
@dataclass
class TrainingConfig:
    # 文本编码配置
    text_encoder_type: str = "tfidf"  # tfidf/bert/bge_m3
    
    # 模型配置
    hidden_size: int = 128
    num_layers: int = 2
    dropout: float = 0.3
    
    # 训练配置
    batch_size: int = 512
    learning_rate: float = 1e-3
    num_epochs: int = 50
    
    # 预测配置
    predict_days: int = 5  # 预测未来N天收益
```

---

## 快速开始

### 1. 准备数据

确保新闻数据已更新：

```bash
# 更新新闻数据
python -m src.news_db_system.update_data --stock

# 更新 ETF 数据
python -m src.etf_db_system.update_data
```

### 2. 训练模型

```bash
# 使用 TF-IDF 编码
python -m src.train.news_etf.train_news_etf --text-encoder tfidf

# 使用 BGE-M3 编码
python -m src.train.news_etf.train_news_etf --text-encoder bge_m3

# 自定义参数
python -m src.train.news_etf.train_news_etf \
    --text-encoder bge_m3 \
    --predict-days 5 \
    --batch-size 256 \
    --num-epochs 100
```

### 3. Python API

```python
from src.train.news_etf.config import TrainingConfig
from src.train.news_etf.train_news_etf import train

config = TrainingConfig(
    text_encoder_type="bge_m3",
    predict_days=5,
    num_epochs=50
)

model, encoder, scaler = train(config)
```

---

## 数据流程

```
新闻文本 (raw)
    ↓
文本编码器 (TF-IDF/BERT/BGE-M3)
    ↓
新闻向量 (dense/sparse)
    ↓
与行情特征拼接
    ↓
预测模型 (LSTM/Transformer)
    ↓
收益预测
```

---

## 模型架构

```
输入:
  - 新闻向量: (batch, text_dim)
  - 行情特征: (batch, window, n_features)

处理流程:
  1. 新闻特征: Dense(128) → ReLU → Dropout
  2. 行情特征: LSTM(window) → 取最后时间步
  3. 特征融合: Concat([news, market])
  4. 预测头: Dense(64) → ReLU → Dense(1)

输出:
  - 预测收益: (batch, 1)
```

---

## 特征工程

### 新闻特征

- 文本编码向量（TF-IDF / BERT / BGE-M3）
- 新闻数量统计
- 新闻情感得分（可选）
- 新闻发布时间特征

### 行情特征

- 收盘价、成交量
- 技术指标（MA、MACD、RSI等）
- 市值、换手率

---

## 训练监控

```bash
tensorboard --logdir runs/news_etf/
```

**监控指标：**

- `train/loss` - 训练损失
- `train/mse` - MSE 损失
- `val/loss` - 验证损失
- `val/direction_acc` - 方向准确率
- `val/ic` - 信息系数

---

## 输出文件

```
runs/news_etf/experiment_xxx/
├── model.pt              # 模型权重
├── text_encoder.pkl      # 文本编码器
├── scaler.pkl            # 归一化参数
├── config.json           # 训练配置
└── logs/                 # TensorBoard 日志
```

---

## 文本编码器对比

| 编码器 | 维度 | 速度 | 效果 | 适用场景 |
|--------|------|------|------|---------|
| TF-IDF | 5000 | 快 | 一般 | 快速实验 |
| BERT | 768 | 慢 | 好 | 中文文本 |
| BGE-M3 | 1024 | 中 | 最好 | 中文金融文本 |

---

## 最佳实践

1. **文本预处理**：清洗噪声、去除停用词
2. **编码器选择**：推荐 BGE-M3
3. **特征对齐**：确保新闻日期与行情日期对齐
4. **防止过拟合**：使用 Dropout、早停
5. **评估指标**：关注方向准确率和 IC

---

## 相关文档

- [新闻数据库](../modules/news_db.md)
- [ETF数据库](../modules/etf_db.md)
- [深度学习训练](deep_stock.md)
