# 网络数据爬取管理系统交付说明

本目录是一个可交付副本，包含网络数据爬取管理系统的代码、数据库建表脚本、运行脚本和配置模板。接收方可以在自己的电脑上导入数据库、安装依赖，然后运行 Web 管理界面和 Scrapy 爬虫。

> 注意：本交付副本不包含原机器的 `.env` 文件，因为其中可能包含数据库密码。请复制 `.env.example` 为 `.env` 后自行填写。

---

## 1. 项目功能概览

本项目实现了一个“网络数据爬取管理系统”，主要能力包括：

1. 管理待爬取网站，例如人民网、新华网、中国政府网等。
2. 为每个网站配置爬取规则，例如正文 XPath、标题 XPath、图片 XPath。
3. 使用 Scrapy 爬取网页、正文内容和图片信息。
4. 使用 trafilatura 优化正文提取，减少导航、页脚、版权声明等无效文本。
5. 对装饰性小图片、图标、像素追踪图进行过滤。
6. 使用 Flask Web 页面查看仪表盘、网站列表、爬取任务、爬取结果、网页详情和搜索结果。
7. 使用 MySQL 保存网站、网页、正文、图片、数据源和爬取日志。

---

## 2. 目录结构说明

```text
network-crawler-delivery/
├─ crawler/                    # Scrapy 爬虫项目
│  ├─ scrapy.cfg                # Scrapy 配置入口
│  ├─ settings.py               # 爬虫配置：数据库、日志、图片过滤、并发、限速
│  ├─ items.py                  # Scrapy Item 定义：网页、正文、图片、数据源
│  ├─ pipelines.py              # 入库 Pipeline：写入 webpage/content/image 等表
│  ├─ middlewares.py            # Scrapy 中间件
│  ├─ spiders/
│  │  ├─ base_spider.py         # 基础爬虫类，负责构造通用 Item
│  │  └─ generic_spider.py      # 通用规则爬虫，从数据库读取网站和规则
│  └─ utils/
│     └─ db.py                  # PyMySQL 数据库连接和插入函数
│
├─ webapp/                      # Flask Web 管理后台
│  ├─ app.py                    # Flask 应用入口，注册蓝图和日志
│  ├─ config.py                 # Web 端数据库配置，读取 .env
│  ├─ models.py                 # SQLAlchemy 模型
│  ├─ routes/                   # 页面路由
│  │  ├─ website.py             # 网站管理、爬取规则管理
│  │  ├─ crawl.py               # 爬取管理、结果页、详情页、日志页
│  │  ├─ search.py              # 内容搜索
│  │  └─ stats.py               # 仪表盘统计
│  └─ templates/                # Jinja2 HTML 模板
│
├─ sql/
│  └─ schema.sql                # MySQL 建表脚本
│
├─ database/                    # 数据库导出文件目录
│  ├─ web_crawler_system_full_dump.sql       # 可选：结构 + 数据导出
│  └─ web_crawler_system_data_only.sql       # 可选：仅数据导出
│
├─ docs/
│  └─ database-design.md        # 数据库设计文档
│
├─ scripts/
│  ├─ verify-user-stories.ts    # 用户故事格式校验脚本
│  └─ ralph/                    # Ralph 自动代理循环相关脚本
│
├─ logs/                        # 运行日志目录，初始为空
├─ data/images/                 # 图片存储目录，初始为空
├─ .env.example                 # 环境变量模板
├─ requirements.txt             # Python 依赖
├─ run-webapp.ps1               # 启动 Web 管理界面
├─ run-crawler.ps1              # 启动 Scrapy 爬虫
├─ import-database.ps1          # 导入数据库结构和数据
├─ export-database.ps1          # 导出数据库结构和数据
└─ README.md                    # 本说明文件
```

---

## 3. 环境要求

推荐环境：

- Windows 10/11
- Python 3.11
- Conda 或 venv
- MySQL 5.7 或 MySQL 8.0
- PowerShell

Python 依赖见 `requirements.txt`，主要包括：

- Flask
- Flask-SQLAlchemy
- Scrapy
- PyMySQL
- BeautifulSoup4
- lxml
- trafilatura
- python-dotenv

---

## 4. 第一次运行步骤

### 4.1 创建 Python 环境

如果使用 Conda：

