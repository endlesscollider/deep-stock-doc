# 训练模块概述

训练模块包含多个子模块，用于不同场景的模型训练。

## 模块结构

```
src/train/
├── deep_stock/      # 深度学习股票预测
├── news_etf/        # 新闻驱动 ETF 预测
├── rl_stock/        # 强化学习股票交易
└── rl_etf/          # 强化学习 ETF 交易
```

## 子模块说明

| 模块 | 算法 | 用途 | 文档 |
|------|------|------|------|
| deep_stock | LSTM/GRU/Transformer | 股票收益预测 | [深度学习训练](deep_stock.md) |
| news_etf | BERT/TF-IDF + LSTM | 新闻驱动ETF预测 | [新闻ETF预测](news_etf.md) |
| rl_stock | PPO | 股票投资组合交易 | [RL股票交易](../rl/rl_training_overview.md) |
| rl_etf | PPO | ETF投资组合交易 | [RL ETF交易](rl_etf.md) |

## 训练流程概览

### 1. 数据准备

```python
# 从数据库获取数据
from src.stock_db_system import DataManager

manager = DataManager()
df = manager.get_merged_data(
    ts_code=["000001.SZ", "000002.SZ"],
    start_date="20200101",
    end_date="20241231"
)
```

### 2. 模型训练

```bash
# 深度学习训练
python -m src.train.deep_stock.train

# RL 股票训练
python -m src.train.rl_stock.train_rl_stock --stock-pool hs300

# 新闻 ETF 训练
python -m src.train.news_etf.train_news_etf
```

### 3. 监控训练

```bash
# TensorBoard 监控
tensorboard --logdir runs/
```

## 训练输出

所有训练模块的输出统一保存在 `runs/` 目录：

```
runs/
├── deep_stock/           # 深度学习训练输出
│   └── experiment_xxx/
│       ├── model.pt
│       ├── scaler.pkl
│       └── logs/
├── rl_stock/             # RL 股票训练输出
│   └── rl_stock_xxx/
│       ├── final_model.zip
│       ├── vec_normalize.pkl
│       └── logs/
├── rl_etf/               # RL ETF 训练输出
└── news_etf/             # 新闻 ETF 训练输出
```

## 选择指南

| 需求 | 推荐模块 |
|------|---------|
| 预测单只股票收益 | deep_stock |
| 预测 ETF 收益（基于新闻） | news_etf |
| 投资组合管理（股票） | rl_stock |
| 投资组合管理（ETF） | rl_etf |
| 回测交易策略 | rl_stock + predict |
