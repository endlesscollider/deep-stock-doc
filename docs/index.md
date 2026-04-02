# Deep Stock 文档

欢迎使用 Deep Stock 文档中心！这里提供项目的完整使用指南、API 参考和模块说明。

## 项目概述

Deep Stock 是一个基于深度学习的股票价格预测系统，包含数据获取、存储、训练和预测功能。

**核心功能：**

- 📊 **多数据源支持**：股票、ETF、财经新闻数据
- 🤖 **强化学习交易**：基于 PPO 算法的投资组合管理
- 🧠 **深度学习预测**：LSTM、Transformer 等模型进行股票收益预测
- 📰 **新闻驱动分析**：央视新闻、经济日历、个股新闻分析
- 📈 **回测引擎**：完整的策略回测和评估系统

## 项目架构

```
deep-stock/
├── src/
│   ├── stock_db_system/     # 股票数据库模块（A股数据）
│   ├── etf_db_system/       # ETF 数据库模块
│   ├── news_db_system/      # 新闻数据库模块
│   ├── train/               # 模型训练模块
│   │   ├── deep_stock/      # 深度学习股票预测
│   │   ├── news_etf/        # 新闻驱动 ETF 预测
│   │   ├── rl_stock/        # 强化学习股票交易
│   │   └── rl_etf/          # 强化学习 ETF 交易
│   └── predict/             # 预测和回测模块
├── data/                    # 数据存储目录
├── models/                  # 模型保存目录
├── runs/                    # 训练运行目录
└── config/                  # 配置文件目录
```

## 快速开始

### 安装依赖

```bash
# 使用 uv（推荐）
uv sync

# 或使用 pip
pip install -e .
```

### 配置 Tushare Token

在 `config.py` 中配置你的 Tushare Token：

```python
TS_TOKEN = "你的_tushare_token"
```

获取 Token：https://tushare.pro/

### 初始化数据库

```python
# 股票数据库
from src.stock_db_system import init_db as init_stock_db
init_stock_db()

# ETF 数据库
from src.etf_db_system import init_db as init_etf_db
init_etf_db()

# 新闻数据库
from src.news_db_system import init_db as init_news_db
init_news_db()
```

### 更新数据

```bash
# 更新股票数据
python -m src.stock_db_system.update_data

# 更新 ETF 数据
python -m src.etf_db_system.update_data

# 更新新闻数据
python -m src.news_db_system.update_data
```

## 主要模块

| 模块 | 功能 | 文档 |
|------|------|------|
| stock_db_system | A股股票数据获取与存储 | [股票数据库](modules/stock_db.md) |
| etf_db_system | ETF 基金数据获取与存储 | [ETF数据库](modules/etf_db.md) |
| news_db_system | 财经新闻数据获取与存储 | [新闻数据库](modules/news_db.md) |
| train/deep_stock | 深度学习股票预测模型 | [深度学习训练](train/deep_stock.md) |
| train/rl_stock | 强化学习股票交易系统 | [RL股票交易](rl/rl_training_overview.md) |
| train/rl_etf | 强化学习 ETF 交易系统 | [RL ETF交易](train/rl_etf.md) |
| train/news_etf | 新闻驱动 ETF 预测 | [新闻ETF预测](train/news_etf.md) |
| predict | 股票预测和回测 | [预测模块](predict/overview.md) |

## 训练示例

### 强化学习股票交易

```bash
# 训练模型
python -m src.train.rl_stock.train_rl_stock --stock-pool hs300 --total-timesteps 1000000

# 恢复训练
python -m src.train.rl_stock.train_rl_stock --resume runs/rl_stock/rl_stock_20260401_120000

# 查看监控
tensorboard --logdir runs/rl_stock/
```

详细文档：[RL Stock 训练指南](rl/rl_training_overview.md)

### 深度学习股票预测

```bash
# 训练模型
python -m src.train.deep_stock.train

# 使用 TensorBoard 监控
tensorboard --logdir runs/
```

详细文档：[深度学习训练指南](train/deep_stock.md)

## 文档目录

### 指南

- [使用指南](guide/usage.md) - 完整使用教程
- [策略说明](guide/strategies.md) - 交易策略详解
- [超参数调优](guide/hyperparameter_tuning.md) - 模型调优指南

### 模块

- [模块概述](modules/overview.md) - 项目模块总览
- [股票数据库](modules/stock_db.md) - 股票数据获取与存储
- [ETF数据库](modules/etf_db.md) - ETF 数据获取与存储
- [新闻数据库](modules/news_db.md) - 新闻数据获取与存储

### RL 训练

- [RL训练概览](rl/rl_training_overview.md) - 强化学习训练总览
- [环境设置详解](rl/rl_environment_setup.md) - 交易环境配置
- [训练框架详解](rl/rl_training_framework.md) - 训练框架说明
- [数据处理详解](rl/rl_data_processing.md) - 数据处理流程

### API 参考

- [PortfolioTradingEnv](api/portfolio_trading_env.md) - RL 交易环境 API

## 技术栈

| 类别 | 技术 |
|------|------|
| 数据处理 | pandas, numpy, SQLAlchemy |
| 数据源 | tushare, akshare |
| 深度学习 | PyTorch, tensorboard |
| 强化学习 | stable-baselines3, gymnasium |
| 数据库 | SQLite |

## 相关链接

- GitHub 仓库：https://github.com/endlesscollider/deep-stock
- Tushare：https://tushare.pro/
- 问题反馈：https://github.com/endlesscollider/deep-stock/issues

## 许可证

MIT License
