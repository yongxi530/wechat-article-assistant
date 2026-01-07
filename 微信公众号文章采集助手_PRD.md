# 产品需求文档（PRD）
## 微信公众号文章采集助手

**版本：** 1.0  
**日期：** 2026-01-07  
**作者：** Product Team  
**状态：** Draft

---

## 1. 执行概要（Executive Summary）

### 1.1 产品概述

微信公众号文章采集助手是一个 Web 应用程序，旨在帮助用户高效地管理、采集和下载微信公众号的历史文章。该系统通过与微信公众平台接口集成，实现自动化的公众号信息获取和文章列表采集，并提供便捷的文章下载功能。

### 1.2 目标用户

- 内容运营人员：需要批量采集和存档公众号文章
- 研究人员：需要收集特定公众号的历史内容进行分析
- 自媒体从业者：需要参考和管理多个公众号的内容
- 个人用户：希望备份关注的公众号文章

### 1.3 核心价值

- **自动化采集**：自动获取公众号信息和历史文章列表，减少手工操作
- **批量管理**：支持多个公众号的统一管理和批量操作
- **灵活下载**：支持单篇、批量、全选等多种下载方式
- **持久化存储**：本地 SQLite 数据库存储，数据安全可控
- **命令行支持**：提供 CLI 工具，满足自动化和脚本化需求

### 1.4 项目目标

- 提供简洁易用的 Web 界面
- 实现稳定可靠的文章采集机制
- 支持大规模公众号和文章数据管理
- 确保与微信公众平台接口的兼容性

---

## 2. 功能需求（Features）

### 2.1 公众号管理

#### 2.1.1 公众号列表展示

**功能描述：**
- 以列表形式展示已添加的公众号信息
- 显示公众号头像、名称、别名、类型、采集状态等关键信息
- 支持列表排序（按创建时间、更新时间等）
- 显示每个公众号的文章采集进度

**交互说明：**
- 列表支持分页显示
- 鼠标悬停显示完整的公众号签名等详细信息
- 提供快速搜索功能，支持按公众号名称搜索

#### 2.1.2 新增公众号

**功能描述：**
点击"新增公众号"按钮，弹出录入界面，支持两种方式添加公众号：

**方式一：手工录入**
- 必填字段：公众号名称（nickname）
- 可选字段：
  - 公众号别名（alias）
  - 公众号唯一标识（fakeid）
  - 备注（memo）
  - 单页起始位置（begin，默认值：0）
  - 单页采集数量（count，默认值：10）

**方式二：自动获取**
- 输入公众号名称
- 点击"搜索"按钮，系统调用微信公众平台搜索接口
- 接口地址：`https://mp.weixin.qq.com/cgi-bin/searchbiz`
- 从搜索结果中选择目标公众号
- 自动填充公众号信息（fakeid、nickname、alias、round_head_img、service_type、signature、verify_status）

**登录态管理：**
- 首次使用或登录态失效时，弹出二维码扫码登录微信公众平台
- 登录成功后，保存登录态信息（Cookie/Token）
- 后续请求自动携带登录态，无需重复登录
- 登录态失效时自动触发重新登录流程

**数据验证：**
- 公众号名称不能为空
- fakeid 唯一性校验（避免重复添加）
- 单页采集数量范围：1-100

**交互说明：**
- 表单提交前进行客户端验证
- 提交成功后关闭弹窗，刷新公众号列表
- 显示操作成功提示信息

#### 2.1.3 编辑公众号

**功能描述：**
- 点击公众号列表中的"编辑"按钮
- 弹出编辑界面，允许修改公众号信息
- 可修改字段：备注、单页起始位置、单页采集数量
- 不可修改字段：fakeid、nickname 等从微信获取的信息

**交互说明：**
- 表单预填充当前公众号信息
- 保存后更新 update_time 字段

#### 2.1.4 删除公众号

**功能描述：**
- 点击"删除"按钮
- 弹出二次确认对话框
- 确认后删除公众号及其关联的所有文章记录

**交互说明：**
- 删除操作不可逆，需明确提示用户
- 删除成功后刷新列表

### 2.2 文章采集

#### 2.2.1 单页采集

**功能描述：**
- 点击公众号列表中的"采集"按钮
- 根据配置的 begin 和 count 参数采集一页文章
- 采集完成后，begin 自动增加 count 的值，便于下次继续采集

**采集流程：**
1. 检查登录态是否有效
2. 调用微信公众平台文章列表接口
3. 解析返回的文章数据
4. 保存到 wechat_article_list 表
5. 更新公众号的 collect_status 和 begin 字段

**数据处理：**
- 文章去重：根据 article_id 判断是否已存在
- 已存在的文章更新信息（标题、封面等可能变更）
- 新文章插入数据库

**登录态管理：**
- 采集过程中若登录态失效，弹出二维码重新登录
- 登录成功后自动继续采集

**交互说明：**
- 采集过程中显示进度提示
- 采集完成后显示采集数量统计
- 采集失败时显示错误信息

#### 2.2.2 全部采集

**功能描述：**
- 点击"采集全部"按钮
- 自动循环采集，直到获取完所有历史文章
- 采集过程中可以暂停或停止

**采集策略：**
- 从当前 begin 位置开始
- 每次采集 count 条
- 当返回的文章数小于 count 时，认为已采集完成
- 采集间隔：每页间隔 2-3 秒（避免请求过快被限制）

**状态管理：**
- 采集中：collect_status = "collecting"
- 采集完成：collect_status = "completed"
- 采集失败：collect_status = "failed"
- 暂停采集：collect_status = "paused"

