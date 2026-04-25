# drpy-node MCP 工具能力图谱

> 来源：drpy-node-mcp 源码分析 (`index.js`, `tools/` 目录)

## 一、MCP 服务器架构

- **入口**: `index.js` — 基于 `@modelcontextprotocol/sdk` 实现
- **传输模式**: stdio（默认）/ SSE / StreamableHTTP
- **认证**: 可选 `MCP_TOKEN` Bearer Token
- **路由**: `ListToolsRequestSchema` → 返回工具列表；`CallToolRequestSchema` → 按名分发
- **定义文件**: `tools/toolDefinitions.js` — 所有工具的 Zod schema 和 handler 注册

## 二、工具分类全景

工具分为 7 大类，共 25 个工具：

| 类别 | 工具数 | 核心职责 |
|------|--------|---------|
| 文件操作 | 6 | 读/写/改/删/搜/列 drpy 项目文件 |
| 爬虫分析 | 9 | 网站分析、模板判断、规则调试、验证 |
| 引擎测试 | 2 | 单接口测试、全流程评估 |
| 仓库管理 | 2 | 仓库验证、文件 CRUD |
| 系统管理 | 3 | 配置 R/W、日志、重启 |
| API 信息 | 1 | drpy API 接口列表 |
| 数据库 | 1 | 只读 SQL 查询 |

> **命名约定**: 所有文件操作工具均以 `drpy_` 前缀命名，以区别于 IDE 内置的通用文件操作工具。非 drpy 场景请使用 IDE 原生工具。

## 三、文件操作工具

文件操作工具由 `drpy-node-mcp/tools/fsTools.js` 实现；网站分析、模板猜测、规则调试、结构验证和 resolved rule 等由 `spiderTools.js` 实现。不要把两者混淆。

### `drpy_read_file(path)`
- **核心能力**: 读取 drpy 项目中任意文件
- **特别机制**: 自动检测 DS 加密格式并解密
- **使用场景**: 读源文件看 rule 配置；读配置文件
- **限制**: 仅限 drpy-node 项目路径

### `drpy_write_file(path, content)`
- **核心能力**: 写入/覆盖文件
- **使用场景**: 创建新源文件、修改配置
- **限制**: 禁止写入 `.md/.txt` 等文档文件

### `drpy_edit_file(path, operation, ...)`
- **核心能力**: 精确修改文件（replace_text / replace_lines / delete_lines / insert_lines）
- **安全机制**: JS 文件编辑后自动语法校验，失败则回滚
- **使用场景**: 修改已有源的特定字段或函数
- **推荐**: `replace_text` 模式（按文本搜索，无需行号）

### `drpy_delete_file(path)`
- 删除文件或目录，仅限 drpy 项目内

### `drpy_find_in_file(path, keyword, regex?)`
- 在文件中搜索文本/正则，返回行号+上下文
- **配合**: 定位代码位置后用 `drpy_edit_file` 的行号操作

### `drpy_list_directory(path?)`
- 列出项目目录中的文件和子目录
- **典型用法**: 浏览 spider/js 目录结构

## 四、爬虫分析工具

### 网站分析类

#### `analyze_website_structure(url, options?)`
- 发起 HTTP 请求 → cheerio 解析 → 移除 script/style/meta 等 → 返回精简 DOM
- **核心价值**: 极大降低 token 消耗，让 AI 快速分析 CSS 选择器
- **数据截断**: 返回 body HTML 截断至 15000 字符
- **典型用法**: 配合站点分析，提取列表/详情/搜索的 CSS 选择器

#### `guess_spider_template(url, options?)`
- 请求目标 URL → 与内置 12 个模板的 `class_parse` 规则匹配 → 判断 CMS 类型
- **返回值**: mx / mxpro / 首图 / 海螺3 / 不要套用模板 等
- **边界**: 这是启发式命中，只证明分类结构相似，不保证 `url/searchUrl/detail/lazy` 都匹配
- **典型用法**: 新建源的第一步，决定走路线 A（模板继承）/ B（签名API）/ C（SPA）

#### `fetch_spider_url(url, options?)`
- 使用 drpy 内置请求库 (`req`) 获取 URL 内容
- **核心价值**: 测试网站连通性、调试请求头/UA/Cookie 等反爬机制
- **典型用法**: 验证 API 接口返回格式；检查是否需要特定 header

