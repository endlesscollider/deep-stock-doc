# 新闻数据库模块 (news_db_system)

## 模块概述

新闻数据库模块用于财经新闻数据的获取、存储和管理，支持历史数据回补和每日增量更新。

**核心功能：**

- 📰 央视新闻联播数据
- 📊 经济日历数据
- 💹 个股/ETF 新闻
- 📈 财报发行时间

**数据源：**

- 央视网 (tv.cctv.com)
- 百度股市通 (gushitong.baidu.com)
- 东方财富 (eastmoney.com)

**数据库：** SQLite (`data/news_data.db`)

---

## 数据表说明

### news_cctv - 央视新闻联播

**数据来源：** 央视网 https://tv.cctv.com/lm/xwlb

**特点：** 支持历史数据回补（2013年至今）

**用途：** 宏观政策分析、重大事件追踪

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 主键 |
| date | String(8) | 新闻日期 YYYYMMDD |
| title | String(500) | 新闻标题 |
| content | Text | 新闻内容 |

### news_economic - 经济日历

**数据来源：** 百度股市通 gushitong.baidu.com/calendar

**特点：** 支持历史数据回补

**用途：** 经济数据发布追踪、重要指标监控

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 主键 |
| date | String(10) | 日期 YYYY-MM-DD |
| time | String(20) | 时间 |
| country | String(50) | 国家 |
| region | String(50) | 地区 |
| event | String(500) | 事件名称 |
| period | String(50) | 统计周期 |
| actual | Float | 公布值 |
| forecast | Float | 预期值 |
| previous | Float | 前值 |
| importance | Integer | 重要性星级 1-5 |

### news_stock - 个股/ETF新闻

**数据来源：** 东方财富 search-api-web.eastmoney.com

**特点：** 仅支持最新新闻，需每日存储积累历史

**用途：** 个股/ETF相关新闻追踪、情感分析

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 主键 |
| keyword | String(20) | 搜索关键词/股票代码 |
| title | String(500) | 新闻标题 |
| content | Text | 新闻内容/摘要 |
| pub_time | String(30) | 发布时间 |
| source | String(100) | 文章来源 |
| url | String(500) | 新闻链接 |
| fetch_date | String(8) | 抓取日期 YYYYMMDD |

### news_report - 财报发行

**数据来源：** 百度股市通

**特点：** 支持历史数据回补

**用途：** 财报发布时间追踪

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 主键 |
| date | String(10) | 发布日期 |
| code | String(20) | 股票代码 |
| name | String(50) | 股票简称 |
| exchange | String(20) | 交易所 |
| report_type | String(50) | 财报类型 |
| pub_time | String(30) | 发布时间 |
| market_value | Float | 市值 |

---

## 模块结构

```
news_db_system/
├── __init__.py        # 模块导出
├── database.py        # 数据库连接配置
├── models.py          # ORM 模型定义
├── provider.py        # 数据获取（爬虫）
├── manager.py         # 数据管理（增删改查）
├── update_data.py     # 一键更新入口
└── README.md          # 使用文档
```

---

## 快速开始

### 1. 安装依赖

```bash
pip install akshare pandas sqlalchemy requests beautifulsoup4 tqdm
```

### 2. 初始化数据库

```python
from src.news_db_system import init_db

# 初始化数据库，创建所有表
init_db()
```

### 3. 历史数据回补（首次运行）

```bash
# 回补所有历史数据
python -m src.news_db_system.update_data --backfill

# 仅回补央视新闻（从2020年开始）
python -m src.news_db_system.update_data --backfill --cctv --cctv-start 20200101

# 仅回补经济数据（最近365天）
python -m src.news_db_system.update_data --backfill --economic --economic-days 365
```

### 4. 每日更新

```bash
# 更新所有模块
python -m src.news_db_system.update_data

# 仅更新个股新闻
python -m src.news_db_system.update_data --stock

# 更新央视新闻和经济数据
python -m src.news_db_system.update_data --cctv --economic
```

---

## 数据查询

### 查询央视新闻

