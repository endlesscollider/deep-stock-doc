# 新闻数据可视化面板

基于 Streamlit 构建的新闻数据可视化 Web 应用。

## 功能特性

- **三种新闻类型**：央视新闻、经济日历、个股新闻
- **日期范围筛选**：自定义开始和结束日期
- **关键词搜索**：支持标题关键词搜索
- **卡片式展示**：现代化暗色主题界面

## 快速启动

```bash
# 启动应用
python -m streamlit run dashboard/news_app.py

# 指定端口
python -m streamlit run dashboard/news_app.py --server.port 8501
```

## 访问地址

启动后访问：http://localhost:8501

## 使用说明

### 侧边栏操作

1. **选择新闻类型**：央视新闻 / 经济日历 / 个股新闻
2. **设置日期范围**：选择开始和结束日期
3. **筛选条件**：输入关键词
4. **每页显示**：调整每页显示的记录数

## 技术栈

- **前端框架**：Streamlit
- **数据源**：SQLite (news_db_system)

## 注意事项

- 确保 `news_db_system` 模块已正确配置
- 数据库文件位于 `data/news_data.db`