**异常处理：**
- 网络错误：自动重试 3 次，重试间隔递增
- 接口限流：延长等待时间后重试
- 登录态失效：触发重新登录

**交互说明：**
- 实时显示采集进度（已采集/总页数）
- 采集过程中禁用其他操作按钮
- 支持后台采集，用户可以浏览其他页面

### 2.3 文章管理

#### 2.3.1 文章列表展示

**功能描述：**
- 展示已采集的所有文章
- 列表字段：
  - 文章封面（缩略图）
  - 文章标题
  - 公众号名称
  - 文章作者
  - 发布时间
  - 下载状态
  - 操作按钮

**列表功能：**
- 分页显示，每页 20/50/100 条可选
- 支持多选、全选
- 点击文章标题可在新窗口打开原文链接

#### 2.3.2 文章筛选

**功能描述：**
提供多维度筛选条件：

**筛选维度：**
- 公众号：下拉选择特定公众号
- 发布时间：时间范围选择（开始日期-结束日期）
- 下载状态：未下载 / 已下载 / 全部
- 关键词：搜索文章标题
- 删除状态：已删除 / 未删除 / 全部

**交互说明：**
- 支持多条件组合筛选
- 实时筛选或点击"应用筛选"按钮
- 提供"重置筛选"按钮清空所有条件
- 记住用户的筛选偏好（SessionStorage）

#### 2.3.3 文章下载

**功能描述：**
支持多种下载方式：
- 单篇下载：点击单条记录的"下载"按钮
- 批量下载：选中多条记录后点击"批量下载"
- 全选下载：勾选全选框后点击"下载选中"

**下载流程：**
1. 检查是否存在以"公众号名称"命名的文件夹
2. 不存在则创建文件夹：`downloads/{公众号名称}/`
3. 访问文章链接，获取文章内容
4. 保存文章到对应文件夹
5. 文件命名规则：`{发布时间}_{文章标题}.html`（特殊字符替换为下划线）
6. 更新数据库中的 is_downloaded 状态为 "yes"

**下载配置：**
- 保存格式：HTML（默认）
- 并发下载：最多 3 个并发任务
- 失败重试：单篇文章失败重试 2 次

**进度显示：**
- 显示下载进度条
- 实时更新已下载/总数
- 下载完成后显示成功/失败统计

**交互说明：**
- 下载过程中可以取消
- 已下载的文章可以重新下载（覆盖）
- 下载失败的文章标记颜色提示

### 2.4 命令行工具

#### 2.4.1 命令行下载

**功能描述：**
提供 CLI 工具，支持从命令行下载文章

**使用方式：**

```bash
# 下载单篇文章
python -m wechat_article_assistant download --url "https://mp.weixin.qq.com/s/xxxxx"

# 从文件批量下载
python -m wechat_article_assistant download --file urls.txt

# 指定输出目录
python -m wechat_article_assistant download --url "xxx" --output /path/to/save
```

**文件格式：**
urls.txt 每行一个文章链接：
```
https://mp.weixin.qq.com/s/xxxxx
https://mp.weixin.qq.com/s/yyyyy
https://mp.weixin.qq.com/s/zzzzz
```

**参数说明：**
- `--url`：单个文章链接
- `--file`：包含文章链接的文本文件路径
- `--output`：下载保存目录（默认：./downloads）
- `--format`：保存格式（html/markdown，默认：html）
- `--concurrent`：并发数（默认：3）

**功能特性：**
- 自动创建公众号文件夹（从文章页面解析公众号名称）
- 显示下载进度
- 错误日志记录
- 支持断点续传（跳过已下载的文章）

---

## 3. 技术架构（Technologies Used）

### 3.1 技术栈

#### 后端
- **Python**: 3.12+
- **Web 框架**: Flask 3.x
  - 轻量级，快速开发
  - 丰富的扩展生态
- **ORM**: SQLAlchemy 2.x
  - 强大的 ORM 功能
  - 支持数据库迁移
- **数据库**: SQLite 3
  - 零配置，文件型数据库
  - 适合单机部署
- **HTTP 客户端**: requests / httpx
  - 用于调用微信接口
- **HTML 解析**: BeautifulSoup4
  - 解析文章内容

#### 前端
- **HTML5 + JavaScript (ES6+)**
- **CSS 框架**: Tailwind CSS (CDN)
  - 实用优先的 CSS 框架
  - 快速构建响应式界面
- **HTTP 请求**: Fetch API
- **UI 组件**: 原生 JavaScript 实现

#### 开发工具
- **依赖管理**: uv
  - 快速的 Python 包管理器
- **代码格式化**: Ruff
  - 快速的 Python linter 和 formatter
- **类型检查**: mypy
  - 静态类型检查
- **测试框架**: pytest
  - 单元测试和集成测试
- **配置管理**: python-dotenv
  - 环境变量管理

### 3.2 系统架构

