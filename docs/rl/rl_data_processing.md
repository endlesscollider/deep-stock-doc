# RL Stock 数据处理文档

本文档详细介绍强化学习股票交易模型的数据处理流程，包括数据加载、预处理、缓存机制和验证策略。

## 目录

1. [架构概览](#1-架构概览)
2. [数据加载机制](#2-数据加载机制)
3. [数据准备管道](#3-数据准备管道)
4. [日期范围过滤](#4-日期范围过滤)
5. [缓存系统](#5-缓存系统)
6. [股票池选择](#6-股票池选择)
7. [数据验证与错误处理](#7-数据验证与错误处理)
8. [最佳实践](#8-最佳实践)

---

## 1. 架构概览

### 1.1 数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           RL 数据处理流程                                │
└─────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐
    │ RlStockConfig │
    │   配置参数    │
    └──────┬───────┘
           │
           ▼
    ┌──────────────────────────────────────────────────────────┐
    │               load_training_dataframe()                   │
    │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
    │  │ use_mock?   │───▶│ cache?      │───▶│ fetch_db?   │  │
    │  │ 模拟数据    │    │ 缓存检查    │    │ 数据库获取   │  │
    │  └─────────────┘    └─────────────┘    └─────────────┘  │
    └──────────────────────────┬───────────────────────────────┘
                               │
                               ▼
    ┌──────────────────────────────────────────────────────────┐
    │                prepare_datasets()                        │
    │  ┌─────────────┐    ┌─────────────┐                     │
    │  │ 日期过滤    │───▶│ 训练/验证分割│                     │
    │  │ train/val   │    │ 空数据处理   │                     │
    │  └─────────────┘    └─────────────┘                     │
    └──────────────────────────┬───────────────────────────────┘
                               │
          ┌────────────────────┴────────────────────┐
          ▼                                         ▼
    ┌─────────────┐                         ┌─────────────┐
    │   训练集     │                         │   验证集     │
    │  train_df   │                         │   eval_df   │
    └──────┬──────┘                         └──────┬──────┘
           │                                       │
           ▼                                       ▼
    ┌─────────────────────────────────────────────────────────┐
    │                    RLPreprocessor                        │
    │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
    │  │ 特征分组    │───▶│ 归一化      │───▶│ 滑动窗口    │ │
    │  │ price/vol   │    │ standard/   │    │ lookback    │ │
    │  │ ratio/fin   │    │ robust      │    │ 构建观测    │ │
    │  └─────────────┘    └─────────────┘    └─────────────┘ │
    └──────────────────────────┬──────────────────────────────┘
                               │
                               ▼
    ┌──────────────────────────────────────────────────────────┐
    │                 PortfolioTradingEnv                       │
    │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
    │  │ 观测空间    │    │ 动作空间    │    │ 奖励计算    │  │
    │  │ (L,S,F)    │    │ (S+1,)     │    │ 夏普比率    │  │
    │  └─────────────┘    └─────────────┘    └─────────────┘  │
    └──────────────────────────────────────────────────────────┘
```

### 1.2 核心模块

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| 配置管理 | `src/train/rl_stock/config.py` | 参数定义与验证 |
| 数据获取 | `src/train/rl_stock/train_rl_stock.py` | 数据库/模拟数据加载 |
| 预处理器 | `src/train/rl_stock/preprocessor.py` | 特征工程与归一化 |
| 交易环境 | `src/train/rl_stock/environment.py` | Gymnasium 环境实现 |
| 数据库管理 | `src/stock_db_system/manager.py` | 数据库查询与合并 |
| 数据源 | `src/stock_db_system/provider.py` | Tushare API 封装 |

---

## 2. 数据加载机制

### 2.1 加载策略选择

数据加载通过 `load_training_dataframe()` 函数实现，支持三种模式：

```python
def load_training_dataframe(config: RlStockConfig) -> pd.DataFrame:
    """
    根据配置加载训练数据。
    
    优先级：模拟数据 > 缓存数据 > 数据库获取
    """
    # 模式1：模拟数据（调试用）
    if config.use_mock_data:
        print("🧪 使用模拟数据加速实验。")
        return create_mock_data(config)
    
    # 模式2：缓存数据
    cache_path = _resolve_cache_path(config)
    if cache_path and os.path.exists(cache_path):
        print(f"♻️ 使用缓存数据: {cache_path}")
        return pd.read_pickle(cache_path)
    
    # 模式3：数据库获取
    df = fetch_stock_data(config)
    
    # 保存缓存
    if cache_path:
        pd.to_pickle(df, cache_path)
        print(f"💾 数据缓存已保存: {cache_path}")
    
    return df
```

### 2.2 模拟数据生成

用于快速调试和测试，无需连接数据库：

```python
def create_mock_data(config: RlStockConfig) -> pd.DataFrame:
    """
    生成模拟股票数据，包含完整特征列。
    
    特点：
    - 5只模拟股票（MOCK000-MOCK004）
    - 至少 60 个交易日
    - 包含 OHLCV、估值指标、财务指标
    - 价格随机游走模拟
    """
    rng = np.random.default_rng(42)
    n_stocks = 5
    n_days = max(config.lookback + 30, 60)
    base_date = pd.to_datetime(config.start_date or "20200101")
    dates = pd.date_range(base_date, periods=n_days, freq="B")
    
    records = []
    for i in range(n_stocks):
        ts_code = f"MOCK{i:03d}.SZ"
        price = 100.0 + rng.normal(0, 1)
        for date in dates:
            # 随机价格变动
            change = rng.normal(0, 1)
            close_p = max(1.0, price + change)
            # ... 生成其他字段
            records.append({...})
    
    return pd.DataFrame.from_records(records)
```

**使用场景：**

| 场景 | 命令参数 | 说明 |
|------|---------|------|
| 快速验证代码 | `--use-mock-data` | 跳过数据库，使用内置模拟数据 |
| CI/CD 测试 | `config.use_mock_data = True` | 无需外部依赖 |
| 算法原型开发 | 同上 | 快速迭代 |

### 2.3 数据库数据获取

从 SQLite 数据库获取真实股票数据：

```python
def fetch_stock_data(config: RlStockConfig) -> pd.DataFrame:
    """从数据库获取股票数据。"""
    print("📊 从数据库获取股票数据...")
    
    # 1. 初始化数据库连接
    from src.stock_db_system.database import init_db as init_stock_db
    from src.stock_db_system.manager import DataManager
    from src.stock_db_system.provider import TushareProvider
    
    init_stock_db()
    manager = DataManager()
    provider = TushareProvider()
    
    # 2. 计算数据日期范围
    date_starts = [d for d in [config.start_date, config.val_start_date] if d]
    date_ends = [d for d in [config.end_date, config.val_end_date] if d]
    data_start = min(date_starts) if date_starts else config.start_date
    data_end = max(date_ends) if date_ends else config.end_date
    
    # 3. 获取股票池成分股
    index_codes = {
        "hs300": "399300.SZ",
        "zz500": "000905.SH",
        "zz1000": "000852.SH",
    }
    index_code = index_codes.get(config.stock_pool, index_codes["hs300"])
    stock_codes = provider.fetch_index_constituents(index_code=index_code)
    
    # 4. 应用股票数量限制（调试用）
    if config.max_stocks and config.max_stocks < len(stock_codes):
        stock_codes = stock_codes[:config.max_stocks]
    
    # 5. 批量获取合并数据
    batch_size = 25 if len(stock_codes) > 200 else 50
    df = manager.get_merged_data(
        ts_code=stock_codes,
        start_date=data_start,
        end_date=data_end,
        batch_size=batch_size,
        show_progress=True,
    )
    
    return df
```

### 2.4 数据合并策略

`DataManager.get_merged_data()` 融合多表数据：

```python
def get_merged_data(self, ts_code, start_date, end_date, batch_size=100):
    """
    超级接口：融合行情 + 每日指标 + 基础信息 + 财务指标
    
    数据来源：
    - daily_prices: 日线行情（OHLCV）
    - daily_basics: 每日指标（PE/PB/PS/市值）
    - stock_profiles: 股票基本信息
    - fina_indicators: 财务指标（ROE/EPS/BPS）
    
    合并策略：
    - 行情 + 指标：LEFT JOIN on (ts_code, trade_date)
    - 基本信息：LEFT JOIN on ts_code
    - 财务指标：ASOF JOIN（向后查找最近公告）
    """
```

**合并流程图：**

```
daily_prices ──┬── LEFT JOIN ──┬── LEFT JOIN ──┬── ASOF JOIN ──▶ 最终数据
               │               │               │
daily_basics ──┘               │               │
                               │               │
stock_profiles ────────────────┘               │
                                               │
fina_indicators ───────────────────────────────┘
         │
         └── 按 ann_date 向后查找最近公告（避免前视偏差）
```

---

## 3. 数据准备管道

### 3.1 训练/验证集分割

`prepare_datasets()` 函数负责数据分割：

```python
def prepare_datasets(config: RlStockConfig) -> Tuple[pd.DataFrame, pd.DataFrame]:
    """
    准备训练集和验证集。
    
    策略：
    1. 加载完整数据集
    2. 按日期范围分割训练集
    3. 按日期范围分割验证集
    4. 处理空数据集情况
    """
    # 加载数据
    df = load_training_dataframe(config).copy()
    df["trade_date"] = df["trade_date"].astype(str)
    
    # 训练集：start_date ~ end_date
    train_df = _filter_by_date_range(df, config.start_date, config.end_date)
    
    # 验证集：val_start_date ~ val_end_date
    eval_df = _filter_by_date_range(
        df,
        config.val_start_date or config.start_date,
        config.val_end_date or config.end_date,
    )
    
    # 空训练集处理：使用完整数据
    if train_df.empty:
        print("⚠️ 训练集为空，回退到完整数据集")
        train_df = df.copy()
    
    # 空验证集处理：使用训练集尾部 20%
    if eval_df.empty:
        print("⚠️ 验证集为空，使用训练集尾部 20% 作为验证数据")
        tail_size = max(int(len(train_df) * 0.2), config.lookback * 2)
        eval_df = train_df.tail(tail_size)
    
    return train_df.reset_index(drop=True), eval_df.reset_index(drop=True)
```

### 3.2 配置示例

```python
from src.train.rl_stock.config import RlStockConfig

# 标准时间分割
config = RlStockConfig(
    start_date="20200101",      # 训练开始
    end_date="20231231",        # 训练结束
    val_start_date="20240101",  # 验证开始
    val_end_date="20241231",    # 验证结束
)

# 滚动训练（验证集与训练集重叠）
config = RlStockConfig(
    start_date="20200101",
    end_date="20240630",
    val_start_date="20240101",  # 与训练重叠
    val_end_date="20241231",
)
```

### 3.3 数据预处理流程

`RLPreprocessor` 执行特征工程：

```python
from src.train.rl_stock.preprocessor import RLPreprocessor

# 初始化预处理器
preprocessor = RLPreprocessor(lookback=config.lookback)

# 拟合（仅训练集）
preprocessor.fit(train_df)

# 转换
train_data = preprocessor.transform(train_df)
eval_data = preprocessor.transform(eval_df)
```

**特征分组归一化策略：**

| 分组 | 特征 | 归一化方法 | 原因 |
|------|------|-----------|------|
| price | open, high, low, close | StandardScaler | 价格分布接近正态 |
| volume | vol, amount | RobustScaler | 成交量有极端值 |
| ratio | pe, pb, ps, turnover_rate | RobustScaler | 比率指标有离群点 |
| market_value | total_mv, circ_mv | RobustScaler | 市值差异大 |
| financial | roe, eps, bps | RobustScaler | 财务指标有异常值 |

```python
# 预处理器内部实现
FEATURE_GROUPS = {
    "price": ["open", "high", "low", "close"],
    "volume": ["vol", "amount"],
    "ratio": ["pe", "pb", "ps", "turnover_rate"],
    "market_value": ["total_mv", "circ_mv"],
    "financial": ["roe", "eps", "bps"],
}

GROUP_NORMALIZE_METHODS = {
    "price": "standard",
    "volume": "robust",
    "ratio": "robust",
    "market_value": "robust",
    "financial": "robust",
}
```

---

## 4. 日期范围过滤

### 4.1 过滤实现

```python
def _filter_by_date_range(
    df: pd.DataFrame,
    start_date: str | None,
    end_date: str | None,
) -> pd.DataFrame:
    """
    按日期范围过滤数据。
    
    Args:
        df: 包含 'trade_date' 列的 DataFrame
        start_date: 开始日期（YYYYMMDD），None 表示不限制
        end_date: 结束日期（YYYYMMDD），None 表示不限制
    
    Returns:
        过滤后的 DataFrame（副本）
    """
    result = df
    if start_date:
        result = result[result["trade_date"] >= start_date]
    if end_date:
        result = result[result["trade_date"] <= end_date]
    return result.copy()
```

### 4.2 日期验证

配置类中包含日期验证：

```python
@dataclass
class RlStockConfig:
    start_date: Optional[str] = "20200101"
    end_date: Optional[str] = "20231231"
    val_start_date: Optional[str] = "20240101"
    val_end_date: Optional[str] = "20241231"
    
    def _validate_split_ranges(self) -> None:
        """确保训练/验证日期范围合法。"""
        for label, start, end in (
            ("train", self.start_date, self.end_date),
            ("val", self.val_start_date, self.val_end_date),
        ):
            if start and end and start > end:
                raise ValueError(f"{label} 日期范围不合法: {start} > {end}")
```

### 4.3 自动日期范围计算

数据获取时自动扩展日期范围：

```python
# 计算覆盖训练和验证的最小日期范围
date_starts = [d for d in [config.start_date, config.val_start_date] if d]
date_ends = [d for d in [config.end_date, config.val_end_date] if d]
data_start = min(date_starts) if date_starts else config.start_date
data_end = max(date_ends) if date_ends else config.end_date
```

---

## 5. 缓存系统

### 5.1 缓存路径生成

```python
def _resolve_cache_path(config: RlStockConfig) -> str | None:
    """
    生成缓存文件路径。
    
    路径格式：{cache_dir}/{stock_pool}_{start}_{end}_{val_start}_{val_end}.pkl
    
    示例：runs/rl_stock/cache/hs300_20200101_20231231_20240101_20241231.pkl
    """
    if not config.cache_data:
        return None
    
    cache_dir = config.data_cache_dir or os.path.join(config.save_dir, "cache")
    os.makedirs(cache_dir, exist_ok=True)
    
    # 使用日期范围生成唯一标识
    date_tokens = [
        config.start_date,
        config.end_date,
        config.val_start_date,
        config.val_end_date,
    ]
    safe_tokens = [token for token in date_tokens if token]
    cache_name = "_".join([config.stock_pool, *safe_tokens]) or "dataset"
    
    return os.path.join(cache_dir, f"{cache_name}.pkl")
```

### 5.2 缓存生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                      缓存生命周期                            │
└─────────────────────────────────────────────────────────────┘

    配置变更 ──────┐
                   │
                   ▼
    ┌─────────────────────────────────────┐
    │     cache_data=True?                │
    └─────────────┬───────────────────────┘
                  │
         ┌───────┴───────┐
         │ No            │ Yes
         ▼               ▼
    ┌─────────┐    ┌─────────────────┐
    │ 禁用缓存 │    │ 检查缓存文件     │
    │ 每次重新 │    └────────┬────────┘
    │ 从DB获取 │             │
    └─────────┘      ┌───────┴───────┐
                     │ 存在          │ 不存在
                     ▼               ▼
              ┌───────────┐   ┌─────────────┐
              │ 加载缓存   │   │ 从DB获取     │
              │ (秒级)     │   │ (分钟级)     │
              └───────────┘   └──────┬──────┘
                                      │
                                      ▼
                               ┌─────────────┐
                               │ 保存缓存     │
                               └─────────────┘
```

### 5.3 缓存配置

```python
# 默认配置（启用缓存）
config = RlStockConfig(
    cache_data=True,
    data_cache_dir="runs/rl_stock/cache",  # 默认路径
)

# 禁用缓存
config = RlStockConfig(
    cache_data=False,
)

# 自定义缓存路径
config = RlStockConfig(
    cache_data=True,
    data_cache_dir="/path/to/custom/cache",
)

# 命令行禁用
# python -m src.train.rl_stock.train_rl_stock --no-cache-data
```

### 5.4 缓存失效策略

| 场景 | 缓存行为 |
|------|---------|
| 修改日期范围 | 生成新缓存文件 |
| 切换股票池 | 生成新缓存文件 |
| 数据库更新 | 需手动删除旧缓存 |
| 禁用缓存 | 跳过缓存检查 |

---

## 6. 股票池选择

### 6.1 支持的股票池

| 股票池 | 指数代码 | 成分股数量 | 特点 |
|--------|---------|-----------|------|
| hs300 | 399300.SZ | ~300 | 沪深300，大盘蓝筹 |
| zz500 | 000905.SH | ~500 | 中证500，中盘成长 |
| zz1000 | 000852.SH | ~1000 | 中证1000，小盘股 |

### 6.2 成分股获取

```python
# TushareProvider.fetch_index_constituents()
def fetch_index_constituents(self, index_code, trade_date=None):
    """
    获取指数成分股。
    
    策略：
    1. 尝试获取最近60天的权重数据
    2. 回退到指数成员接口
    3. 过滤已退市股票
    """
    if trade_date:
        df = self.pro.index_weight(index_code=index_code, trade_date=trade_date)
    else:
        # 查找最近有数据的日期
        for days_back in range(0, 60):
            check_date = (datetime.now() - timedelta(days=days_back)).strftime("%Y%m%d")
            df = self.pro.index_weight(index_code=index_code, trade_date=check_date)
            if df is not None and not df.empty:
                break
    
    # 回退方案
    if df is None or df.empty:
        df = self.pro.index_member(index_code=index_code)
        today = datetime.now().strftime("%Y%m%d")
        active = df[(df["out_date"].isna()) | (df["out_date"] > today)]
    
    return active["con_code"].unique().tolist()
```

### 6.3 股票数量限制

用于调试和加速训练：

```python
# 通过配置限制
config = RlStockConfig(
    stock_pool="hs300",
    max_stocks=50,  # 仅使用前50只股票
)

# 通过环境变量限制
# export RL_STOCK_MAX_CODES=50
```

### 6.4 股票池配置示例

```python
# 大盘蓝筹策略
config = RlStockConfig(
    stock_pool="hs300",
    start_date="20180101",
    end_date="20231231",
)

# 中小盘成长策略
config = RlStockConfig(
    stock_pool="zz1000",
    start_date="20200101",
    end_date="20231231",
)

# 调试模式
config = RlStockConfig(
    stock_pool="hs300",
    max_stocks=10,
    use_mock_data=True,
)
```

---

## 7. 数据验证与错误处理

### 7.1 配置验证

`RlStockConfig` 在初始化时执行完整验证：

```python
@dataclass
class RlStockConfig:
    def __post_init__(self) -> None:
        """初始化后执行所有验证。"""
        self._validate_stock_pool()
        self._validate_environment_params()
        self._validate_training_params()
        self._validate_ppo_params()
        self._validate_reward_params()
        self._validate_backtest_params()
        self._validate_rebalance_params()
        self._validate_split_ranges()
        self._validate_runtime_options()
        self._validate_data_limits()
```

**验证规则：**

| 参数 | 验证规则 | 错误信息 |
|------|---------|---------|
| stock_pool | 必须是 hs300/zz500/zz1000 | "stock_pool 必须是 {valid_pools} 之一" |
| initial_balance | 必须 > 0 | "initial_balance 必须为正数" |
| transaction_cost | 必须在 [0, 0.1] | "transaction_cost 必须在 [0, 0.1] 范围内" |
| lookback | 必须 > 0 | "lookback 必须为正整数" |
| trade_threshold | 必须在 [0, 1] | "trade_threshold 必须在 [0, 1] 范围内" |
| learning_rate | 必须 > 0 | "learning_rate 必须为正数" |
| batch_size | 必须 > 0 | "batch_size 必须为正整数" |
| gamma | 必须在 (0, 1] | "gamma 必须在 (0, 1] 范围内" |

### 7.2 环境数据验证

`PortfolioTradingEnv` 在初始化时验证输入数据：

```python
def _validate_dataframe(self, df: pd.DataFrame) -> None:
    """验证输入 DataFrame 包含必需列。"""
    # 检查空数据
    if df.empty:
        raise ValueError("输入 DataFrame 为空")
    
    # 检查必需列
    required_cols = ["ts_code", "trade_date", "close"]
    missing = [col for col in required_cols if col not in df.columns]
    if missing:
        raise ValueError(f"缺少必需列: {missing}")
    
    # 检查数据量
    unique_dates = df["trade_date"].nunique()
    if unique_dates < self.lookback + 1:
        raise ValueError(
            f"交易日数量 {unique_dates} 小于 lookback + 1 = {self.lookback + 1}"
        )
```

### 7.3 预处理器验证

```python
def fit(self, df: pd.DataFrame) -> Self:
    """拟合预处理器时验证数据。"""
    # 检查特征列存在性
    available_features = [col for col in self.feature_cols if col in df.columns]
    missing_features = [col for col in self.feature_cols if col not in df.columns]
    
    if missing_features:
        print(f"⚠️ 以下特征列不存在，将被忽略: {missing_features}")
    
    if not available_features:
        raise ValueError("没有可用的特征列")
    
    # 检查数据有效性
    df_clean = df.dropna(subset=self.feature_cols)
    if df_clean.empty:
        raise ValueError("处理后数据为空，所有行都包含缺失值")
```

### 7.4 错误处理最佳实践

```python
# 数据获取错误处理
try:
    df = fetch_stock_data(config)
except RuntimeError as e:
    print(f"❌ 数据获取失败: {e}")
    print("💡 请检查：")
    print("   1. 数据库是否已初始化")
    print("   2. Tushare Token 是否有效")
    print("   3. 网络连接是否正常")
    sys.exit(1)

# 空数据处理
if df is None or df.empty:
    raise RuntimeError("数据库数据为空，无法继续训练")

# 成分股获取失败
stock_codes = provider.fetch_index_constituents(index_code=index_code)
if not stock_codes:
    raise RuntimeError(f"获取 {config.stock_pool} 成分股失败")
```

---

## 8. 最佳实践

### 8.1 开发调试

```bash
# 使用模拟数据快速验证
python -m src.train.rl_stock.train_rl_stock \
    --use-mock-data \
    --total-timesteps 10000 \
    --n-envs 1

# 限制股票数量加速调试
python -m src.train.rl_stock.train_rl_stock \
    --stock-pool hs300 \
    --max-stocks 10 \
    --total-timesteps 50000
```

### 8.2 生产训练

```bash
# 标准训练配置
python -m src.train.rl_stock.train_rl_stock \
    --stock-pool zz1000 \
    --start-date 20180101 \
    --end-date 20231231 \
    --val-start-date 20240101 \
    --val-end-date 20241231 \
    --total-timesteps 5000000 \
    --n-envs 8 \
    --device cuda
```

### 8.3 内存优化

对于大规模股票池，注意内存管理：

```python
# 估算内存占用
def _estimate_observation_bytes(df, preprocessor, config):
    n_stocks = df["ts_code"].nunique()
    n_days = max(df["trade_date"].nunique() - config.lookback, 0)
    n_features = len(preprocessor.fitted_feature_cols)
    total_floats = n_days * config.lookback * n_stocks * n_features
    return total_floats * 4  # float32

# 自动降级策略
if estimated_total_bytes > 6 * 1024**3:  # 6GB
    print("⚠️ 数据体积较大，自动降级为 DummyVecEnv")
    config.n_envs = 1
    config.vec_backend = "dummy"
```

### 8.4 数据缓存管理

```bash
# 查看缓存
ls -la runs/rl_stock/cache/

# 清除特定股票池缓存
rm runs/rl_stock/cache/hs300_*.pkl

# 清除所有缓存
rm -rf runs/rl_stock/cache/

# 数据库更新后清除缓存
python -m src.stock_db_system.update_data && rm -rf runs/rl_stock/cache/
```

### 8.5 完整工作流示例

```python
from src.train.rl_stock.config import RlStockConfig
from src.train.rl_stock.preprocessor import RLPreprocessor
from src.train.rl_stock.environment import PortfolioTradingEnv
from src.train.rl_stock.train_rl_stock import (
    load_training_dataframe,
    prepare_datasets,
)

# 1. 配置
config = RlStockConfig(
    stock_pool="hs300",
    start_date="20200101",
    end_date="20231231",
    val_start_date="20240101",
    val_end_date="20241231",
    lookback=30,
    cache_data=True,
)

# 2. 数据准备
train_df, eval_df = prepare_datasets(config)
print(f"训练集: {len(train_df)} 条, 验证集: {len(eval_df)} 条")

# 3. 预处理
preprocessor = RLPreprocessor(lookback=config.lookback)
preprocessor.fit(train_df)

# 4. 创建环境
train_env = PortfolioTradingEnv(
    df=train_df,
    config=config,
    preprocessor=preprocessor,
    mode="train",
)

# 5. 训练（使用 stable-baselines3）
from stable_baselines3 import PPO
model = PPO("MlpPolicy", train_env)
model.learn(total_timesteps=config.total_timesteps)

# 6. 保存
model.save("runs/rl_stock/model.zip")
preprocessor.save("runs/rl_stock/preprocessor.pkl")
```

---

## 附录

### A. 数据字段说明

| 字段 | 来源表 | 说明 | 用途 |
|------|--------|------|------|
| ts_code | daily_prices | 股票代码 | 股票标识 |
| trade_date | daily_prices | 交易日期 | 时间索引 |
| open/high/low/close | daily_prices | OHLC 价格 | 价格特征 |
| vol/amount | daily_prices | 成交量/额 | 流动性特征 |
| pe/pb/ps | daily_basics | 估值指标 | 估值特征 |
| turnover_rate | daily_basics | 换手率 | 活跃度特征 |
| total_mv/circ_mv | daily_basics | 总市值/流通市值 | 规模特征 |
| roe/eps/bps | fina_indicators | 财务指标 | 质量特征 |

### B. 命令行参数速查

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--stock-pool` | hs300 | 股票池选择 |
| `--start-date` | 20200101 | 训练开始日期 |
| `--end-date` | 20231231 | 训练结束日期 |
| `--val-start-date` | 20240101 | 验证开始日期 |
| `--val-end-date` | 20241231 | 验证结束日期 |
| `--lookback` | 30 | 历史窗口大小 |
| `--max-stocks` | None | 限制股票数量 |
| `--use-mock-data` | False | 使用模拟数据 |
| `--no-cache-data` | False | 禁用缓存 |
| `--data-cache-dir` | runs/rl_stock/cache | 缓存目录 |

### C. 故障排除

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 数据库连接失败 | 未初始化数据库 | 运行 `init_stock_db()` |
| 成分股获取失败 | Tushare权限不足 | 检查Token权限 |
| 训练集为空 | 日期范围无数据 | 检查日期范围设置 |
| 内存溢出 | 股票数量过多 | 使用 `max_stocks` 限制 |
| 缓存加载失败 | 缓存文件损坏 | 删除缓存重新生成 |
