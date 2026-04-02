# 强化学习 ETF 交易 (train/rl_etf)

## 模块概述

强化学习 ETF 交易模块使用 PPO 算法训练 ETF 投资组合管理策略。

**核心功能：**

- 🤖 PPO 算法
- 📊 ETF 特征工程
- 💰 自定义奖励函数
- 📈 回测评估

---

## 模块结构

```
train/rl_etf/
├── train_rl_etf.py       # 训练脚本
├── config.py             # 配置定义
├── environment.py        # 交易环境
├── preprocessor.py       # 数据预处理
├── rewards.py            # 奖励函数
├── callbacks.py          # 训练回调
├── data.py               # 数据获取
├── features/             # 特征工程
└── tests/                # 测试用例
```

---

## 与 rl_stock 的区别

| 方面 | rl_stock | rl_etf |
|------|----------|--------|
| 交易标的 | 股票 | ETF |
| 股票池 | hs300/zz500/zz1000 | 自定义 ETF 列表 |
| 特征 | 财务指标 | 无财务指标 |
| 数据频率 | 日频 | 日频 |
| 交易成本 | 0.01% | 0.01% |

---

## 快速开始

### 1. 训练模型

```bash
# 使用默认 ETF 列表
python -m src.train.rl_etf.train_rl_etf

# 自定义参数
python -m src.train.rl_etf.train_rl_etf \
    --total-timesteps 1000000 \
    --learning-rate 3e-4 \
    --n-envs 4
```

### 2. Python API

```python
from src.train.rl_etf.config import RlEtfConfig
from src.train.rl_etf.train_rl_etf import train

config = RlEtfConfig(
    total_timesteps=1000000,
    n_envs=4,
    learning_rate=3e-4
)

model = train(config)
```

---

## 配置参数

```python
@dataclass
class RlEtfConfig:
    # ETF 列表
    etf_codes: List[str] = None  # None 使用默认宽基 ETF
    
    # 环境参数
    initial_balance: float = 1000000.0
    transaction_cost: float = 0.0001
    lookback: int = 30
    
    # 训练参数
    total_timesteps: int = 5000000
    learning_rate: float = 3e-4
    batch_size: int = 4096
    n_envs: int = 4
```

---

## 奖励函数

### 默认奖励

```python
# 收益率 + 夏普比率
reward = portfolio_return * scaling + sharpe_bonus
```

### 自定义奖励

```python
from src.train.rl_etf.rewards import create_reward_function

reward_fn = create_reward_function(
    reward_type="sharpe",
    scaling=10.0,
    clip_range=5.0
)
```

---

## 特征工程

### ETF 特征

| 特征 | 说明 |
|------|------|
| close | 收盘价 |
| vol | 成交量 |
| amount | 成交额 |
| pct_chg | 涨跌幅 |
| turnover_rate | 换手率（如可用）|

### 技术指标（可选）

- MA5、MA10、MA20
- MACD
- RSI
- 布林带

---

## 输出文件

```
runs/rl_etf/rl_etf_xxx/
├── config.json           # 训练配置
├── final_model.zip       # 最终模型
├── vec_normalize.pkl     # 归一化统计
├── preprocessor.pkl      # 预处理器
├── best_model/           # 最佳模型
└── logs/                 # TensorBoard 日志
```

---

## 相关文档

- [RL股票交易](../rl/rl_training_overview.md)
- [ETF数据库](../modules/etf_db.md)
- [预测模块](../predict/overview.md)