```
┌─────────────────────────────────────────────────────┐
│                   Browser (用户界面)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ 公众号管理页 │  │  文章列表页  │  │  CLI 工具 │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
└─────────────────────────────────────────────────────┘
                          │
                          │ HTTP / REST API
                          ↓
┌─────────────────────────────────────────────────────┐
│                Flask Application                     │
│  ┌──────────────────────────────────────────────┐  │
│  │              Route Handlers                   │  │
│  │  /api/wechat/list    /api/article/list       │  │
│  │  /api/wechat/add     /api/article/download   │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │           Business Logic Layer                │  │
│  │  • WeChatService   • ArticleService           │  │
│  │  • LoginService    • DownloadService          │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │            Data Access Layer                  │  │
│  │  • WeChatRepository  • ArticleRepository      │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                          │
                          │ SQLAlchemy ORM
                          ↓
┌─────────────────────────────────────────────────────┐
│                  SQLite Database                     │
│  ┌──────────────┐          ┌──────────────────┐    │
│  │ wechat_list  │──────────│wechat_article_list│    │
│  └──────────────┘  1 : N   └──────────────────┘    │
└─────────────────────────────────────────────────────┘

         ┌────────────────────────────────┐
         │   External Services            │
         │  • 微信公众平台 API             │
         │  • 文章内容获取                │
         └────────────────────────────────┘
```

### 3.3 目录结构设计

```
wechat-article-assistant/
├── src/
│   └── wechat_article_assistant/
│       ├── __init__.py
│       ├── __main__.py              # CLI 入口
│       ├── app.py                   # Flask 应用入口
│       ├── config.py                # 配置管理
│       ├── models/                  # 数据模型
│       │   ├── __init__.py
│       │   ├── wechat.py           # 公众号模型
│       │   └── article.py          # 文章模型
│       ├── repositories/            # 数据访问层
│       │   ├── __init__.py
│       │   ├── wechat_repository.py
│       │   └── article_repository.py
│       ├── services/                # 业务逻辑层
│       │   ├── __init__.py
│       │   ├── wechat_service.py
│       │   ├── article_service.py
│       │   ├── login_service.py
│       │   └── download_service.py
│       ├── routes/                  # 路由处理
│       │   ├── __init__.py
│       │   ├── wechat_routes.py
│       │   └── article_routes.py
│       ├── utils/                   # 工具函数
│       │   ├── __init__.py
│       │   ├── http_client.py
│       │   ├── file_helper.py
│       │   └── validators.py
│       ├── cli/                     # 命令行工具
│       │   ├── __init__.py
│       │   └── commands.py
│       └── static/                  # 静态资源
│           ├── css/
│           │   └── custom.css
│           └── js/
│               ├── wechat.js
│               └── article.js
├── templates/                       # HTML 模板
│   ├── layout.html                 # 基础布局
│   ├── wechat_list.html           # 公众号列表页
│   └── article_list.html          # 文章列表页
├── tests/                          # 测试文件
│   ├── __init__.py
│   ├── test_wechat_service.py
│   ├── test_article_service.py
│   └── conftest.py
├── docs/                           # 文档
│   ├── PRD.md
│   ├── API.md
│   └── DEPLOYMENT.md
├── downloads/                      # 下载文件存储目录
├── data/                           # 数据库文件目录
│   └── wechat_assistant.db
├── requirements.txt                # 依赖列表
├── pyproject.toml                  # 项目配置
├── .env.example                    # 环境变量示例
├── .gitignore
├── README.md
└── LICENSE
```

### 3.4 技术实现要点

#### 3.4.1 登录态管理
- 使用 Flask Session 存储微信登录态信息
- Cookie 持久化到本地文件
- 登录态过期检测与自动刷新机制

#### 3.4.2 并发处理
- 使用 ThreadPoolExecutor 实现并发下载
- 控制并发数量，避免资源占用过高
- 实现任务队列，支持暂停和恢复

#### 3.4.3 错误处理
- 统一异常处理中间件
- 详细的错误日志记录
- 友好的错误提示信息

#### 3.4.4 性能优化
- 数据库索引优化（fakeid, article_id）
- 查询结果缓存
- 分页查询减少内存占用
- 惰性加载文章内容

---

## 4. 数据模型（Database Schema）

### 4.1 公众号列表表（wechat_list）

**表名**: `wechat_list`

**表描述**: 存储微信公众号的基本信息和采集配置

| 字段名称        | 数据类型      | 约束           | 默认值       | 说明                                    |
|----------------|--------------|----------------|-------------|----------------------------------------|
| id             | INTEGER      | PRIMARY KEY    | AUTO        | 主键，自增                              |
| fakeid         | VARCHAR(100) | UNIQUE         | NULL        | 公众号唯一标识（微信系统 ID）             |
| nickname       | VARCHAR(50)  | NOT NULL       | NULL        | 公众号名称                              |
| alias          | VARCHAR(50)  |                | NULL        | 公众号别名（微信号）                     |
| round_head_img | VARCHAR(200) |                | NULL        | 公众号圆形头像 URL                       |
| service_type   | VARCHAR(10)  |                | NULL        | 公众号类型（0-订阅号，1-服务号，2-企业号）|
| signature      | VARCHAR(200) |                | NULL        | 公众号签名（简介）                       |
| verify_status  | VARCHAR(10)  |                | NULL        | 认证状态（0-未认证，1-已认证）           |
| memo           | VARCHAR(200) |                | NULL        | 备注信息                                |
| begin          | INTEGER      |                | 0           | 单页采集起始位置                         |
| count          | INTEGER      |                | 10          | 单页采集数量                            |
| collect_status | VARCHAR(50)  |                | 'pending'   | 采集状态（pending/collecting/completed/failed/paused）|
| create_time    | DATETIME     | NOT NULL       | CURRENT_TIMESTAMP | 记录创建时间                      |
| update_time    | DATETIME     | NOT NULL       | CURRENT_TIMESTAMP | 记录更新时间（自动更新）          |

