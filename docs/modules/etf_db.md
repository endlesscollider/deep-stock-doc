# ETF数据库模块

## 使用

```bash
python -m src.etf_db_system.update_data
```

## Python API

```python
from src.etf_db_system import init_db, EtfDataManager

init_db()
manager = EtfDataManager()

df = manager.get_merged_etf_data(
    ts_code=["510300.SH", "510500.SH"],
    start_date="20240101",
    end_date="20241231"
)
```

## 数据表

| 表名 | 说明 |
|------|------|
| etf_basics | ETF基本信息 |
| etf_daily | ETF日线行情 |
