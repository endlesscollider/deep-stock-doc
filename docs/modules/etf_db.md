# ETF 数据库模块 (etf_db_system)

## 模块概述

ETF 数据库模块用于 ETF（交易所交易基金）数据的获取、存储和管理。

**核心功能：**

- 📊 ETF 基本信息管理
- 📈 ETF 日线行情数据更新
- 🔄 数据合并查询

**数据源：** Tushare Pro

**数据库：** SQLite (`data/etf_data.db`)

---

## 数据表结构

### etf_basic - ETF 基本信息

| 字段 | 类型 | 说明 |
|------|------|------|
| ts_code | String(20) | ETF 代码（主键）|
| name | String(50) | ETF 名称 |
| management | String(50) | 管理人 |
| custody | String(50) | 托管人 |
| fund_type | String(20) | 投资类型 |
| fund_setupdate | String(8) | 成立日期 |
| list_date | String(8) | 上市日期 |
| benchmark | String(200) | 基准 |
| ... | ... | 更多字段 |

### etf_daily - ETF 日线行情

| 字段 | 类型 | 说明 |
|------|------|------|
| ts_code | String(20) | ETF 代码 |
| trade_date | String(8) | 交易日期 |
| pre_close | Float | 昨收价 |
| open | Float | 开盘价 |
| high | Float | 最高价 |
| low | Float | 最低价 |
| close | Float | 收盘价 |
| change | Float | 涨跌额 |
| pct_chg | Float | 涨跌幅(%) |
| vol | Float | 成交量(手) |
| amount | Float | 成交额(千元) |

---

## 模块结构

```
etf_db_system/
├── __init__.py        # 模块导出
├── database.py        # 数据库连接配置
├── models.py          # ORM 模型定义
├── provider.py        # Tushare 数据提供者
├── manager.py         # ETF 数据管理器
├── update_data.py     # 数据更新入口
└── README.md          # 使用说明
```

---

## 快速开始

### 1. 初始化数据库

```python
from src.etf_db_system import init_db

# 初始化数据库，创建所有表
init_db()
```

### 2. 创建数据管理器

```python
from src.etf_db_system import EtfDataManager

manager = EtfDataManager()
```

### 3. 更新数据

#### 命令行方式

```bash
# 增量更新（从数据库最新日期开始）
python -m src.etf_db_system.update_data

# 从指定日期开始更新
python -m src.etf_db_system.update_data --start-date 20230101

# 历史数据回补
python -m src.etf_db_system.update_data --backfill --start-date 20200101
```

#### Python 代码方式

```python
# 更新 ETF 基本信息（全量覆盖）
manager.update_etf_basic()

# 更新 ETF 日线行情（增量更新）
manager.update_etf_daily()
```

---

## 数据查询

### 获取 ETF 基本信息

```python
# 获取所有 ETF 列表
etf_list = manager.get_etf_list()

# 按类型筛选
index_etfs = manager.get_etf_list(fund_type="指数型")
```

### 获取 ETF 日线数据

```python
# 获取单只 ETF 日线
df = manager.get_etf_daily(
    ts_code="510300.SH",
    start_date="20240101",
    end_date="20241231"
)

# 获取多只 ETF 日线
df = manager.get_etf_daily(
    ts_code=["510300.SH", "510500.SH"],
    start_date="20240101"
)
```

### 获取合并数据

```python
# 获取合并的日线+基本信息
df = manager.get_merged_etf_data(
    ts_code=["510300.SH", "510500.SH"],
    start_date="20240101",
    end_date="20241231"
)

print(df.head())
```

**返回字段：**

```
ts_code, trade_date, open, high, low, close, vol, amount,
name, management, fund_type, benchmark, ...
```

---

## ETF 代码示例

### 宽基指数 ETF

| 代码 | 名称 | 跟踪指数 |
|------|------|---------|
| 510300.SH | 沪深300ETF | 沪深300 |
| 510500.SH | 中证500ETF | 中证500 |
| 159915.SZ | 创业板ETF | 创业板指 |
| 588000.SH | 科创50ETF | 科创50 |

### 行业 ETF

| 代码 | 名称 | 跟踪指数 |
|------|------|---------|
| 512880.SH | 证券ETF | 证券公司 |
| 512690.SH | 酒ETF | 中证酒 |
| 159996.SZ | 消费ETF | 消费80 |

---

## 高级用法

### 筛选特定类型 ETF

```python
# 获取所有指数型 ETF
index_etfs = manager.get_merged_etf_data()

# 按名称筛选宽基 ETF
guangji_etfs = index_etfs[
    index_etfs['name'].str.contains('300|500|创业板|科创')
]

print(f"宽基 ETF 数量: {len(guangji_etfs['ts_code'].unique())}")
```

### 获取 ETF 净值数据

```python
# 从合并数据中提取净值信息
df = manager.get_merged_etf_data(
    ts_code="510300.SH",
    start_date="20240101"
)

# 计算累计收益
df['cumulative_return'] = (1 + df['pct_chg'] / 100).cumprod() - 1
```

---

## 定时任务

### Linux Crontab

```bash
# 每天 18:00 更新 ETF 数据
0 18 * * * cd /path/to/deep-stock && python -m src.etf_db_system.update_data >> logs/etf_update.log 2>&1
```

---

## 与 stock_db_system 的关系

**相同点：**

- 都使用 SQLAlchemy + SQLite 存储
- 都封装了统一的 Manager 管理类
- 都支持增量更新 + 合并查询

**不同点：**

| 方面 | stock_db_system | etf_db_system |
|------|----------------|---------------|
| 对象 | 个股（股票） | ETF 基金 |
| 数据库 | stock_data.db | etf_data.db |
| Tushare 接口 | daily, daily_basic, fina_indicator | fund_basic, fund_daily |
| 数据特点 | 包含财务指标 | 无财务指标，但净值清晰 |

---

## 常见问题

### Q: 如何获取所有宽基 ETF？

A: 可以通过名称或基准筛选：

```python
df = manager.get_merged_etf_data()

# 方法1：通过名称筛选
guangji = df[df['name'].str.contains('沪深300|中证500|创业板|科创50')]

# 方法2：通过基准筛选
guangji = df[df['benchmark'].str.contains('沪深300|中证500')]
```

### Q: ETF 数据更新频率？

A: 建议：

- 基本信息每周更新一次
- 日线行情每天更新一次

### Q: 如何区分场内 ETF 和场外基金？

A: Tushare 的 `fund_basic` 接口包含场外基金，需要过滤：

```python
# 只保留场内 ETF（代码以 51、15、58 开头）
etf_only = df[
    df['ts_code'].str.startswith(('51', '15', '58'))
]
```

---

## 相关文档

- [股票数据库](stock_db.md)
- [新闻数据库](news_db.md)
- [模块概述](overview.md)