**索引**:
- `idx_fakeid`: fakeid (UNIQUE)
- `idx_nickname`: nickname
- `idx_collect_status`: collect_status

**约束**:
- `check_count`: count >= 1 AND count <= 100

### 4.2 文章列表表（wechat_article_list）

**表名**: `wechat_article_list`

**表描述**: 存储公众号文章的详细信息

| 字段名称             | 数据类型      | 约束           | 默认值       | 说明                                    |
|---------------------|--------------|----------------|-------------|----------------------------------------|
| id                  | INTEGER      | PRIMARY KEY    | AUTO        | 主键，自增                              |
| wechat_list_id      | INTEGER      | FOREIGN KEY    | NOT NULL    | 关联公众号 ID                           |
| nickname            | VARCHAR(50)  |                | NULL        | 公众号名称（冗余字段，便于查询）          |
| article_id          | VARCHAR(50)  | UNIQUE         | NULL        | 文章唯一标识（微信系统 ID）              |
| article_title       | VARCHAR(200) |                | NULL        | 文章标题                                |
| article_cover       | VARCHAR(200) |                | NULL        | 文章封面图 URL                          |
| article_link        | VARCHAR(500) |                | NULL        | 文章链接                                |
| article_author_name | VARCHAR(50)  |                | NULL        | 文章作者名称                            |
| article_is_deleted  | VARCHAR(10)  |                | 'no'        | 文章是否已删除（yes/no）                 |
| article_create_time | DATETIME     |                | NULL        | 文章发布时间（来自微信）                 |
| article_update_time | DATETIME     |                | NULL        | 文章更新时间（来自微信）                 |
| is_downloaded       | VARCHAR(10)  |                | 'no'        | 是否已下载（yes/no）                    |
| create_time         | DATETIME     | NOT NULL       | CURRENT_TIMESTAMP | 记录创建时间                      |
| update_time         | DATETIME     | NOT NULL       | CURRENT_TIMESTAMP | 记录更新时间（自动更新）          |

**索引**:
- `idx_wechat_list_id`: wechat_list_id
- `idx_article_id`: article_id (UNIQUE)
- `idx_nickname`: nickname
- `idx_is_downloaded`: is_downloaded
- `idx_article_create_time`: article_create_time
- `idx_composite`: (wechat_list_id, article_create_time)

**外键**:
- `fk_wechat_list`: wechat_list_id REFERENCES wechat_list(id) ON DELETE CASCADE

**约束**:
- `check_is_deleted`: article_is_deleted IN ('yes', 'no')
- `check_is_downloaded`: is_downloaded IN ('yes', 'no')

### 4.3 数据库初始化 SQL

```sql
-- 创建公众号列表表
CREATE TABLE IF NOT EXISTS wechat_list (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    fakeid VARCHAR(100) UNIQUE,
    nickname VARCHAR(50) NOT NULL,
    alias VARCHAR(50),
    round_head_img VARCHAR(200),
    service_type VARCHAR(10),
    signature VARCHAR(200),
    verify_status VARCHAR(10),
    memo VARCHAR(200),
    begin INTEGER DEFAULT 0,
    count INTEGER DEFAULT 10,
    collect_status VARCHAR(50) DEFAULT 'pending',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CHECK (count >= 1 AND count <= 100)
);

-- 创建文章列表表
CREATE TABLE IF NOT EXISTS wechat_article_list (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    wechat_list_id INTEGER NOT NULL,
    nickname VARCHAR(50),
    article_id VARCHAR(50) UNIQUE,
    article_title VARCHAR(200),
    article_cover VARCHAR(200),
    article_link VARCHAR(500),
    article_author_name VARCHAR(50),
    article_is_deleted VARCHAR(10) DEFAULT 'no',
    article_create_time DATETIME,
    article_update_time DATETIME,
    is_downloaded VARCHAR(10) DEFAULT 'no',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (wechat_list_id) REFERENCES wechat_list(id) ON DELETE CASCADE,
    CHECK (article_is_deleted IN ('yes', 'no')),
    CHECK (is_downloaded IN ('yes', 'no'))
);

-- 创建索引
CREATE INDEX idx_wechat_fakeid ON wechat_list(fakeid);
CREATE INDEX idx_wechat_nickname ON wechat_list(nickname);
CREATE INDEX idx_wechat_collect_status ON wechat_list(collect_status);

CREATE INDEX idx_article_wechat_list_id ON wechat_article_list(wechat_list_id);
CREATE INDEX idx_article_article_id ON wechat_article_list(article_id);
CREATE INDEX idx_article_nickname ON wechat_article_list(nickname);
CREATE INDEX idx_article_is_downloaded ON wechat_article_list(is_downloaded);
CREATE INDEX idx_article_create_time ON wechat_article_list(article_create_time);
CREATE INDEX idx_article_composite ON wechat_article_list(wechat_list_id, article_create_time);

-- 创建触发器：自动更新 update_time
CREATE TRIGGER update_wechat_list_timestamp 
AFTER UPDATE ON wechat_list
FOR EACH ROW
BEGIN
    UPDATE wechat_list SET update_time = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;

CREATE TRIGGER update_article_list_timestamp 
AFTER UPDATE ON wechat_article_list
FOR EACH ROW
BEGIN
    UPDATE wechat_article_list SET update_time = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;
```

### 4.4 数据关系图