```powershell
conda create -n sqlenv python=3.11
conda activate sqlenv
```

如果已有环境，直接激活即可。

### 4.2 安装依赖

在项目根目录运行：

```powershell
cd path\to\network-crawler-delivery
pip install -r requirements.txt
```

### 4.3 配置数据库连接

复制配置模板：

```powershell
Copy-Item .env.example .env
```

然后编辑 `.env`：

```text
SECRET_KEY=change-me

DB_USER=root
DB_PASSWORD=你的MySQL密码
DB_HOST=localhost
DB_PORT=3306
DB_NAME=web_crawler_system
```

说明：

- `DB_USER`：MySQL 用户名
- `DB_PASSWORD`：MySQL 密码
- `DB_HOST`：数据库主机，通常是 `localhost`
- `DB_PORT`：MySQL 端口，通常是 `3306`
- `DB_NAME`：数据库名，默认 `web_crawler_system`

### 4.4 导入数据库

确保 MySQL 服务已经启动，然后运行：

```powershell
.\import-database.ps1
```

这个脚本会：

1. 创建数据库 `web_crawler_system`。
2. 导入 `sql/schema.sql` 建表。
3. 如果存在 `database/web_crawler_system_data_only.sql`，继续导入数据。

如果你只拿到了结构，没有拿到数据 dump，那么导入后数据库是空的，需要在 Web 页面里添加网站和规则，或者自己插入示例数据。

---

## 5. 运行 Web 管理界面

在项目根目录运行：

```powershell
.\run-webapp.ps1
```

等看到类似输出：

```text
* Running on http://127.0.0.1:5000
```

然后浏览器打开：

```text
http://localhost:5000
```

常用页面：

- `/stats/`：仪表盘
- `/websites/`：网站管理
- `/crawl/`：爬取管理
- `/crawl/results`：爬取结果
- `/search/`：内容搜索

停止 Web 服务：在终端按 `Ctrl + C`。

---

## 6. 运行爬虫

爬虫依赖数据库中的 `website` 和 `crawl_rule` 表。

运行：

```powershell
.\run-crawler.ps1
```

这个脚本会进入 `crawler/` 目录，并设置：

```powershell
$env:PYTHONPATH = $PSScriptRoot
```

这样 Scrapy 才能正确 import 项目包。

如果你手动运行，也可以这样：

```powershell
cd path\to\network-crawler-delivery\crawler
$env:PYTHONPATH = "path\to\network-crawler-delivery"
scrapy crawl generic
```

---

## 7. 数据库导入与导出脚本

### 7.1 导入数据库

```powershell
.\import-database.ps1
```

用途：接收方第一次部署项目时使用。

导入顺序：

1. `sql/schema.sql`
2. `database/web_crawler_system_data_only.sql`，如果存在

### 7.2 导出数据库

```powershell
.\export-database.ps1
```

用途：交付者需要把当前数据库数据一并交给别人时使用。

会生成：

```text
database/web_crawler_system_full_dump.sql
database/web_crawler_system_data_only.sql
```

说明：

- `full_dump` 包含结构和数据，适合完整备份。
- `data_only` 只包含数据，适合配合 `sql/schema.sql` 使用。

---

## 8. 代码模块说明

### 8.1 爬虫入口：`crawler/spiders/generic_spider.py`

这是当前最核心的爬虫文件。

主要职责：

1. 从数据库读取启用的网站：

   ```sql
   SELECT website_id, domain FROM website WHERE is_active = 1
   ```

2. 访问每个网站首页。
3. 从 `crawl_rule` 表读取规则。
4. 保存网页元数据。
5. 提取正文内容。
6. 提取图片信息。
7. 提取后续链接继续爬取。

正文提取策略：

1. 优先使用 `trafilatura` 提取正文。
2. 失败后使用智能 CSS 选择器，例如 `article`、`main`、`.article-content`。
3. 再失败后使用数据库配置的 XPath。
4. 最后才兜底提取 `body` 文本。

文本清洗策略：

- 删除导航栏碎片，例如 `首页 > 政策 > 解读`
- 删除工具栏碎片，例如 `字号 | 默认 | 大 | 超大 | 打印`
- 删除分享、登录、邮箱、无障碍等功能按钮
- 丢弃纯版权声明、纯 URL、过短文本等低质量内容

### 8.2 图片过滤：`crawler/settings.py` + `generic_spider.py`

