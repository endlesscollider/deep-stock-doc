# 股票数据库模块 (stock_db_system)

## 模块概述

股票数据库模块用于 A股股票数据的获取、存储和管理。

**核心功能：**

- 📊 股票基本信息管理
- 📈 日线行情数据更新
- 📉 每日指标数据存储
- 💰 财务指标数据获取

**数据源：** Tushare Pro

**数据库：** SQLite (`data/stock_data.db`)

---

## 数据表结构

### stock_profile - 股票基本信息

| 字段 | 类型 | 说明 |
|------|------|------|
| ts_code | String(20) | 股票代码（主键）|
| symbol | String(10) | 股票代码 |
| name | String(50) | 股票名称 |
| area | String(20) | 地域 |
| industry | String(50) | 所属行业 |
| market | String(20) | 市场类型 |
| list_date | String(8) | 上市日期 |
| ... | ... | 更多字段 |

### daily_price - 日线行情

| 字段 | 类型 | 说明 |
|------|------|------|
| ts_code | String(20) | 股票代码 |
| trade_date | String(8) | 交易日期 |
| open | Float | 开盘价 |
| high | Float | 最高价 |
| low | Float | 最低价 |
| close | Float | 收盘价 |
| pre_close | Float | 昨收价 |
| change | Float | 涨跌额 |
| pct_chg | Float | 涨跌幅(%) |
| vol | Float | 成交量(手) |
| amount | Float | 成交额(千元) |

### daily_basic - 每日指标

| 字段 | 类型 | 说明 |
|------|------|------|
| ts_code | String(20) | 股票代码 |
| trade_date | String(8) | 交易日期 |
| close | Float | 收盘价 |
| turnover_rate | Float | 换手率(%) |
| turnover_rate_f | Float | 换手率（自由流通股）|
| volume_ratio | Float | 量比 |
| pe | Float | 市盈率（总市值/净利润）|
| pe_ttm | Float | 市盈率TTM |
| pb | Float | 市净率 |
| ps | Float | 市销率 |
| dv_ratio | Float | 股息率(%) |
| total_mv | Float | 总市值(万元) |
| circ_mv | Float | 流通市值(万元) |

### fina_indicator - 财务指标

| 字段 | 类型 | 说明 |
|------|------|------|
| ts_code | String(20) | 股票代码 |
| ann_date | String(8) | 公告日期 |
| end_date | String(8) | 报告期 |
| roe | Float | 净资产收益率 |
| roe_dt | Float | 净资产收益率-摊薄 |
| roa | Float | 总资产报酬率 |
| eps | Float | 每股收益 |
| bps | Float | 每股净资产 |
| ... | ... | 更多财务指标 |

---

## 模块结构

```
stock_db_system/
├── __init__.py        # 模块导出
├── database.py        # 数据库连接配置
├── models.py          # ORM 模型定义
├── provider.py        # Tushare 数据提供者
├── manager.py         # 数据管理器
├── update_data.py     # 数据更新入口
├── validator.py       # 数据验证器
└── validate_only.py   # 验证脚本
```

---

## 快速开始

### 1. 初始化数据库

```python
from src.stock_db_system import init_db

# 初始化数据库，创建所有表
init_db()
```

### 2. 创建数据管理器

```python
from src.stock_db_system import DataManager

manager = DataManager()
```

### 3. 更新数据

#### 命令行方式

```bash
# 增量更新（从数据库最新日期开始）
python -m src.stock_db_system.update_data

# 从指定日期开始更新
python -m src.stock_db_system.update_data --start-date 20230101

# 历史数据回补
python -m src.stock_db_system.update_data --backfill --start-date 20200101 --end-date 20231231
```

#### Python 代码方式

```python
# 更新日线数据
manager.update_daily_prices()

# 更新每日指标
manager.update_daily_basic()

# 更新财务指标
manager.update_fina_indicator()
```

---

