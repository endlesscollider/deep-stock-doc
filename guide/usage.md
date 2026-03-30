# Deep Stock 使用指南

本文档介绍如何使用 Deep Stock 项目进行股票数据更新、模型训练和预测分析。

## 快速开始

### 1. 安装依赖

```bash
cd deep-stock
pip install -e .
```

### 2. 配置Tushare Token

在 `config.py` 中配置你的 Tushare Token：

```python
TS_TOKEN = "你的_tushare_token"
```

### 3. 更新数据

```bash
python -m src.stock_db_system.update_data
python -m src.etf_db_system.update_data
python -m src.news_db_system.update_data
```

### 4. 训练模型

```bash
python -m src.train.train_stock
```

### 5. 预测

```bash
python examples/01_predict_today.py
```

## 数据更新

### 命令行

```bash
# 股票数据
python -m src.stock_db_system.update_data --start-date 20240101

# ETF数据
python -m src.etf_db_system.update_data

# 新闻数据
python -m src.news_db_system.update_data --backfill
```

## 模型训练

```bash
python -m src.train.train_stock --epochs 50 --batch-size 512
```

## 预测

```bash
python examples/01_predict_today.py
python examples/04_backtest.py --start-date 20250101 --end-date 20250331
```

## 注意事项

1. 股市有风险：预测结果仅供参考
2. 确保数据库中有对应日期的数据
3. 首次使用需要训练模型