图片过滤配置位于 `crawler/settings.py`：

```python
IMAGE_MIN_WIDTH = 80
IMAGE_MIN_HEIGHT = 80
IMAGE_SKIP_PATTERNS = [...]
IMAGE_DECORATION_NAME_PATTERNS = [...]
```

用于过滤：

- `1x1`、`pixel`、`tracking`、`beacon`
- `arrow`、`icon`、`btn_`、`sprite`
- 小于 80x80 的图片

### 8.3 入库逻辑：`crawler/pipelines.py`

包含三个 Pipeline：

- `WebpagePipeline`：写入 `webpage`
- `ContentPipeline`：写入 `content`
- `ImagePipeline`：写入 `image`

注意：`ContentItem` 和 `ImageItem` 通过 `page_url` 反查 `webpage_id`，避免正文和图片入库时找不到对应网页。

### 8.4 数据库工具：`crawler/utils/db.py`

封装了数据库连接和插入函数：

- `get_connection()`
- `insert_webpage()`
- `insert_content()`
- `insert_image()`
- `find_webpage_id()`

### 8.5 Web 页面入口：`webapp/app.py`

Flask 应用入口，主要做：

1. 创建 Flask app。
2. 加载配置。
3. 初始化 SQLAlchemy。
4. 注册蓝图：
   - `website_bp`
   - `crawl_bp`
   - `search_bp`
   - `stats_bp`
5. 配置日志文件 `logs/webapp.log`。

---

## 9. 系统测试与功能验证

完成安装、数据库导入和 Web 启动后，可以按下面步骤验证系统是否部署成功。

### 9.1 Web 服务连通性测试

启动 Web 后台：

```powershell
.\run-webapp-prod.ps1
```

另开一个 PowerShell 窗口，执行：

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:5000/
```

如果返回状态码 `200`，说明 Flask Web 服务已经正常启动。

也可以检查端口监听：

```powershell
Get-NetTCPConnection -LocalPort 5000 -State Listen
```

如果显示 `0.0.0.0:5000`，说明系统支持局域网内其他设备通过 `http://本机IP:5000` 访问。

### 9.2 数据库导入验证

导入数据库后，可以用 MySQL 客户端执行：

```sql
USE web_crawler_system;

SELECT COUNT(*) AS websites FROM website;
SELECT COUNT(*) AS webpages FROM webpage;
SELECT COUNT(*) AS contents FROM content;
SELECT COUNT(*) AS images FROM image;
SELECT COUNT(*) AS search_index_rows FROM content_search_index;
SELECT COUNT(*) AS ai_analysis_rows FROM content_ai_analysis;
```

如果使用随包提供的完整快照，应该能看到网站、网页、正文、图片和搜索索引中都有数据。

### 9.3 页面功能验证

浏览器打开：

```text
http://localhost:5000
```

建议依次检查这些页面：

| 页面 | 地址 | 验证内容 |
| --- | --- | --- |
| 仪表盘 | `/` 或 `/stats/` | 能看到网站数、网页数、正文数、图片数和各网站统计 |
| 网站管理 | `/websites/` | 能看到已配置站点，例如政府网、新华网、虎扑等 |
| 爬取结果 | `/crawl/results` | 能按站点、状态码、有无正文、有无图片筛选 |
| 内容检索 | `/search/` | 能输入关键词，使用全文索引搜索文章 |
| 图片管理 | `/images/` | 能查看图片列表，支持格式、宽度、站点筛选 |
| 智能分析 | `/ai/` | 能看到已分析数量、最近分类和最近分析结果 |
| 维护页面 | `/maintenance/` | 能查看缺失标题、空正文、异常任务等维护指标 |

### 9.4 内容检索验证

进入 `/search/` 页面，测试关键词：

```text
篮球
政策
教育
AI
```

预期结果：

- 能返回匹配文章。
- 搜索结果显示搜索模式，例如“全文索引”或“语义全文索引”。
- 可以按“智能分类”“智能标签”“有图片 / 无图片”等条件进一步筛选。

如果选择语义搜索，系统会先调用大模型理解查询意图；如果没有配置 API Key，仍可使用普通关键词搜索。

### 9.5 数据库优化验证

可以运行数据库健康检查脚本：

```powershell
python scripts\database_health_report.py
```