## 数据查询

### 查询股票列表

```python
# 获取所有股票列表
stocks = manager.get_stock_list()

# 按条件筛选
stocks = manager.get_stock_list(
    industry="银行",
    market="主板"
)
```

### 查询日线数据

```python
# 获取单只股票日线
df = manager.get_daily_prices(
    ts_code="000001.SZ",
    start_date="20240101",
    end_date="20241231"
)

# 获取多只股票日线
df = manager.get_daily_prices(
    ts_code=["000001.SZ", "000002.SZ"],
    start_date="20240101"
)
```

### 查询合并数据

```python
# 获取合并的日线+指标数据
df = manager.get_merged_data(
    ts_code=["000001.SZ", "000002.SZ"],
    start_date="20240101",
    end_date="20241231",
    batch_size=50  # 批量大小
)
```

**返回字段：**

```
ts_code, trade_date, open, high, low, close, vol, amount,
pe, pb, ps, turnover_rate, total_mv, circ_mv, roe, eps, bps, ...
```

---

## 断点续传

数据更新支持自动断点续传：

- 中断后重新运行相同命令，自动从中断处继续
- 进度文件保存在 `.cache/` 目录
- 每个日期完成后立即保存进度

```python
# 示例：更新中断后恢复
# 第一次运行（更新到 2024-06-15 时中断）
python -m src.stock_db_system.update_data

# 第二次运行（自动从 2024-06-16 继续）
python -m src.stock_db_system.update_data
```

---

## 数据验证

### 验证数据完整性

```bash
# 验证所有数据
python -m src.stock_db_system.validate_only
```

### Python 代码验证

```python
from src.stock_db_system.validator import DataValidator

validator = DataValidator()

# 验证日线数据
result = validator.validate_daily_prices(
    start_date="20240101",
    end_date="20241231"
)

print(f"数据完整性: {result['completeness']:.2%}")
print(f"缺失日期: {result['missing_dates']}")
```

---

## 高级用法

### 获取指数成分股

```python
from src.stock_db_system.provider import TushareProvider

provider = TushareProvider()

# 获取沪深300成分股
hs300_stocks = provider.fetch_index_constituents(
    index_code="399300.SZ"
)

# 获取中证500成分股
zz500_stocks = provider.fetch_index_constituents(
    index_code="000905.SH"
)
```

### 自定义数据查询

```python
from src.stock_db_system.database import SessionLocal
from src.stock_db_system.models import DailyPrice
from sqlalchemy import func

session = SessionLocal()

# 查询某日涨幅最大的股票
top_gainers = session.query(DailyPrice)\
    .filter(DailyPrice.trade_date == "20241231")\
    .order_by(DailyPrice.pct_chg.desc())\
    .limit(10)\
    .all()

session.close()
```

---

## 定时任务

### Linux Crontab

```bash
# 每天 18:00 更新股票数据
0 18 * * * cd /path/to/deep-stock && python -m src.stock_db_system.update_data >> logs/stock_update.log 2>&1
```

### Windows 任务计划程序

创建批处理文件 `update_stock.bat`：

```batch
@echo off
cd /d D:\deep-stock
python -m src.stock_db_system.update_data
pause
```

---

## 常见问题

### Q: Tushare 权限不足？

A: 部分接口需要积分或单独申请权限。登录 Tushare 官网查看你的权限。

### Q: 数据更新失败？

A: 检查：

1. Tushare Token 是否配置正确
2. 网络连接是否正常
3. Tushare 服务是否正常

### Q: 如何只更新特定股票？

A: 目前支持全市场更新。如需特定股票，可在代码中过滤：

```python
df = manager.get_merged_data(
    ts_code=["000001.SZ", "000002.SZ"],
    start_date="20240101"
)
```

---

## 相关文档

- [ETF数据库](etf_db.md)
- [新闻数据库](news_db.md)
- [模块概述](overview.md)