#### `extract_website_filter(url, urls?, gzip?, options?)`
- 分析分类页面 DOM，提取筛选条件（地区/年份/类型等）
- **支持**: 单 URL 或批量 URL；gzip+base64 压缩输出；可识别常见 CMS filter block，也会从 query 参数中归纳筛选
- **典型用法**: 生成 filter 配置，用于有筛选需求的源

#### `extract_iframe_src(url, options?)`
- 抓取播放页 → 提取 iframe 的 src 属性
- **典型用法**: 辅助编写 lazy 函数，定位真实的播放源地址

### 规则调试类

#### `debug_spider_rule(html/url, rule, mode, baseUrl?, options?)`
- 三种模式:
  - `pdfa`: 获取列表数组（测试 CSS 选择器命中哪些元素）
  - `pdfh`: 提取节点文本/属性
  - `pd`: 提取并标准化 URL
- **重要边界**: `pdfa` 模式应传纯 CSS selector（如 `.list li`），不要传完整分号规则（如 `.list li;a&&Text;img&&src;...`）
- **典型用法**: 在写源时验证选择器是否正确，减少反复 write → test 的迭代

### 模板与验证类

#### `get_spider_template()`
- 返回标准 JS 源文件模板（`@header` + `var rule = {}` 骨架）
- **典型用法**: 新建源的起点骨架

#### `get_claw_ds_skill(lang?)`
- 返回 MCP 侧的写源指南（`skills.md` 或 `skills-zh.md`）
- **典型用法**: 在写源前获取最新的编写规范和格式说明

#### `drpy_check_syntax(path)`
- 使用 Node.js 语法检查器验证 JS 文件语法
- **典型用法**: 每次 write/edit 后调用，快速发现语法错误

#### `validate_spider(path)`
- 语法检查 + rule 对象结构验证（必填字段、字段类型、值合法性）
- **证据层级**: L1 结构验证，只能说明源形态基本合法，不能证明接口真实可用
- **典型用法**: 比 `drpy_check_syntax` 更全面的验证

#### `get_resolved_rule(path)`
- 获取模板继承处理后的最终 rule 摘要
- **典型用法**: 排查模板继承后的实际生效字段

### 信息查询类

#### `list_sources()`
- 列出 `spider/js/` 和 `spider/catvod/` 下的所有源文件
- **典型用法**: 查看有哪些源、确认源名

#### `get_routes_info()`
- 获取已注册的 API 路由和控制器
- **典型用法**: 了解项目 API 入口

#### `get_drpy_libs_info()`
- 获取沙箱中可用的全局函数库信息（pdfh/pd/req/local 等）
- **典型用法**: 写源时查阅可用工具函数

#### `get_drpy_api_list()`
- 获取 drpy API 完整接口列表（含参数和返回示例）
- **典型用法**: 了解框架提供的标准 API

## 五、引擎测试工具

### `test_spider_interface(source_name, interface, ...)`
- **引擎**: 通过 `localDsCore` → `globalThis.getEngine()` 调用真实引擎
- **支持的接口**: `home` / `category` / `detail` / `search` / `play`
- **各接口参数**:
  - `home`: 无额外参数 → 返回分类 + 推荐列表
  - `category`: `class_id`(分类ID), `ext`(筛选参数 base64) → 返回列表
  - `detail`: `ids`(影片ID, 逗号分隔) → 返回详情
  - `search`: `keyword`(关键词) → 返回搜索结果
  - `play`: `play_url`(播放链接), `flag`(来源标识) → 返回播放信息
- **返回结构**: `{ interface, success, data_preview, item_count/class_count, error, duration_ms }`
- **数据截断**: 返回数据截断至 3000 字符
- **典型用法**: 
  - 逐接口排查问题
  - 修源后验证修复效果
  - 确认 detail 稳定产出 `vod_play_from/vod_play_url` 后再测 play

### `evaluate_spider_source(source_name, class_id?, keyword?, timeout_seconds?)`
- **全流程自动化测试**: 首页(20分) → 一级(20分) → 二级(25分) → 播放(25分) → 搜索(10分) = 100分
- **自动串联**: 从上级结果自动提取下级所需的分类ID/影片ID/播放URL
- **评分标准**: 一级+二级+播放均通过 = 源有效
- **超时控制**: 默认 120 秒
- **典型用法**: 
  - 新建源后的完整验证
  - 修源后的回归测试
  - 上传前评估

### 证据层级与工具边界

| 层级 | 工具 | 结论边界 |
|---|---|---|
| L1 | `drpy_check_syntax` + `validate_spider` | 语法/结构合法，不代表运行时可用 |
| L2 | `test_spider_interface` | 单接口经真实 `localDsCore/getEngine` 验证通过或失败 |
| L3 | `evaluate_spider_source` | 首页→一级→二级→播放→搜索自动串联评分 |