也可以运行慢查询分析脚本：

```powershell
python scripts\slow_query_report.py
```

还可以在 MySQL 中检查关键索引：

```sql
SHOW INDEX FROM content_search_index;
SHOW INDEX FROM webpage;
SHOW INDEX FROM image;
SHOW INDEX FROM content_ai_label;
```

重点确认：

- `content_search_index` 存在全文索引 `ft_search_text`。
- `webpage` 存在 `url_hash` 唯一索引和时间分页相关索引。
- `image` 存在站点、格式、宽度、创建时间相关联合索引。
- `content_ai_label` 存在标签索引。

### 9.6 爬虫功能验证

如果只复现当前展示效果，不需要重新运行爬虫。

如果要验证爬虫功能，可以运行：

```powershell
.\run-crawler.ps1
```

然后检查：

- `/crawl/` 页面中是否出现新的爬取日志。
- `/crawl/results` 页面中网页数量是否增加。
- `webpage`、`content`、`image` 表中是否写入新记录。
- `content_search_index` 和 `website_stats` 是否刷新。

### 9.7 智能分析验证

如果 `.env` 中配置了 DeepSeek API Key，可以进入 `/ai/` 页面，选择少量文章进行分析。

验证点：

- `content.summary` 是否生成摘要。
- `content_ai_analysis` 是否新增或更新分类、质量分和置信度。
- `content_ai_label` 是否写入标签。
- `content_search_index` 中的 `ai_category` 和 `ai_labels_text` 是否同步更新。

如果没有 API Key，已有数据库快照中的智能分析结果仍然可以展示，只是不能继续生成新的摘要和标签。

---

## 10. 常见问题

### 10.1 `ModuleNotFoundError: No module named 'crawler'`

原因：没有设置 `PYTHONPATH`。

解决：使用脚本运行：

```powershell
.\run-crawler.ps1
```

或者手动：

```powershell
$env:PYTHONPATH = "项目根目录"
```

### 10.2 `Access denied for user 'root'@'localhost'`

原因：`.env` 中数据库用户名或密码错误。

检查：

```text
DB_USER=root
DB_PASSWORD=你的MySQL密码
```

### 10.3 `Can't connect to MySQL server on 'localhost'`

原因：MySQL 服务没有启动，或端口不是 3306。

检查：

```powershell
Get-Service *mysql*
```

启动服务需要管理员权限。

### 10.4 Web 页面能打开，但没有数据

可能原因：

1. 只导入了 `schema.sql`，没有导入数据 dump。
2. `website` 表为空。
3. 爬虫还没有运行。
4. 爬虫运行失败，检查 `logs/crawler.log`。

### 10.5 爬虫运行很久

正常。当前配置比较礼貌：

```python
DOWNLOAD_DELAY = 2
AUTOTHROTTLE_ENABLED = True
DEPTH_LIMIT = 3
```

如果只是调试，可以临时降低深度或减少网站数量。

---

## 11. 日志位置

```text
logs/webapp.log       # Flask Web 报错日志
logs/crawler.log      # Scrapy 默认日志
logs/crawl_debug.log  # 手动调试时可用
```

如果页面报错，可以先看：

```powershell
Get-Content logs\webapp.log -Tail 80
```

如果爬虫报错，可以先看：

```powershell
Get-Content logs\crawler.log -Tail 80
```

---

## 12. 交付注意事项

交付给别人前，请确认：

1. 不要包含真实 `.env`。
2. 如果要包含真实数据，请运行：

   ```powershell
   .\export-database.ps1
   ```

3. 如果不想交真实爬取数据，只保留 `sql/schema.sql` 即可。
4. 如果数据库 dump 很大，可以只交 `web_crawler_system_data_only.sql` 或准备少量示例数据。
5. 接收方第一次运行时，应先导入数据库，再启动 Web 和爬虫。

---

## 13. 推荐调试顺序

接收方拿到项目后，建议按这个顺序排查：

1. `pip install -r requirements.txt`
2. 配置 `.env`
3. `.\import-database.ps1`
4. `.\run-webapp.ps1`
5. 打开 `http://localhost:5000`
6. 查看网站列表是否有数据
7. `.\run-crawler.ps1`
8. 查看 `爬取结果` 和 `搜索`

这样最容易定位问题出在环境、数据库、Web 还是爬虫。