```
┌─────────────────────────────────┐
│        wechat_list              │
├─────────────────────────────────┤
│ • id (PK)                       │
│ • fakeid (UNIQUE)               │
│ • nickname                      │
│ • alias                         │
│ • round_head_img                │
│ • service_type                  │
│ • signature                     │
│ • verify_status                 │
│ • memo                          │
│ • begin                         │
│ • count                         │
│ • collect_status                │
│ • create_time                   │
│ • update_time                   │
└─────────────────────────────────┘
            │ 1
            │
            │ has many
            │
            │ N
┌─────────────────────────────────┐
│    wechat_article_list          │
├─────────────────────────────────┤
│ • id (PK)                       │
│ • wechat_list_id (FK)           │
│ • nickname                      │
│ • article_id (UNIQUE)           │
│ • article_title                 │
│ • article_cover                 │
│ • article_link                  │
│ • article_author_name           │
│ • article_is_deleted            │
│ • article_create_time           │
│ • article_update_time           │
│ • is_downloaded                 │
│ • create_time                   │
│ • update_time                   │
└─────────────────────────────────┘
```

---

## 5. API 端点（API Endpoints）

### 5.1 公众号管理接口

#### 5.1.1 获取公众号列表

**接口**: `GET /api/wechat/list`

**描述**: 获取所有公众号列表

**请求参数**:
| 参数名    | 类型   | 必填 | 说明           |
|----------|--------|------|---------------|
| page     | int    | 否   | 页码，默认 1   |
| per_page | int    | 否   | 每页数量，默认 20 |
| keyword  | string | 否   | 搜索关键词     |

**响应示例**:
```json
{
    "code": 200,
    "message": "success",
    "data": {
        "total": 50,
        "page": 1,
        "per_page": 20,
        "items": [
            {
                "id": 1,
                "fakeid": "MzI1NjE3NzQyMA==",
                "nickname": "技术博客",
                "alias": "tech_blog",
                "round_head_img": "https://...",
                "service_type": "0",
                "signature": "分享技术文章",
                "verify_status": "1",
                "memo": "优质技术公众号",
                "begin": 0,
                "count": 10,
                "collect_status": "pending",
                "article_count": 120,
                "create_time": "2026-01-01 10:00:00",
                "update_time": "2026-01-07 15:30:00"
            }
        ]
    }
}
```

#### 5.1.2 搜索公众号（微信接口）

**接口**: `POST /api/wechat/search`

**描述**: 通过微信公众平台接口搜索公众号

**请求体**:
```json
{
    "query": "公众号名称"
}
```

**响应示例**:
```json
{
    "code": 200,
    "message": "success",
    "data": {
        "need_login": false,
        "qrcode_url": null,
        "results": [
            {
                "fakeid": "MzI1NjE3NzQyMA==",
                "nickname": "技术博客",
                "alias": "tech_blog",
                "round_head_img": "https://...",
                "service_type": "0",
                "signature": "分享技术文章",
                "verify_status": "1"
            }
        ]
    }
}
```

**登录态失效响应**:
```json
{
    "code": 401,
    "message": "需要登录",
    "data": {
        "need_login": true,
        "qrcode_url": "https://mp.weixin.qq.com/qrcode/xxx"
    }
}
```

#### 5.1.3 新增公众号

**接口**: `POST /api/wechat/add`

**描述**: 添加新公众号

**请求体**:
```json
{
    "fakeid": "MzI1NjE3NzQyMA==",
    "nickname": "技术博客",
    "alias": "tech_blog",
    "round_head_img": "https://...",
    "service_type": "0",
    "signature": "分享技术文章",
    "verify_status": "1",
    "memo": "备注信息",
    "begin": 0,
    "count": 10
}
```

**响应示例**:
```json
{
    "code": 200,
    "message": "公众号添加成功",
    "data": {
        "id": 1
    }
}
```

#### 5.1.4 编辑公众号

**接口**: `PUT /api/wechat/{id}`

**描述**: 更新公众号信息

**请求体**:
```json
{
    "memo": "更新备注",
    "begin": 10,
    "count": 20
}
```

**响应示例**:
```json
{
    "code": 200,
    "message": "更新成功"
}
```

#### 5.1.5 删除公众号

**接口**: `DELETE /api/wechat/{id}`

**描述**: 删除公众号及其所有文章

**响应示例**:
```json
{
    "code": 200,
    "message": "删除成功"
}
```

#### 5.1.6 采集公众号文章

**接口**: `POST /api/wechat/{id}/collect`

**描述**: 采集指定公众号的文章

**请求体**:
```json
{
    "mode": "single",  // single: 单页采集, all: 全部采集
    "begin": 0,        // 可选，指定起始位置
    "count": 10        // 可选，指定采集数量
}
```

**响应示例**:
```json
{
    "code": 200,
    "message": "采集完成",
    "data": {
        "collected": 10,
        "next_begin": 10,
        "has_more": true
    }
}
```

**登录态失效响应**:
```json
{
    "code": 401,
    "message": "需要登录",
    "data": {
        "need_login": true,
        "qrcode_url": "https://mp.weixin.qq.com/qrcode/xxx"
    }
}
```

### 5.2 文章管理接口

#### 5.2.1 获取文章列表

**接口**: `GET /api/article/list`

**描述**: 获取文章列表

**请求参数**:
| 参数名          | 类型   | 必填 | 说明                          |
|----------------|--------|------|------------------------------|
| page           | int    | 否   | 页码，默认 1                  |
| per_page       | int    | 否   | 每页数量，默认 20             |
| wechat_list_id | int    | 否   | 筛选公众号 ID                 |
| keyword        | string | 否   | 搜索关键词（标题）             |
| is_downloaded  | string | 否   | 下载状态（yes/no/all），默认 all |
| is_deleted     | string | 否   | 删除状态（yes/no/all），默认 all |
| start_date     | string | 否   | 开始日期 YYYY-MM-DD           |
| end_date       | string | 否   | 结束日期 YYYY-MM-DD           |