`test_spider_interface` / `evaluate_spider_source` 都调用真实引擎；结论强度高于静态结构检查。上传或“源已修好”这类结论应至少说明采用了哪一层证据。

## 六、仓库管理工具

### `house_verify()`
- 检查 `config/env.json` 中的 `HOUSER_URL` 和 `HOUSE_TOKEN`
- 测试与仓库的连通性
- **典型用法**: 仓库操作前的第一步

### `house_file(action, ...)`
- **7 种操作**:
  | action | 必填参数 | 功能 |
  |--------|---------|------|
  | `list` | — | 文件列表（支持 search/tag/page/limit/uploader 筛选） |
  | `upload` | path, tags?, is_public? | 上传文件（自动检测同名替换） |
  | `replace` | file_id, path | 按 ID 替换已有文件 |
  | `delete` | file_id | 删除文件 |
  | `info` | cid | 查询文件元数据 |
  | `toggle_visibility` | file_id | 切换公开/私密 |
  | `update_tags` | file_id, tags | 更新标签 |

- **自动标签检测**: 源码中内置了 `detectSourceTypeInfo()` 自动识别类型标签 (`ds`/`catvod`/`php`/`hipy`/`json`/`优`/`jx`)，上传时可能自动检测/合并标签；用户明确指定标签时应以用户要求为准，上传后用 `info` 核验元数据
- **MIME 映射**: 内置常见文件类型的 MIME 映射表
- **典型用法**: 上传新源、替换旧源、修正标签

## 七、系统管理工具

### `manage_config(action, key?, value?)`
- `get`: 读取配置（支持点语法嵌套，如 `system.timeout`）
- `set`: 更新配置
- **典型用法**: 查看/修改 `config/env.json`

### `read_logs(lines?)`
- 读取最近 N 行运行日志
- **典型用法**: 排查运行时错误

### `restart_service()`
- 假定使用 pm2 且进程名 `drpys`，执行 `pm2 restart drpys`
- **典型用法**: 修改核心配置后重启

## 八、数据库工具

### `sql_query(query)`
- 执行只读 SQL 查询
- **典型用法**: 诊断数据库数据

## 九、工具调用策略

### 新建源最优调用序列

```
get_spider_template()               → 获取标准骨架
get_claw_ds_skill('zh')             → 获取最新编写规范 (可选)
guess_spider_template(url)          → 判断站型（路线 A/B/C）
analyze_website_structure(url)      → 分析 DOM（路线 A/B）
fetch_spider_url(api_url)           → 测试 API 连通（路线 C）
drpy_write_file(path, content)      → 写入源文件
drpy_check_syntax(path)             → 语法检查
validate_spider(path)               → 结构验证
test_spider_interface(home)         → 首页测试
test_spider_interface(category)     → 一级测试
test_spider_interface(detail)       → 二级测试
test_spider_interface(search)       → 搜索测试
test_spider_interface(play)         → 播放测试
evaluate_spider_source()            → 全流程评估
```

### 修源最优调试序列

```
drpy_read_file(path)                → 读源码
get_resolved_rule(path)             → 看模板继承后字段
test_spider_interface(...)          → 拆开测各接口
debug_spider_rule(url, rule, mode)  → 验证选择器
drpy_edit_file(path, replace_text, ...) → 修源
drpy_check_syntax(path)             → 语法校验
test_spider_interface(...)          → 复测
```

### 上传最优序列

```
house_verify()                      → 验证连接
drpy_check_syntax(path)             → 语法检查
validate_spider(path)               → 结构验证
evaluate_spider_source()            → 全流程评估
house_file(action='upload')         → 上传
```

## 十、工具限制与边界

| 限制 | 说明 |
|------|------|
| 路径范围 | 所有工具仅限操作 drpy-node 项目内的文件 |
| 文档限制 | `drpy_write_file`/`drpy_edit_file` 禁止操作 `.md/.txt` 文档 |
| 数据截断 | engine 测试返回截断至 3000 字符 |
| HTML 截断 | analyze_website_structure 截断至 15000 字符 |
| 无浏览器环境 | 不执行 JS，仅获取静态 HTML |
| noSQL 注入防护 | sql_query 仅执行只读查询 |
| 引擎依赖 | test/evaluate 依赖 localDsCore 正确加载 |
| 仓库依赖 | house 操作依赖 env.json 中正确配置 TOKEN |
