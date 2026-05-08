# 筋斗云旅游助手
筋斗云是一个基于 **FastAPI + LangChain** 的本地网页智能助手，主要用于天气查询和旅行辅助。它支持当前天气、未来 5 天天气趋势、景点推荐、美食推荐、酒店查询、路线规划、预算估算、历史会话、长期记忆和附件上传。
## 一、项目整体作用

这个项目的目标是把一个命令行天气助手扩展成可在浏览器中使用的聊天式旅行助手。

用户可以在网页中输入自然语言，例如：

```text
南京今天天气怎么样？未来 5 天适合去哪玩？
上海有什么美食和酒店推荐？
从南京站到夫子庙怎么走？
北京 3 天 2 人舒适旅行预算大概多少？
记住我喜欢高铁出行
```

系统会根据问题调用对应工具，返回天气、旅行、路线、预算等结果。

------

## 二、目录结构说明

项目主要结构如下：

```text
weather/
  .env                         # 本地环境变量，保存 API Key，不应上传 GitHub
  .gitignore                   # Git 忽略规则
  .python-version              # Python 版本提示
  pyproject.toml               # 项目依赖配置
  uv.lock                      # uv 锁定依赖版本
  uv.toml                      # uv 配置
  README.md                    # 项目说明文档
  GITHUB_DEPLOY_NOTES.md       # GitHub 发布整理建议
  weather/                     # 核心 Python 包
    __init__.py                # 包入口
    agent.py                   # 智能体创建逻辑
    tools.py                   # 天气、旅行相关工具函数
    storage.py                 # SQLite 数据持久化
    web.py                     # FastAPI 网页服务入口
    cli.py                     # 命令行聊天入口
    ui/                        # 前端页面
      index.html               # 页面结构
      styles.css               # 页面样式
      app.js                   # 前端交互逻辑
    test/                      # 单元测试
      test_tools.py            # 工具函数测试
      test_storage.py          # 存储功能测试
    .data/                     # 运行时数据，自动生成/保存
      conversations.sqlite3    # 会话、消息、记忆、附件记录数据库
      uploads/                 # 上传附件文件
```

------

## 三、根目录文件作用

### `.env`

本地环境变量文件，保存模型和地图服务 API Key。

常见内容包括：

```text
DEEPSEEK_API_KEY=你的 DeepSeek Key
AMAP_API_KEY=你的高德地图 Key
```

作用：

- `DEEPSEEK_API_KEY`：用于调用大模型。
- `AMAP_API_KEY`：用于景点、美食、酒店、路线规划等高德地图接口。
- 可选的 `WEATHER_AGENT_MODEL`：用于指定模型名称，默认是 `deepseek-chat`。

注意：

```text
.env 不能上传 GitHub
```

因为里面可能包含私密 Key。

------

### `.gitignore`

用于告诉 Git 哪些文件不需要上传。

建议忽略：

```text
.env
.venv/
weather/.data/
weather/test_data/
__pycache__/
*.sqlite3
*.log
```

作用：

- 避免上传密钥。
- 避免上传本地聊天历史。
- 避免上传虚拟环境和缓存。
- 保持 GitHub 仓库干净。

------

### `.python-version`

记录建议使用的 Python 版本。

当前项目使用 Python 3.14 相关环境。别人克隆项目后，可以根据这个文件选择合适解释器。

------

### `pyproject.toml`

Python 项目配置文件，记录项目名称、版本和依赖。

当前项目依赖中包含：

- `fastapi`：Web 后端框架。
- `uvicorn`：FastAPI 服务启动器。
- `langchain`：智能体框架。
- `langchain-deepseek`：DeepSeek 模型接入。
- `python-dotenv`：读取 `.env` 文件。
- `requests`：调用天气和地图 HTTP 接口。

如果依赖爆红，通常是 PyCharm 没选对解释器，或者没有执行依赖安装。

------

### `uv.lock`

`uv` 生成的依赖锁文件。