**响应示例**:
```json
{
    "code": 200,
    "message": "success",
    "data": {
        "total": 500,
        "page": 1,
        "per_page": 20,
        "items": [
            {
                "id": 1,
                "wechat_list_id": 1,
                "nickname": "技术博客",
                "article_id": "123456789",
                "article_title": "Python 最佳实践",
                "article_cover": "https://...",
                "article_link": "https://mp.weixin.qq.com/s/xxx",
                "article_author_name": "张三",
                "article_is_deleted": "no",
                "article_create_time": "2026-01-05 10:00:00",
                "article_update_time": "2026-01-05 10:00:00",
                "is_downloaded": "no",
                "create_time": "2026-01-06 12:00:00",
                "update_time": "2026-01-06 12:00:00"
            }
        ]
    }
}
```

#### 5.2.2 下载文章

**接口**: `POST /api/article/download`

**描述**: 下载选中的文章

**请求体**:
```json
{
    "article_ids": [1, 2, 3, 4, 5],
    "format": "html",  // html 或 markdown
    "overwrite": false  // 是否覆盖已下载的文章
}
```

**响应示例**:
```json
{
    "code": 200,
    "message": "下载任务已创建",
    "data": {
        "task_id": "uuid-task-id",
        "total": 5
    }
}
```

#### 5.2.3 获取下载进度

**接口**: `GET /api/article/download/progress/{task_id}`

**描述**: 获取下载任务进度

**响应示例**:
```json
{
    "code": 200,
    "message": "success",
    "data": {
        "task_id": "uuid-task-id",
        "status": "processing",  // pending, processing, completed, failed
        "total": 5,
        "completed": 3,
        "failed": 0,
        "progress": 60
    }
}
```

#### 5.2.4 取消下载任务

**接口**: `DELETE /api/article/download/{task_id}`

**描述**: 取消正在进行的下载任务

**响应示例**:
```json
{
    "code": 200,
    "message": "任务已取消"
}
```

### 5.3 登录管理接口

#### 5.3.1 获取登录二维码

**接口**: `GET /api/login/qrcode`

**描述**: 获取微信公众平台登录二维码

**响应示例**:
```json
{
    "code": 200,
    "message": "success",
    "data": {
        "qrcode_url": "https://mp.weixin.qq.com/qrcode/xxx",
        "session_id": "session-uuid"
    }
}
```

#### 5.3.2 检查登录状态

**接口**: `GET /api/login/status/{session_id}`

**描述**: 轮询检查登录状态

**响应示例**:
```json
{
    "code": 200,
    "message": "success",
    "data": {
        "status": "success",  // waiting, success, timeout
        "token": "login-token-xxx"
    }
}
```

### 5.4 通用响应格式

**成功响应**:
```json
{
    "code": 200,
    "message": "success",
    "data": {}
}
```

**错误响应**:
```json
{
    "code": 400,
    "message": "错误描述",
    "errors": {
        "field": "字段错误信息"
    }
}
```

**状态码说明**:
- 200: 成功
- 400: 请求参数错误
- 401: 需要登录
- 404: 资源不存在
- 500: 服务器内部错误

---

## 6. 项目结构（Project Structure）

详见 **3.3 目录结构设计**

---

## 7. 页面设计（UI）

### 7.1 公众号管理页面

**路由**: `/`

**页面布局**:
```
┌─────────────────────────────────────────────────────────┐
│  Header: 微信公众号文章采集助手                           │
│  [公众号管理] [文章列表]                                  │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  [+ 新增公众号]  [搜索框]                [刷新]          │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  公众号列表                                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │ [头像] 公众号名称  @别名                           │  │
│  │        签名：这是一个技术分享公众号                 │  │
│  │        采集状态：待采集 | 文章数：120               │  │
│  │        [编辑] [采集] [采集全部] [删除]              │  │
│  ├───────────────────────────────────────────────────┤  │
│  │ [头像] 另一个公众号                                 │  │
│  │        ...                                         │  │
│  └───────────────────────────────────────────────────┘  │
│  [上一页] 1 2 3 4 5 [下一页]                            │
└─────────────────────────────────────────────────────────┘
```

**主要元素**:
- **新增公众号按钮**: 打开模态框
- **搜索框**: 实时搜索公众号名称
- **公众号卡片**: 显示公众号信息和操作按钮
  - 头像：圆形头像，120x120px
  - 名称：加粗显示
  - 别名：灰色小字
  - 签名：单行截断，鼠标悬停显示完整内容
  - 采集状态：标签形式显示
  - 文章数：显示已采集文章数量
  - 操作按钮：编辑、采集、采集全部、删除

**新增公众号模态框**:
```
┌─────────────────────────────────────────────┐
│  新增公众号                           [X]   │
├─────────────────────────────────────────────┤
│  [手工录入] [自动获取]                       │
│                                             │
│  自动获取：                                  │
│  ┌─────────────────────────────────────┐   │
│  │ 请输入公众号名称                     │   │
│  └─────────────────────────────────────┘   │
│  [搜索]                                     │
│                                             │
│  搜索结果：                                  │
│  ○ 技术博客 (@tech_blog) [已认证]          │
│  ○ 科技资讯 (@tech_news)                   │
│                                             │
│  或手工录入：                                │
│  公众号名称*: [________________]            │
│  公众号别名:  [________________]            │
│  Fake ID:     [________________]            │
│  备注:        [________________]            │
│  单页起始:    [0__]  单页数量: [10_]        │
│                                             │
│  [取消]                        [确定]       │
└─────────────────────────────────────────────┘
```

