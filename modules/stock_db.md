# 股票数据库模块

## 使用

```bash
python -m src.stock_db_system.update_data
python -m src.stock_db_system.update_data --start-date 20240101
python -m src.stock_db_system.update_data --backfill --start-date 20200101
```

## Python API

```python
from src.stock_db_system import init_db, StockDataManager

init_db()
manager = StockDataManager()

df = manager.get_daily_data(
    ts_code="000001.SZ",
    start_date="20240101",
    end_date="20241231"
)
```

## 数据表

| 表名 | 说明 |
|------|------|
| stock_basic | 股票基本信息 |
| stock_daily | 日线行情 |
| stock_daily_basic | 每日指标 |