作用：

- 锁定依赖版本。
- 保证不同电脑安装出来的依赖尽量一致。
- 适合上传 GitHub。

------

### `uv.toml`

`uv` 的配置文件。

当前主要用于配置 uv 缓存目录。

如果别人电脑路径不同，这个文件不是核心运行逻辑；必要时可以调整或删除。

------

### `GITHUB_DEPLOY_NOTES.md`

这是发布 GitHub 前的整理建议文档。

作用：

- 记录哪些文件应该上传。
- 记录哪些文件不应上传。
- 提供 `.gitignore`、`.env.example`、README、Git 命令建议。

它不参与程序运行，只是辅助文档。

------

## 四、`weather/` 核心包说明

### `weather/__init__.py`

Python 包入口文件。

作用：

- 让 `weather` 成为可导入的 Python 包。
- 对外暴露常用函数，例如：
  - `create_weather_agent`
  - `get_weather`
  - `get_attractions`
  - `get_food_recommendations`
  - `get_hotels`
  - `plan_route`
  - `estimate_budget`

例如其它代码可以这样导入：

```python
from weather import get_weather
```

------

### `weather/agent.py`

智能体创建模块。

主要作用：

- 读取 `.env` 环境变量。
- 创建 LangChain Agent。
- 注册所有工具函数。
- 设置系统提示词。
- 指定模型，例如默认的 `deepseek-chat`。

核心函数：

```python
create_weather_agent(model: str | None = None)
```

它会创建一个具备以下工具能力的智能体：

- `get_weather`
- `get_attractions`
- `get_food_recommendations`
- `get_hotels`
- `plan_route`
- `estimate_budget`

系统提示词会约束智能体：

- 用户问天气时必须调用天气工具。
- 用户问景点时调用景点工具。
- 用户问美食时调用美食工具。
- 用户问酒店时调用酒店工具。
- 用户问路线时调用路线工具。
- 用户问预算时调用预算工具。
- 回答天气时需要包含当天实时天气和未来 5 天天气趋势。

------

### `weather/tools.py`

工具函数模块，是项目的业务能力核心。

它不直接负责网页，也不负责数据库，只负责“查询和计算”。

主要工具如下。

#### `get_weather(city)`

作用：查询城市天气。

功能包括：

- 查询当前天气。
- 查询未来 5 天天气趋势。
- 返回温度、体感温度、湿度、风速、天气描述等。
- 根据天气给出出游建议。

依赖接口：

- `wttr.in`
- `Open-Meteo`

------

#### `get_attractions(city)`

作用：推荐城市景点。

依赖：

```text
AMAP_API_KEY
```

使用高德地图地点搜索接口，根据城市查询旅游景点。

------

#### `get_food_recommendations(city)`

作用：推荐美食或餐厅。

依赖：

```text
AMAP_API_KEY
```

使用高德地图 POI 搜索餐饮类型数据。

------

#### `get_hotels(city)`

作用：查询酒店推荐。

依赖：

```text
AMAP_API_KEY
```

使用高德地图 POI 搜索酒店住宿类型数据。

------

#### `plan_route(origin, destination, city='', mode='driving')`

作用：路线规划。

参数说明：

- `origin`：起点。
- `destination`：终点。
- `city`：城市，可选。
- `mode`：出行方式，例如 driving、walking、transit。

功能：

- 先把起点和终点转换为经纬度。
- 再调用高德路线接口。
- 返回距离和耗时。

------

#### `estimate_budget(city, days=1, people=1, level='舒适')`

作用：估算旅行预算。

参数说明：

- `city`：城市。
- `days`：旅行天数。
- `people`：人数。
- `level`：预算档位，例如经济、舒适、豪华。

它是本地估算工具，不依赖外部接口。

------

### `weather/storage.py`

本地数据持久化模块。

它使用 SQLite 保存：

- 会话列表
- 聊天消息
- 上传附件记录
- 长期记忆