**登录二维码模态框**:
```
┌─────────────────────────────────────────────┐
│  微信公众平台登录                       [X] │
├─────────────────────────────────────────────┤
│                                             │
│       ┌───────────────────────┐            │
│       │                       │            │
│       │    [二维码图片]        │            │
│       │                       │            │
│       └───────────────────────┘            │
│                                             │
│  请使用微信扫码登录微信公众平台              │
│                                             │
│  [取消]                                     │
└─────────────────────────────────────────────┘
```

### 7.2 文章列表页面

**路由**: `/articles`

**页面布局**:
```
┌─────────────────────────────────────────────────────────┐
│  Header: 微信公众号文章采集助手                           │
│  [公众号管理] [文章列表]                                  │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  筛选条件：                                              │
│  公众号: [全部▼]  时间: [开始日期] - [结束日期]         │
│  状态: [全部▼]  关键词: [______]  [筛选] [重置]         │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  [☐ 全选]  [下载选中 (0)]  [批量删除]                    │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│  文章列表                                                │
│  ┌───────────────────────────────────────────────────┐  │
│  │ ☐ [封面] Python 最佳实践                           │  │
│  │          技术博客 · 张三 · 2026-01-05              │  │
│  │          [已下载] [查看] [下载] [删除]              │  │
│  ├───────────────────────────────────────────────────┤  │
│  │ ☐ [封面] Flask 框架入门                            │  │
│  │          技术博客 · 李四 · 2026-01-04              │  │
│  │          [未下载] [查看] [下载] [删除]              │  │
│  └───────────────────────────────────────────────────┘  │
│  [上一页] 1 2 3 4 5 [下一页]  每页: [20▼]条            │
└─────────────────────────────────────────────────────────┘
```

**主要元素**:
- **筛选区域**:
  - 公众号下拉选择
  - 日期范围选择器
  - 下载状态筛选
  - 关键词搜索
- **操作栏**:
  - 全选复选框
  - 批量下载按钮（显示选中数量）
  - 批量删除按钮
- **文章卡片**:
  - 复选框
  - 文章封面（缩略图，150x150px）
  - 文章标题（可点击打开原文）
  - 公众号名称、作者、发布时间
  - 下载状态标签
  - 操作按钮：查看、下载、删除

**下载进度模态框**:
```
┌─────────────────────────────────────────────┐
│  下载文章                               [X] │
├─────────────────────────────────────────────┤
│                                             │
│  正在下载文章，请稍候...                    │
│                                             │
│  进度: [████████░░░░░░░░] 60%              │
│                                             │
│  已完成: 3/5                                │
│  失败: 0                                    │
│                                             │
│  [取消]                                     │
└─────────────────────────────────────────────┘
```

### 7.3 样式设计

**色彩方案**:
- 主色调: `#07C160` (微信绿)
- 辅助色: `#10AEFF` (信息蓝)
- 警告色: `#FFBE00` (警告黄)
- 错误色: `#FA5151` (错误红)
- 背景色: `#F7F8FA` (浅灰)
- 文字色: `#191919` (深灰)
- 次要文字色: `#8C8C8C` (灰色)

**字体**:
- 主字体: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif
- 字号: 标题 18px, 正文 14px, 辅助 12px

**组件样式**:
- 按钮: 圆角 4px, 高度 36px
- 输入框: 圆角 4px, 高度 36px, 边框 1px solid #E5E5E5
- 卡片: 圆角 8px, 阴影 0 2px 8px rgba(0,0,0,0.1)
- 模态框: 圆角 8px, 最大宽度 600px

**响应式设计**:
- 桌面端: >= 1024px 宽度
- 平板端: 768px - 1023px
- 移动端: < 768px

---

## 8. 非功能性需求

### 8.1 性能要求

- **响应时间**:
  - 页面加载时间: < 2 秒
  - API 响应时间: < 1 秒
  - 采集单页文章: < 5 秒
  - 下载单篇文章: < 10 秒

- **并发处理**:
  - 支持同时采集 3 个公众号
  - 支持同时下载 5 篇文章
  - 单个公众号最多 10,000 篇文章

- **数据库性能**:
  - 查询响应时间: < 100ms
  - 支持 10,000+ 公众号记录
  - 支持 100,000+ 文章记录

### 8.2 可靠性

- **错误处理**:
  - 网络错误自动重试（最多 3 次）
  - 数据库操作事务保护
  - 详细的错误日志记录

- **数据完整性**:
  - 外键约束确保数据一致性
  - 采集过程中断后可恢复
  - 文章去重机制避免重复

- **容错能力**:
  - 登录态失效自动触发重新登录
  - 接口限流时自动降速
  - 单篇文章下载失败不影响其他文章

### 8.3 可用性

- **用户友好**:
  - 简洁直观的界面设计
  - 清晰的操作反馈
  - 友好的错误提示信息

- **易用性**:
  - 无需复杂配置即可使用
  - 提供操作指引和帮助文档
  - 键盘快捷键支持（可选）

### 8.4 可维护性

- **代码质量**:
  - 遵循 PEP 8 编码规范
  - 使用 Ruff 进行代码格式化
  - 使用 mypy 进行类型检查
  - 单元测试覆盖率 > 80%

