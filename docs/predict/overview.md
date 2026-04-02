# 预测模块 (predict)

## 模块概述

预测模块负责加载训练好的模型，进行股票收益预测和策略回测。

**核心功能：**

- 🎯 模型预测
- 📊 策略回测
- 📈 收益分析
- 📉 风险评估

---

## 模块结构

```
predict/
├── predictor.py          # 模型预测器
├── backtest.py           # 回测引擎
├── strategies.py         # 交易策略
└── example_backtest.py   # 回测示例
```

---

## 模型预测器

### 加载模型

```python
from src.predict.predictor import StockPredictor

predictor = StockPredictor(
    model_path="runs/deep_stock/experiment_xxx/model.pt",
    scaler_path="runs/deep_stock/experiment_xxx/scaler.pkl",
    device="cuda"  # 或 "cpu"
)
```

### 单只股票预测

```python
# 预测单只股票
result = predictor.predict(
    ts_code="000001.SZ",
    trade_date="20241231"
)

print(f"预测收益: {result.predicted_return:.4f}")
print(f"置信区间: [{result.lower_bound:.4f}, {result.upper_bound:.4f}]")
```

### 批量预测

```python
# 批量预测多只股票
results = predictor.predict_batch(
    ts_codes=["000001.SZ", "000002.SZ", "600000.SH"],
    trade_date="20241231"
)

for r in results:
    print(f"{r.ts_code}: {r.predicted_return:.4f}")
```

### 概率预测

```python
# 获取预测分布参数
result = predictor.predict_with_uncertainty(
    ts_code="000001.SZ",
    trade_date="20241231"
)

print(f"均值: {result.quantile_50:.4f}")
print(f"10%分位: {result.quantile_10:.4f}")
print(f"90%分位: {result.quantile_90:.4f}")
```

---

## 回测引擎

### 回测配置

```python
from src.predict.backtest import BacktestEngine, BacktestConfig

config = BacktestConfig(
    initial_capital=1000000.0,  # 初始资金
    commission_rate=0.0003,     # 手续费率
    slippage=0.0001,            # 滑点
    start_date="20240101",      # 开始日期
    end_date="20241231"         # 结束日期
)
```

### 运行回测

```python
from src.predict.backtest import BacktestEngine
from src.predict.strategies import TopNStrategy

# 创建回测引擎
engine = BacktestEngine(config)

# 创建策略
strategy = TopNStrategy(
    predictor=predictor,
    top_n=10,
    rebalance_freq=5  # 每5天调仓
)

# 运行回测
result = engine.run(strategy)

# 打印结果
print(result.summary())
```

### 回测结果

```python
@dataclass
class BacktestResult:
    strategy_name: str
    start_date: str
    end_date: str
    total_return: float      # 总收益率
    annual_return: float     # 年化收益率
    sharpe_ratio: float      # 夏普比率
    max_drawdown: float      # 最大回撤
    win_rate: float          # 胜率
    total_trades: int        # 总交易次数
    positions: List[DailyPosition]
    trades: List[Trade]
```

---

## 交易策略

### TopN 策略

选择预测收益最高的 N 只股票：

```python
from src.predict.strategies import TopNStrategy

strategy = TopNStrategy(
    predictor=predictor,
    top_n=10,              # 选择前10只
    rebalance_freq=5,      # 5天调仓一次
    min_score=0.0          # 最低预测收益阈值
)
```

### 组合分配策略

根据预测收益分配权重：

```python
from src.predict.strategies import PortfolioAllocation

strategy = PortfolioAllocation(
    predictor=predictor,
    allocation_method="equal",  # equal / softmax / risk_parity
    max_position=0.1,           # 单只股票最大仓位
    rebalance_freq=5
)
```

### 自定义策略

```python
from src.predict.strategies import TradingStrategy

class MyStrategy(TradingStrategy):
    def get_trading_signals(self, date, positions, data):
        # 实现自定义信号逻辑
        signals = {}
        
        for ts_code in data['ts_code'].unique():
            prediction = self.predictor.predict(ts_code, date)
            
            if prediction.predicted_return > 0.02:
                signals[ts_code] = 'buy'
            elif prediction.predicted_return < -0.02:
                signals[ts_code] = 'sell'
        
        return signals
```

---

## 回测报告

### 文本报告

```python
print(result.summary())

# 输出示例：
# ============================================================
# 回测结果: TopN策略
# ============================================================
# 回测区间: 20240101 ~ 20241231
# 总收益率: 15.23%
# 年化收益率: 15.23%
# 夏普比率: 1.234
# 最大回撤: -8.45%
# 胜率: 55.67%
# 总交易次数: 120
# ============================================================
```

### 可视化报告

```python
# 绘制净值曲线
result.plot_equity_curve(
    save_path="backtest_result.png",
    show=True
)

# 绘制回撤曲线
result.plot_drawdown(
    save_path="drawdown.png"
)

# 绘制持仓分布
result.plot_position_distribution(
    save_path="positions.png"
)
```

---

## 性能指标

### 收益指标

| 指标 | 说明 |
|------|------|
| total_return | 总收益率 |
| annual_return | 年化收益率 |
| monthly_return | 月度收益率 |

### 风险指标

| 指标 | 说明 |
|------|------|
| max_drawdown | 最大回撤 |
| volatility | 波动率 |
| var_95 | 95% VaR |
| cvar_95 | 95% CVaR |

### 风险调整收益

| 指标 | 说明 |
|------|------|
| sharpe_ratio | 夏普比率 |
| sortino_ratio | 索提诺比率 |
| calmar_ratio | 卡玛比率 |

### 交易指标

| 指标 | 说明 |
|------|------|
| win_rate | 胜率 |
| profit_factor | 盈亏比 |
| avg_holding_days | 平均持仓天数 |

---

## 使用示例

### 完整回测流程

```python
from src.predict.predictor import StockPredictor
from src.predict.backtest import BacktestEngine, BacktestConfig
from src.predict.strategies import TopNStrategy

# 1. 加载模型
predictor = StockPredictor(
    model_path="models/best_model.pt",
    scaler_path="models/scaler.pkl"
)

# 2. 配置回测
config = BacktestConfig(
    initial_capital=1000000,
    start_date="20240101",
    end_date="20241231"
)

# 3. 创建策略
strategy = TopNStrategy(
    predictor=predictor,
    top_n=10,
    rebalance_freq=5
)

# 4. 运行回测
engine = BacktestEngine(config)
result = engine.run(strategy)

# 5. 分析结果
print(result.summary())
result.plot_equity_curve()

# 6. 保存结果
result.save("backtest_results/")
```

---

## 相关文档

- [深度学习训练](../train/deep_stock.md)
- [RL股票交易](../rl/rl_training_overview.md)
- [股票数据库](../modules/stock_db.md)
