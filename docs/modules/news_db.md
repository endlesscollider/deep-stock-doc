# 新闻数据库模块

## 使用

```bash
# 每日更新
python -m src.news_db_system.update_data

# 历史回补
python -m src.news_db_system.update_data --backfill
```

## Python API

```python
from src.news_db_system import NewsDataManager

manager = NewsDataManager()

# 央视新闻
df = manager.get_cctv_news(start_date="20240101", end_date="20241231")

# 经济日历
df = manager.get_economic_data(start_date="2024-01-01", country="美国")

# 个股新闻
df = manager.get_stock_news(keyword="510300", start_date="20240101")
```

## 数据源

| 数据源 | 表名 | 说明 |
|--------|------|------|
| 央视新闻 | news_cctv | 宏观政策 |
| 经济日历 | news_economic | 经济数据 |
| 个股新闻 | news_stock | 个股新闻 |
| 财报发行 | news_report | 财报时间 |