默认数据库位置：

```text
weather/.data/conversations.sqlite3
```

默认附件目录：

```text
weather/.data/uploads/
```

主要数据表：

#### `conversations`

保存会话。

字段包括：

- `id`
- `title`
- `created_at`
- `updated_at`

用途：左侧历史会话列表。

------

#### `messages`

保存聊天消息。

字段包括：

- `id`
- `conversation_id`
- `role`
- `content`
- `created_at`

用途：切换会话后恢复聊天记录，并作为上下文传给智能体。

------

#### `attachments`

保存附件元数据。

字段包括：

- `id`
- `conversation_id`
- `message_id`
- `original_name`
- `stored_name`
- `content_type`
- `size`
- `created_at`

真实文件保存在：

```text
weather/.data/uploads/
```

用途：图片、视频、文件上传后可以在历史消息中重新展示。

------

#### `memories`

保存长期记忆。

字段包括：

- `id`
- `content`
- `created_at`
- `updated_at`

用途：跨会话保存用户偏好，例如：

```text
用户称呼：小明
用户喜欢：高铁出行
用户常住地：南京
```

删除会话不会删除长期记忆。只有点击记忆旁边的删除按钮，才会删除对应记忆。

------

### `weather/web.py`

FastAPI 网页服务入口，是网页版本的核心文件。

启动时使用：

```powershell
python -m uvicorn weather.web:app --host 127.0.0.1 --port 8000
```

主要职责：

- 创建 FastAPI 应用。
- 提供首页 HTML。
- 托管静态前端资源。
- 托管上传附件目录。
- 提供聊天 API。
- 提供历史会话 API。
- 提供长期记忆 API。
- 提供附件上传 API。
- 调用 LangChain Agent 生成回复。

主要接口：

#### `GET /`

返回前端首页：

```text
weather/ui/index.html
```

------

#### `GET /api/conversations`

返回历史会话列表。

用于左侧历史会话栏。

------

#### `POST /api/conversations`

新建会话。

------

#### `GET /api/conversations/{conversation_id}/messages`

读取某个会话的历史消息。

切换会话时使用。

------

#### `DELETE /api/conversations/{conversation_id}`

删除会话。

删除后数据库中的该会话消息也会删除。

------

#### `GET /api/memories`

读取长期记忆列表。

用于左侧“记忆”区域展示。

------

#### `DELETE /api/memories/{memory_id}`

删除单条长期记忆。

------

#### `POST /api/uploads`

上传附件。

支持：

- 图片
- 视频
- PDF
- 文本文件
- Word/Excel/PPT 等常见文件

限制：

- 单个文件最大 25MB。
- 一次最多 8 个文件。

------

#### `POST /api/chat`

聊天接口。

前端发送用户消息后，请求这个接口。

处理流程：

1. 获取用户输入。
2. 如果有附件，关联附件记录。
3. 判断是否需要写入长期记忆。
4. 保存用户消息到数据库。
5. 读取当前会话完整历史。
6. 读取长期记忆并注入给智能体。
7. 调用 LangChain Agent。
8. 保存助手回复。
9. 返回回复给前端。

------

### `weather/cli.py`

命令行聊天入口。

作用：

- 不打开网页，直接在终端中和智能体聊天。
- 适合调试 Agent 和工具。

运行方式示例：

```powershell
python -m weather.cli
```

当前主要使用网页版本，CLI 是辅助入口。

------

## 五、前端目录 `weather/ui/` 说明

### `weather/ui/index.html`

网页结构文件。

负责定义页面骨架：

- 左侧栏
- 品牌区域
- 新会话按钮
- 快捷问题按钮
- 长期记忆区域
- 历史会话区域
- 聊天消息区域
- 输入框
- 附件上传按钮
- 发送按钮

它只定义结构，不负责复杂逻辑。

------

### `weather/ui/styles.css`

页面样式文件。

负责：