- **文档完善**:
  - 代码注释清晰
  - API 文档完整
  - 部署文档详细
  - 用户手册清楚

- **日志系统**:
  - 分级日志记录（DEBUG, INFO, WARNING, ERROR）
  - 日志文件按日期轮转
  - 关键操作审计日志

### 8.5 安全性

- **数据安全**:
  - 敏感信息（Cookie/Token）加密存储
  - 本地数据库文件权限控制
  - 不存储用户微信密码

- **接口安全**:
  - CSRF 防护（Flask 内置）
  - 请求频率限制
  - 参数验证和过滤

- **隐私保护**:
  - 不收集用户个人信息
  - 数据仅存储在本地
  - 支持数据清除功能

### 8.6 兼容性

- **浏览器支持**:
  - Chrome 90+
  - Firefox 88+
  - Safari 14+
  - Edge 90+

- **操作系统**:
  - Windows 10/11
  - macOS 11+
  - Linux (Ubuntu 20.04+)

- **Python 版本**:
  - Python 3.12+

### 8.7 可扩展性

- **架构设计**:
  - 模块化设计，低耦合高内聚
  - 服务层与数据层分离
  - 易于添加新功能

- **数据库扩展**:
  - 预留数据库迁移方案
  - 支持切换到 MySQL/PostgreSQL（可选）

- **功能扩展**:
  - 支持添加新的文章下载格式
  - 支持扩展新的导出功能
  - 预留 API 接口供第三方集成

### 8.8 部署要求

- **部署方式**:
  - 本地单机部署
  - 支持 Docker 容器化部署（可选）

- **资源要求**:
  - CPU: 2 核+
  - 内存: 2GB+
  - 磁盘: 10GB+（根据文章数量增长）

- **依赖环境**:
  - Python 3.12+
  - 无需额外数据库服务
  - 需要网络访问微信公众平台

---

## 9. 项目里程碑

### 9.1 开发阶段

**阶段一: 基础框架搭建（Week 1-2）**
- [ ] 项目结构初始化
- [ ] 数据库模型定义
- [ ] Flask 应用基础配置
- [ ] 基础路由和模板搭建

**阶段二: 核心功能开发（Week 3-5）**
- [ ] 公众号管理功能
- [ ] 微信登录态管理
- [ ] 文章采集功能
- [ ] 文章列表展示

**阶段三: 下载功能开发（Week 6-7）**
- [ ] 单篇文章下载
- [ ] 批量下载功能
- [ ] 下载进度显示
- [ ] 命令行工具开发

**阶段四: 测试和优化（Week 8）**
- [ ] 单元测试编写
- [ ] 集成测试
- [ ] 性能优化
- [ ] Bug 修复

**阶段五: 文档和发布（Week 9）**
- [ ] 用户文档编写
- [ ] API 文档完善
- [ ] 部署指南
- [ ] 发布 v1.0

### 9.2 后续规划

**v1.1 功能增强**
- 文章内容全文搜索
- 数据导出功能（Excel, CSV）
- 文章标签分类

**v1.2 体验优化**
- 暗色主题支持
- 批量操作撤销功能
- 更多文件格式支持（PDF, Markdown）

**v2.0 高级功能**
- 多用户支持
- 云端备份
- 文章智能分类
- 定时采集任务

---

## 10. 风险与挑战

### 10.1 技术风险

**微信接口变更**
- 风险: 微信公众平台可能更改接口或限制访问
- 应对: 建立接口监控，及时适配变更

**登录态管理**
- 风险: Cookie/Token 失效机制不稳定
- 应对: 实现健壮的登录态检测和刷新机制

**大规模数据处理**
- 风险: 大量文章采集和下载可能影响性能
- 应对: 实现分页、缓存、异步处理

### 10.2 业务风险

**频率限制**
- 风险: 频繁请求可能被微信限制
- 应对: 实现智能限速和间隔控制

**数据完整性**
- 风险: 采集中断导致数据不完整
- 应对: 实现断点续传和数据校验

### 10.3 法律合规

**版权问题**
- 风险: 批量下载可能涉及版权问题
- 应对: 
  - 明确声明仅供个人学习研究使用
  - 不提供公开分享和商业使用功能
  - 用户需自行承担使用责任

**隐私保护**
- 风险: 存储用户登录信息
- 应对: 
  - 本地加密存储
  - 明确隐私政策
  - 提供数据清除功能

---

## 11. 附录

### 11.1 术语表

| 术语 | 说明 |
|-----|------|
| fakeid | 微信公众号唯一标识符 |
| 登录态 | 用户登录后的会话状态，通常通过 Cookie/Token 维护 |
| 采集 | 获取公众号历史文章列表的过程 |
| 单页采集 | 一次采集指定数量的文章 |
| 全部采集 | 循环采集直到获取所有历史文章 |
| ORM | Object-Relational Mapping，对象关系映射 |

### 11.2 参考资料

- Flask 官方文档: https://flask.palletsprojects.com/
- SQLAlchemy 文档: https://docs.sqlalchemy.org/
- Tailwind CSS 文档: https://tailwindcss.com/docs
- Ruff 文档: https://docs.astral.sh/ruff/
- pytest 文档: https://docs.pytest.org/

### 11.3 更新记录

| 版本 | 日期 | 更新内容 | 作者 |
|-----|------|---------|------|
| 1.0 | 2026-01-07 | 初始版本 | Product Team |

---

**文档结束**