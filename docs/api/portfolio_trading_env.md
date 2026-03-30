# PortfolioTradingEnv API

投资组合交易环境，用于训练强化学习智能体。

## 概述

基于 Gymnasium 接口的股票投资组合管理环境。

**特性：**
- 无前视偏差
- 交易成本模拟
- 交易阈值过滤
- 夏普比率奖励

---

## 初始化

```python
from src.train.rl_stock.environment import PortfolioTradingEnv
from src.train.rl_stock.config import RlStockConfig

config = RlStockConfig(
    stock_pool="hs300",
    initial_balance=1000000.0,
    transaction_cost=0.001,
    lookback=30,
)

env = PortfolioTradingEnv(df=df, config=config)
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `df` | pd.DataFrame | 股票数据 |
| `config` | RlStockConfig | 环境配置 |
| `preprocessor` | RLPreprocessor | 预处理器(可选) |
| `mode` | str | train/val 模式 |

---

## 属性

- `action_space`: Box(low=0, high=1, shape=(n_stocks+1,))
- `observation_space`: Box(lookback, n_stocks, n_features)
- `portfolio_value`: 当前组合价值
- `weights`: 当前持仓权重

---

## 方法

### reset()

重置环境。

```python
obs, info = env.reset()
```

### step(action)

执行一步。

```python
obs, reward, terminated, truncated, info = env.step(action)
```

### render()

渲染状态。

```python
env.render()
```

### close()

关闭环境。

```python
env.close()
```

---

## 使用示例

```python
env = PortfolioTradingEnv(df=df, config=config)
obs, info = env.reset()

done = False
while not done:
    action = env.action_space.sample()
    obs, reward, terminated, truncated, info = env.step(action)
    done = terminated or truncated

env.close()
```
