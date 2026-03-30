# Deep Stock 文档

欢迎使用 Deep Stock 文档中心！这里提供项目的完整使用指南、API 参考和模块说明。

## 项目概述

Deep Stock 是一个基于深度学习的股票价格预测系统，包含数据获取、存储、训练和预测功能。

## 主要特性

- **多数据源支持**：股票、ETF、财经新闻数据
- **强化学习交易**：基于 PPO 算法的投资组合管理
- **深度学习预测**：LSTM 等模型进行股票收益预测
- **新闻驱动分析**：央视新闻、经济日历、个股新闻分析
- **可视化面板**：Streamlit 构建的数据可视化应用

## 快速导航

| 分类 | 内容 |
|------|------|
| 入门指南 | [使用指南](guide/usage.md)、[策略说明](guide/strategies.md) |
| 模块文档 | [股票数据库](modules/stock_db.md)、[ETF数据库](modules/etf_db.md)、[新闻数据库](modules/news_db.md) |
| API 参考 | [PortfolioTradingEnv](api/portfolio_trading_env.md) |
| 可视化 | [新闻面板](news_dashboard.md) |

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

### API

- [PortfolioTradingEnv](api/portfolio_trading_env.md) - RL 交易环境 API

## 相关链接

- GitHub 仓库：https://github.com/endlesscollider/deep-stock
- Tushare：https://tushare.pro/