```python
from src.news_db_system import NewsDataManager

manager = NewsDataManager()

# 查询指定日期范围的央视新闻
df = manager.get_cctv_news(
    start_date="20240101",
    end_date="20241231"
)

# 按关键词筛选
df = manager.get_cctv_news(
    start_date="20240101",
    keyword="经济"
)

print(df.head())
```

### 查询经济日历

```python
# 查询指定日期范围的经济数据
df = manager.get_economic_data(
    start_date="2024-01-01",
    end_date="2024-12-31"
)

# 按国家筛选
df = manager.get_economic_data(
    start_date="2024-01-01",
    country="美国"
)

# 按重要性筛选
df = manager.get_economic_data(
    start_date="2024-01-01",
    importance=5  # 只看5星重要数据
)

print(df.head())
```

### 查询个股/ETF新闻

```python
# 查询指定关键词的新闻
df = manager.get_stock_news(
    keyword="510300",
    start_date="20240101",
    end_date="20241231"
)

print(df.head())
```

### 自定义关键词更新

```python
# 更新自定义ETF的新闻
manager.update_stock_news(
    keywords=["510300", "510500", "159915", "588000"],
    page_size=30
)
```

---

## 高级用法

### 新闻情感分析

```python
# 获取央视新闻
df = manager.get_cctv_news(
    start_date="20240101",
    keyword="经济"
)

# 简单情感分析（示例）
positive_keywords = ["增长", "上升", "改善", "利好"]
negative_keywords = ["下降", "衰退", "风险", "利空"]

def simple_sentiment(text):
    pos_count = sum(1 for kw in positive_keywords if kw in text)
    neg_count = sum(1 for kw in negative_keywords if kw in text)
    if pos_count > neg_count:
        return "positive"
    elif neg_count > pos_count:
        return "negative"
    return "neutral"

df['sentiment'] = df['content'].apply(simple_sentiment)
```

### 经济数据提醒

```python
# 获取本周重要经济数据
from datetime import datetime, timedelta

today = datetime.now()
week_end = today + timedelta(days=7)

df = manager.get_economic_data(
    start_date=today.strftime("%Y-%m-%d"),
    end_date=week_end.strftime("%Y-%m-%d"),
    importance=5
)

print("本周重要经济数据：")
print(df[['date', 'time', 'country', 'event']])
```

### 财报日历

```python
# 获取即将发布财报的公司
df = manager.get_report_data(
    start_date="2024-01-01",
    end_date="2024-03-31"
)

# 按财报类型分组
report_counts = df.groupby('report_type').size()
print(report_counts)
```

---

## 定时任务配置

### Linux Crontab

```bash
# 每天 18:00 更新新闻数据
0 18 * * * cd /path/to/deep-stock && python -m src.news_db_system.update_data >> logs/news_update.log 2>&1
```

### Windows 任务计划程序

创建批处理文件 `update_news.bat`：

```batch
@echo off
cd /d D:\deep-stock
python -m src.news_db_system.update_data
pause
```

---

## 数据更新策略

### 央视新闻

- **历史回补：** 支持 2013年至今的数据
- **更新频率：** 每日一次
- **特点：** 官方权威，适合宏观分析

### 经济日历

- **历史回补：** 支持历史数据
- **更新频率：** 每日一次
- **特点：** 数据驱动，重要指标监控

### 个股新闻

- **历史回补：** 不支持，需每日积累
- **更新频率：** 每日一次
- **特点：** 实时性强，适合个股分析

---

## 常见问题

### Q: 为什么个股新闻没有历史数据？

A: 东方财富的搜索API只返回最新新闻，需要每天运行积累历史数据。

### Q: 央视新闻能回补多久的数据？

A: 理论上支持2013年至今的数据，但早期数据可能不完整。

### Q: 如何添加新的ETF关键词？

A: 在调用 `update_stock_news()` 时传入 `keywords` 参数：

```python
manager.update_stock_news(
    keywords=["510300", "510500", "159915", "588000"],
    page_size=30
)
```

### Q: 数据抓取失败怎么办？

A: 检查：

1. 网络连接是否正常
2. 目标网站是否可访问
3. 是否被反爬虫机制拦截
4. 适当降低抓取频率

---

## 相关文档

- [股票数据库](stock_db.md)
- [ETF数据库](etf_db.md)
- [模块概述](overview.md)
