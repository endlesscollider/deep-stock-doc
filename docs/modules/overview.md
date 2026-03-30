# 模块概述

## 模块结构

```
src/
├── stock_db_system/   # 股票数据库
├── etf_db_system/     # ETF数据库
├── news_db_system/    # 新闻数据库
├── train/             # 模型训练
│   ├── news_etf/      # 新闻驱动ETF预测
│   └── rl_stock/      # 强化学习股票交易
└── predict/           # 预测模块
```

## 模块说明

| 模块 | 功能 |
|------|------|
| stock_db_system | A股数据获取存储 |
| etf_db_system | ETF数据获取存储 |
| news_db_system | 财经新闻获取存储 |
| train/news_etf | 新闻驱动ETF预测 |
| train/rl_stock | 强化学习组合交易 |
| predict | 股票收益预测 |