- 整体布局
- 左侧栏样式
- 聊天气泡样式
- 用户/助手消息样式
- 筋斗云图标样式
- Markdown 内容样式
- 附件预览样式
- 记忆列表样式
- 响应式移动端适配

如果要改界面颜色、间距、按钮样式，主要改这个文件。

------

### `weather/ui/app.js`

前端交互逻辑文件。

负责：

- 发送消息
- 渲染聊天记录
- 调用 `/api/chat`
- 加载历史会话
- 切换会话
- 删除会话
- 加载长期记忆
- 删除长期记忆
- 上传附件
- 展示附件预览
- Markdown 渲染
- 输入框自动调整高度
- 状态提示，例如“思考中”“就绪”

它是前端最核心的文件。

------

## 六、测试目录 `weather/test/` 说明

### `test_tools.py`

测试工具函数。

覆盖内容包括：

- 天气查询格式化
- 城市为空时的错误提示
- 缺少 AMap Key 时的提示
- 景点、美食、酒店 POI 格式化
- 路线规划格式化
- 预算估算格式化

这些测试不会真实请求外部接口，而是使用 mock 模拟接口返回。

------

### `test_storage.py`

测试 SQLite 存储逻辑。

覆盖内容包括：

- 创建会话
- 添加消息
- 读取消息
- 删除会话
- 附件关联
- 长期记忆新增、去重、删除

------

## 七、运行时数据 `.data/` 说明

目录：

```text
weather/.data/
```

这个目录不是代码，是程序运行后产生的数据。

里面通常有：

```text
conversations.sqlite3
uploads/
```

### `conversations.sqlite3`

SQLite 数据库。

保存：

- 历史会话
- 聊天消息
- 长期记忆
- 附件记录

### `uploads/`

保存用户上传的文件。

例如图片、视频、文本文件。

注意：

```text
weather/.data/ 不建议上传 GitHub
```

因为里面可能包含用户聊天记录、记忆和私人附件。

------

## 八、安装依赖

推荐使用 uv：

```powershell
cd E:\pythonstudy\PycharmProjects\weather
uv sync
```

如果不用 uv，可以手动安装：

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install fastapi uvicorn langchain langchain-deepseek python-dotenv requests
```

如果 PyCharm 里依赖爆红，通常不是代码问题，而是解释器没选对。

PyCharm 解释器应选择：

```text
E:\pythonstudy\PycharmProjects\weather\.venv\Scripts\python.exe
```

------

## 九、启动网页

在 PowerShell 中运行：

```powershell
cd E:\pythonstudy\PycharmProjects\weather
.\.venv\Scripts\python.exe -m uvicorn weather.web:app --host 127.0.0.1 --port 8000
```

浏览器打开：

```text
http://127.0.0.1:8000
```

如果 8000 端口被占用：

```powershell
.\.venv\Scripts\python.exe -m uvicorn weather.web:app --host 127.0.0.1 --port 8001
```

然后打开：

```text
http://127.0.0.1:8001
```

------

## 十、关闭服务器

如果是在当前 PowerShell 里启动的，按：

```text
Ctrl + C
```

如果是后台运行，先查端口：

```powershell
netstat -ano | Select-String ":8000"
```

找到最后一列 PID 后关闭：

```powershell
Stop-Process -Id 进程ID
```

------

## 十一、PyCharm 启动配置

在 PyCharm 中打开：

```text
Edit Configurations...
```

新建 Python 配置：

```text
Name: 筋斗云 Web
Script path: E:\pythonstudy\PycharmProjects\weather\.venv\Scripts\python.exe
Parameters: -m uvicorn weather.web:app --host 127.0.0.1 --port 8000
Working directory: E:\pythonstudy\PycharmProjects\weather
Environment variables: 留空
Paths to .env files: 留空
```

注意：

不要把 `uvicorn.exe` 填到 `Paths to .env files` 中。否则 PyCharm 会把 exe 当成环境变量文件读取，出现乱码环境变量错误。
