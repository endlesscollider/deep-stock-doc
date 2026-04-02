# 模块概述

## 模块结构

```
src/
├── stock_db_system/     # 股票数据库模块
├── etf_db_system/       # ETF 数据库模块
├── news_db_system/      # 新闻数据库模块
├── train/               # 模型训练模块
│   ├── deep_stock/      # 深度学习股票预测
│   ├── news_etf/        # 新闻驱动 ETF 预测
│   ├── rl_stock/        # 强化学习股票交易
│   └── rl_etf/          # 强化学习 ETF 交易
└── predict/             # 预测模块
```

## 数据模块

### stock_db_system - 股票数据库

**功能：** A股股票数据获取与存储

**数据内容：**

| 数据类型 | 说明 |
|---------|------|
| 股票基本信息 | 股票代码、名称、行业、上市日期等 |
| 日线行情 | 开高低收、成交量、涨跌幅等 |
| 每日指标 | PE、PB、PS、总市值、流通市值等 |
| 财务指标 | ROE、EPS、BPS 等财务数据 |

**数据源：** Tushare Pro

**数据库：** SQLite (`data/stock_data.db`)

**详细文档：** [股票数据库](stock_db.md)

---

### etf_db_system - ETF 数据库

**功能：** ETF 基金数据获取与存储

**数据内容：**

| 数据类型 | 说明 |
|---------|------|
| ETF 基本信息 | 基金名称、管理人、托管人、基准等 |
| ETF 日线行情 | 开高低收、成交量、涨跌幅等 |

**数据源：** Tushare Pro

**数据库：** SQLite (`data/etf_data.db`)

**详细文档：** [ETF数据库](etf_db.md)

---

### news_db_system - 新闻数据库

**功能：** 财经新闻数据获取与存储

**数据内容：**

| 数据表 | 数据来源 | 特点 | 用途 |
|-------|---------|------|------|
| news_cctv | 央视新闻联播 | 支持历史回补（2013年至今） | 宏观政策分析 |
| news_economic | 百度股市通 | 支持历史回补 | 经济数据追踪 |
| news_stock | 东方财富 | 每日增量存储 | 个股/ETF新闻 |
| news_report | 百度股市通 | 支持历史回补 | 财报发行时间 |

**数据库：** SQLite (`data/news_data.db`)

**详细文档：** [新闻数据库](news_db.md)

---

## 训练模块

### train/deep_stock - 深度学习股票预测

**功能：** 使用深度学习模型预测股票收益

**支持模型：**

- LSTM
- GRU
- Transformer
- CNN1D
- LSTM + Attention

**特性：**

- 多股票联合训练
- 分组特征归一化
- 行业嵌入
- 多种损失函数支持

**详细文档：** [深度学习训练](../train/deep_stock.md)

---

### train/rl_stock - 强化学习股票交易

**功能：** 基于强化学习的投资组合交易系统

**核心特性：**

- **算法：** PPO (Proximal Policy Optimization)
- **股票池：** 沪深300、中证500、中证1000
- **交易环境：** 自定义投资组合交易环境
- **奖励设计：** 夏普比率 + 收益率组合

**关键组件：**

| 组件 | 文件 | 说明 |
|------|------|------|
| 配置管理 | config.py | RlStockConfig 配置类 |
| 交易环境 | environment.py | PortfolioTradingEnv |
| 数据预处理 | preprocessor.py | RLPreprocessor |
| 奖励函数 | reward.py | simple/sharpe 奖励 |
| 训练脚本 | train_rl_stock.py | 训练入口 |

**详细文档：** [RL股票交易](../rl/rl_training_overview.md)

---

### train/rl_etf - 强化学习 ETF 交易

**功能：** 基于强化学习的 ETF 投资组合交易

**特性：**

- ETF 特定的交易环境
- 自定义特征工程
- ETF 奖励函数设计

**详细文档：** [RL ETF交易](../train/rl_etf.md)

---

### train/news_etf - 新闻驱动 ETF 预测

**功能：** 使用新闻数据预测 ETF 收益

**特性：**

- 文本编码（TF-IDF、BERT、BGE-M3）
- 新闻情感分析
- 新闻特征与行情特征融合

**详细文档：** [新闻ETF预测](../train/news_etf.md)

---

## 预测模块

### predict - 预测和回测

**功能：** 模型预测和策略回测

**组件：**

| 组件 | 说明 |
|------|------|
| predictor.py | 模型预测器，加载模型进行预测 |
| backtest.py | 回测引擎，模拟历史交易 |
| strategies.py | 交易策略定义 |

**特性：**

- 加载训练好的模型进行预测
- 完整的回测系统
- 多种交易策略支持
- 详细的回测报告和可视化

**详细文档：** [预测模块](../predict/overview.md)

---

## 模块依赖关系

```
┌─────────────────────────────────────────────────┐
│                   predict/                       │
│              (预测 & 回测)                       │
└─────────────────────────────────────────────────┘
                      ↑
                      │
┌─────────────────────────────────────────────────┐
│                   train/                         │
│    (deep_stock, rl_stock, rl_etf, news_etf)    │
└─────────────────────────────────────────────────┘
                      ↑
                      │
┌─────────┬───────────┴──────────┬───────────────┐
│ stock_  │    etf_db_           │   news_db_    │
│ db_     │    system            │   system      │
│ system  │                      │               │
└─────────┴──────────────────────┴───────────────┘
                      ↑
                      │
              ┌───────┴────────┐
              │  Tushare /     │
              │  Akshare /     │
              │  爬虫           │
              └────────────────┘
```

## 使用流程

1. **数据准备**
   - 初始化数据库
   - 更新/回补数据

2. **模型训练**
   - 选择训练模块（deep_stock / rl_stock / news_etf）
   - 配置训练参数
   - 启动训练

3. **预测/回测**
   - 加载训练好的模型
   - 进行预测或回测
   - 分析结果

## 快速命令参考

```bash
# 数据更新
python -m src.stock_db_system.update_data     # 更新股票数据
python -m src.etf_db_system.update_data       # 更新 ETF 数据
python -m src.news_db_system.update_data      # 更新新闻数据

# RL 训练
python -m src.train.rl_stock.train_rl_stock   # 训练 RL 股票模型

# 深度学习训练
python -m src.train.deep_stock.train          # 训练深度学习模型

# 监控
tensorboard --logdir runs/                     # 查看训练监控
```
