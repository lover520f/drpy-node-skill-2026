---
name: drpy-node-source-create
description: 适用于 drpy-node 新建 DS 源。用户提到"新建源""写个 drpy 源""分析这个站做 DS 源""创建新规则""从零开始做源""写源""做源""建源""生成源""爬虫源"时使用。专注站点分析、模板判断、规则生成与初步验证；已有源修复、播放专项和仓库上传应分流到对应 skill。
---

# drpy-node Source Create

## 快速索引

| 用户意图/站点特征 | 直接入口 | 成功判据 |
|---|---|---|
| 标准 CMS / 模板命中 | 路线 A：继承模板站 | 最小覆盖后 home/category/detail/search/play 可串联 |
| HTML 有内容但接口带签名 | 路线 B2：非模板签名接口站 | 一级/搜索使用真实签名请求，二级优先字典 |
| SPA / body 近空 / JSON API | 路线 C：纯 API 驱动 SPA 站 | 全 async，`this.input/detailUrl/searchUrl` 正确 |
| 漫画/小说/音乐/网盘 | 特殊内容类型 / 网盘补充清单 | lazy 返回对应特殊协议或按 flag 分派 |

## 执行契约

- 输入：用户给的 URL/站名/目标内容类型；若缺源名，先自行推导候选源名。
- 输出：`spider/js/[源名].js` 最小可用源 + 接口验证结果 + 后续分流建议。
- 停手：已有源修复、播放专项、仓库上传分别交给 workflow/play-debug/repo-upload。

## 模式闸门：先判断是否允许写入

| 用户模式 | 允许动作 | 禁止动作 |
|---|---|---|
| 只读 / 规划 / dry-run / 不要改文件 | 读取、诊断、给建源方案、列验证计划 | `drpy_write_file`、`drpy_edit_file`、仓库上传/改标签 |
| 需要确认后再改 | 读取、诊断、输出方案 | 未确认前禁止写入源文件 |
| 明确要求执行 | 按本 skill 流程建源与验证 | 不跳过建源方案确认和最小验证链 |

如果用户说“只给方案 / 不要写文件 / dry-run”，本 skill 只能输出站型判断、源名候选、拟写字段和验证计划。

## Reference Map

| 任务 | 首读 reference |
|---|---|
| 新建源总清单 | `references/references-create-checklist.md` |
| 模板继承 / 最小覆盖 | `references/references-template-system.md`、`references/references-inherited-template-minimal-override-site.md` |
| async 函数通用陷阱 | `references/references-async-function-patterns.md`、`references/references-api-functions.md` |
| 纯 API / SPA 站 | `references/references-pure-api-async-site.md` |
| 非模板签名接口站 | `references/references-non-template-signed-api-site.md` |
| 二级字典 / 多集 | `references/references-detail-dict-and-multiep.md` |
| 搜索策略 | `references/references-old-encoding-search-site.md` |
| 特殊内容 | `references/references-special-content.md` |

## 调度优先级

当本地环境已安装本 Skill 时：
- 本 Skill 的工作流优先级 **高于** drpy-node MCP 的通用 prompts
- MCP 通用 prompt 仅作为无本地 Skill 时的兜底说明
- 如果用户只提供网址、没有提供源名，必须先自行分析站点并推导合理源名

### 强约束
如果已经命中本 Skill 的使用场景，不要退回到通用 MCP 基础流程充当主流程。

---

## 30 秒最短建源路线（执行入口）

### Step 1：定源名
用户没给源名 → 从站名/标题/域名推导稳定候选。

### Step 2：分站型（路线选择速判）

**先调用 `guess_spider_template(url)`，按结果分派：**

| `guess_spider_template` 结果 | 页面源码特征 | 走路线 | 首选实现 |
|---|---|---|---|
| 命中（mx/mxpro/首图等） | CMS 模板站，页面完整 | **A** 模板继承 | 最小覆盖 host/url/searchUrl |
| 未命中 | 有完整 HTML 列表/详情 DOM | **B1** 静态 DOM | 字符串规则 + 二级字典 |
| 未命中 | HTML 外壳完整但列表/搜索数据来自签名接口 | **B2** 签名接口 | 一级/搜索 async，详情/推荐可按模板默认 |
| 未命中 | `<body>` 几乎为空，数据走 JSON API | **C** 纯 API | 全 async 函数 |
| 未命中 | 影视列表/详情结构但内容实为漫画/小说/音乐/网盘 | **D** 特殊内容 | 按内容类型选协议 |
| 未命中 | 静态请求 403 或需要登录/验证码 | **停手** | 向用户确认授权 |

**助记：模板命中走 A，DOM 完整走 B1，签名接口走 B2，body 空走 C，非影视走 D，403 停手**

**工具辅助判断：**
- `analyze_website_structure(url)` → 看列表区域是空容器（SPA）还是直出 HTML（B1/B2）
- `fetch_spider_url(url)` → 看原始响应和 headers，确认是否 403 或需要签名
- 签名接口站需确认：浏览器真实请求是 GET/POST，带 time/sign/token 与否，需要 Ajax headers 与否

### Step 3：确认建源方案（写入前）

在正式写入 `spider/js/[源名].js` 前，先向用户呈现：

```markdown
## 建源方案
- 源名：...
- 站型判断：路线 A / B / C / 特殊内容
- 证据：guess_spider_template / DOM / API / 浏览器抓包摘要
- 拟使用模式：模板继承 / 字符串规则 / async / 网盘型
- 源元数据：`@header` 的 title/类型/lang/searchable/filterable/quickSearch 与实际能力一致
- 最小验证链：home → category → detail → search → play
```

必须得到用户确认后再写入；如果涉及登录态、复杂反爬、批量抓取或上传，也必须先确认。源内 metadata 只描述运行时源身份，不等同于仓库 tags；上传标签交给 repo-upload 处理。

### Step 4：保住最小可用链路

**工具调用顺序：**
```
1. get_spider_template()       → 获取标准模板
2. drpy_write_file()           → 保存到 spider/js/[源名].js
3. drpy_check_syntax(path)     → 语法检查
4. validate_spider(path)       → 结构检查

### 🛑 检查点：确认写入结果再继续验证
源文件已写入并通过语法/结构检查，向用户确认后再跑接口验证链：

```markdown
## 写入确认
- 文件：spider/js/[源名].js
- 语法检查：通过 / 未通过
- 结构检查：通过 / 未通过
- 源 metadata（title/类型/lang/searchable）：与实际预期一致 / 需修正
- 拟验证链：home → category → detail → search → play
```

确认后继续 Step 5-10 验证链；若语法或结构未通过，修好后再继续。

5. test_spider_interface(home) → 测试首页
6. test_spider_interface(category, class_id) → 测试一级
7. test_spider_interface(detail, ids)        → 测试二级
8. test_spider_interface(search, keyword)    → 测试搜索
9. test_spider_interface(play, play_url)     → 测试播放
10. evaluate_spider_source()   → 全流程评估（可选）
```

**单接口调试工具：**
- `debug_spider_rule(rule, mode)` → 测试 CSS/Regex 选择器
- `fetch_spider_url(url)` → 测试 API 连通性和响应
- `extract_website_filter(url)` → 提取分类筛选条件

**筛选 filter 处理要点：**
1. 分类页有地区/年份/排序等筛选时，先调用 `extract_website_filter(url)`；筛选很长时用 `gzip: true` 压缩输出。
2. 生成结果应落到 rule 的 `filter` / `filter_def` / `filter_url`，不要把筛选硬编码进一级 async。
3. 验证筛选时用 `test_spider_interface(category, class_id, ext)`，`ext` 传工具生成的 base64 筛选参数。
4. 如果筛选导致列表为空，先测无筛选 category，再逐个筛选项定位错误。

### 工具参数示例（MCP 真实字段）

```text
test_spider_interface(source_name='源名', interface='category', class_id='1', ext='')
test_spider_interface(source_name='源名', interface='detail', ids='一级返回的真实 vod_id')
test_spider_interface(source_name='源名', interface='search', keyword='高频宽匹配词')
test_spider_interface(source_name='源名', interface='play', play_url='二级返回的真实播放地址', flag='线路名')
```



### 失败兜底与停手边界

| 失败点 | 不要做 | 正确动作 |
|---|---|---|
| `guess_spider_template` 不确定 | 硬套最像的模板 | 用 `analyze_website_structure` + `fetch_spider_url` 判 A/B/C，必要时按 B/C 最小 async |
| 静态请求 403/空 body | 直接判站点不可做 | 用浏览器网络请求复现 headers/method/body；若仍需登录/验证码则停手说明 |
| 签名算法找不到 | 搬整段前端 bundle | 只定位生成 sign/token 的最小函数；找不到就输出阻塞点，不写死抓包值 |
| `fetch_spider_url` 能通但源内不通 | 反复改 selector | 对照 method、headers、body、`this.input`、`searchUrl **` 是否一致 |
| 验证中途失败 | 继续补其它接口 | 停在第一个断点，用真实上游返回值复测后再继续 |
| 涉及登录态/验证码/强风控 | 绕过或批量尝试 | 向用户确认授权范围和可用凭据，未确认不继续 |

---

## 路线 A：继承模板站

当 `guess_spider_template` 命中内置模板时走此路线。

### 核心原则：模板内置优先，最小覆盖
能走模板内置就不要急着手写覆盖。很多问题是继承残留导致的，不是模板本身失效。

### 12 个内置模板选择指南

`guess_spider_template` 返回模板名后，理解其对应关系：

| 模板名 | CMS 类型 | URL 模式 | double | 二级 lazy |
|--------|---------|----------|--------|-----------|
| mx | 苹果CMS旧版 | `/vodshow/fyclass--------fypage---/` | true | common_lazy(提取player_* JSON) |
| mxpro | 苹果CMS Pro | `/vodshow/fyclass--------fypage---.html` | true | common_lazy |
| mxone5 | One5主题 | `/show/fyclass--------fypage---.html` | true | common_lazy |
| 首图 | 首图CMS | `/vodshow/fyclass--------fypage---/` | true | common_lazy |
| 首图2 | 首图CMS v2 | `/list/fyclass-fypage.html` | true | common_lazy |
| vfed | VFed CMS | `/index.php/vod/show/id/fyclass/page/fypage.html` | true | common_lazy |
| 海螺3 | 海螺CMS v3 | `/vod_____show/fyclass--------fypage---.html` | true | common_lazy |
| 海螺2 | 海螺CMS v2 | `/index.php/vod/show/id/fyclass/page/fypage/` | true | common_lazy |
| 短视 | 短视频 | `/channel/fyclass-fypage.html` | true | common_lazy |
| 短视2 | 短视频v2 | API驱动(`#type=fyclass&page=fypage`) | true | common_lazy |
| 采集1 | 采集站 | API: `?ac=detail&pg=fypage&t=fyclass` | false | cj_lazy(依赖parse_url) |
| 默认 | 通用兜底 | 空/自定义 | false | def_lazy(嗅探兜底) |

**double: true** 意味着推荐需要两层解析（先取外层容器，再从内层提取数据）。如果首页推荐空，优先检查是否误用了 `double: true`。

### 什么时候不要盲信模板命中
- `guess_spider_template` 只说明 class_parse 特征相似，不保证 `url/searchUrl/二级/lazy` 都匹配。
- 命中模板但 `get_resolved_rule(path)` 显示 url/searchUrl 与真实站点不同 → 只覆盖 URL 模板，不重写整套规则。
- 命中模板但分类页实际是 API/签名接口 → 保留模板可用字段，一级/搜索局部切 async。
- 命中默认模板 → 视为兜底骨架，不等同于 CMS 模板站。

详细模板字段见 `references/references-template-system.md`。

### 模板继承核查清单（必须完成后再写规则）
1. `class_parse` 是否残留并覆盖 `class_name/class_url`？→ 显式补 `class_parse: ''`
2. `double` 是否导致首页推荐为空？→ 单层推荐优先 `double: false`
3. `url` 是否是真实分类模板？→ 必须验证真实分类页和翻页结构
4. `searchUrl` 是否是真实搜索模板？
5. 首页推荐节点是否真实存在？
6. 删除手写 一级/搜索，优先验证模板内置规则

### 最小覆盖示例（模板继承完整模板）
```js
// spider/js/源名.js
var rule = {
    标题: '站点名',
    模板: 'mxpro',                                    // guess_spider_template 返回
    host: 'https://example.com',
    url: '/vodshow/fyclass--------fypage---.html',   // 验证真实分类 URL
    searchUrl: '/vodsearch/**----------fypage---.html', // 验证真实搜索 URL
    class_parse: '.stui-header__item li;a&&href;/(\\d+)/', // 验证 class_parse
    // 无手写 推荐/一级/搜索/二级 — 全走模板内置
};
```
必须用 `get_resolved_rule(path)` 验证继承后字段，再删除手写规则激活模板链。

### 强禁止
未完成核查清单前，**禁止**一上来就大面积手写 推荐/一级/搜索/二级。

### 常见排障
| 症状 | 优先检查 |
|---|---|
| 首页推荐空 | `double` 是否为 true，推荐节点是否真实存在 |
| detail 不通 | `detailUrl` 是否缺失（纯数字 vod_id 必须设 detailUrl） |
| 搜索空 | 搜索页 DOM 是否独立于一级（不要默认 `搜索: '*'`） |
| `*` 含义不清 | 先读 `references/references-template-system.md`，`*` 继承一级 |

### 参考资料
- `references/references-inherited-template-minimal-override-site.md`

---

## 路线 B：非模板 HTML / 签名接口站

先把非模板站拆成两类，避免把普通静态 DOM 站过度分析成签名接口站：

| 子路线 | 判定 | 首选实现 | 升级条件 |
|---|---|---|---|
| B1 静态 DOM 站 | 分类/详情/搜索 HTML 直出 | 字符串规则 + 二级字典 | 选择器无法稳定覆盖时局部 async |
| B2 签名接口站 | 列表/搜索由 XHR/API 返回，带 time/sign/token/header | async 函数复现浏览器请求 | 签名算法复杂或需登录时停手确认 |

### 路线 B1：静态 DOM/字符串规则站

当 HTML 直接包含列表、详情和搜索结果时，不要先抓签名 API：
1. 用 `analyze_website_structure(url)` 确认列表节点、标题、图片、链接字段。
2. 用 `debug_spider_rule` 先验证一级字符串规则。
3. 详情优先用二级字典 `{title, img, desc, content, tabs, lists}`。
4. 搜索页如果 DOM 独立，单独写 `搜索`，不要默认 `搜索: '*'`。
5. 只有翻页/搜索/章节依赖 XHR 时，才把对应接口局部升级为 async。

**字符串规则源示例：**
```js
var rule = {
    标题: '站点名',
    host: 'https://example.com',
    url: '/list/fyclass-fypage.html',
    searchUrl: '/search/**-fypage.html',
    一级: '.list-box li;a&&title;img&&data-src;.desc&&Text;a&&href',
    二级: {                       // 二级字典 — 详情页 HTML 直出
        title: 'h1&&Text',
        img: '.pic&&img&&src',
        desc: '.meta&&Text',
        content: '.intro&&Text',
        tabs: '.tabs-box>.tab',
        lists: '.tab-content ul:eq(#id) li;a&&title;a&&href'
    },
    搜索: '*',                    // 继承一级规则
};
```

### 路线 B2：非模板签名接口站

当站点不是内置模板，有完整 HTML 外壳，但分类/搜索等数据由前端带签名的接口驱动时走 B2。B2 介于路线 A（纯模板继承）和路线 C（全 async API）之间——页面结构可通过 HTML 解析，但数据加载依赖接口调用。

### 先判断一级是否由签名接口驱动
不要看到 `data-api` 就直接假设这是可裸 GET 的 JSON 接口。必须确认：
1. 浏览器真实请求是 GET 还是 POST
2. 是否带 `time/key/token` 等签名参数
3. 是否需要 Ajax 请求头

### 已验证经验
裸 GET 可能只返回无效文本，浏览器真实请求却是带签名的 POST。

### 分级编写策略
按复杂度从低到高逐步尝试，不要一上来就写全 async：

1. **先试二级字典**：如果详情页是 HTML 而非 JSON，优先用二级字典映射（`{title, img, desc, content, tabs, lists}`）。`lists` 容器层级从 ul 下沉到 li 即可解决多集问题，无需切 async。
2. **一级优先 async**：签名接口必须用 async 函数处理，无法用字符串规则。用 `this.MY_CATE` / `this.MY_PAGE` 代替手动拼 URL。
3. **搜索独立验证**：签名站的搜索接口通常独立于一级，不要假设 `搜索: '*'` 继承生效。先用 `fetch_spider_url` 测试搜索 API 连通性。
4. **模板可混合**：签名接口站的首页推荐和播放页 lazy 可能仍可使用模板默认逻辑。优先保留模板的推荐/lazy，只覆盖一级/搜索。（此处的"模板"指 `get_spider_template()` 生成的代码骨架中的默认实现，不是路线 A 的 CMS 模板继承。）

### 参考资料
- `references/references-non-template-signed-api-site.md`

---

## 路线 C：纯 API 驱动 SPA 站

当页面源码 `<body>` 几乎为空，所有数据来自 `/api/xxx` JSON 响应时走此路线。

### 立即参考
- `references/references-pure-api-async-site.md`

### 核心：全 async 函数模式
不要尝试模板继承，不要写规则字符串，直接用全 async 函数。

### 必读（11 条跨站型规则见下方「通用规则」）
Route C 场景额外注意：
1. `request` POST 用 `body`（`JSON.stringify`）不是 `data`（通用规则 #3）
2. 外部 API 可能需要 `Authorization` → 从浏览器抓包
3. 推荐数据要全量聚合 → 别只取 featured（通用规则 #5）

### 核心要点
1. `this.input` 是渲染后的 URL 字符串 → 通用规则 #1
2. 纯数字 vod_id 必须设 `detailUrl` → 通用规则 #2
3. `searchUrl` 必须带 `**` → 通用规则 #4

### 全 async 源示例（Route C 骨架）
```js
var rule = {
    标题: '站点名',
    host: 'https://example.com',
    url: '/api/fyclass?page=fypage',          // searchUrl 也需带 fyclass/fypage
    detailUrl: '/api/video/fyid',             // 纯数字 vod_id 必须设
    searchUrl: '/api/search?wd=**&page=fypage', // ** 是 this.KEY
    class_parse: '.nav li;a&&Text;/api/(\\d+)',
    推荐: async function() {
        let html = await request(this.input);      // this.input 是 URL 不是响应
        let data = JSON.parse(html).list;
        return data.map(v => ({
            vod_id: v.id, vod_name: v.name, vod_pic: v.pic
        }));
    },
    一级: async function() {
        let html = await request(this.input);
        let data = JSON.parse(html).list;
        return data.map(v => ({
            vod_id: v.id, vod_name: v.name, vod_pic: v.pic
        }));
    },
    二级: async function() {
        let html = await request(this.input);
        let data = JSON.parse(html);
        return { title: data.name, img: data.pic, desc: data.desc, content: data.content };
    },
    搜索: async function() {
        // searchUrl 带 **，this.KEY 自动填充
        let html = await request(this.input);
        let data = JSON.parse(html).list;
        return data.map(v => ({...}));
    },
};
```

---

## 路线 D：特殊内容类型

漫画、小说、音乐、网盘不是普通影视 lazy 的变体，先确认内容形态再建源。合法的特殊内容源应满足：`@header` 的 `类型` 与实际内容一致，detail 产出章节/曲目/文件而不是普通视频集，lazy 返回对应特殊协议或真实音频/网盘输出。

| 类型 | detail 目标 | play/lazy 返回 | 验证重点 |
|---|---|---|---|
| 漫画 | 章节名 + 章节 URL | `pics://` + 图片 URL 列表 | 图片顺序、分页/懒加载、反盗链 headers |
| 小说 | 章节名 + 正文页 URL | `novel://` 或正文内容协议 | 编码、分页、正文清洗 |
| 音乐 | 歌曲名 + 音频页/接口 | mp3/m4a 直链或解析 | content-type、时效签名 |
| 网盘 | 资源标题 + 分享链接 | `push://` 或网盘专用字段 | 提取码、失效提示、多网盘线路 |

特殊内容验证顺序仍是 home → category → detail → play；只是 play 的成功标准换成对应特殊协议，不要强行套 m3u8/mp4 判断。

### lazy 返回示例

当源类型非影视时，lazy 需返回特殊协议而非普通视频链路：

| 类型 | lazy 实现 | 返回格式 |
|------|----------|---------|
| 漫画 | `pdfa(html, '.comic-pages&&img')` → `pdfh(it, 'img&&data-src')` | `{parse:0, url: 'pics://' + urls.join('&&'), js: ''}` |
| 小说 | 请求正文页 → 提取 title/content → JSON 化 | `{parse:0, url: 'novel://' + JSON.stringify({title, content}), js: ''}` |
| 音乐 | 请求播放页 → 提取 m4a/mp3 → urljoin 补全 | 返回音频 URL 字符串（框架自动 parse:0） |
| 网盘 | 按 flag 分派逻辑 → 提取直链 | `{parse:0, url: 'push://...' }` 或对应该网盘协议 |

---

## 通用规则（跨所有站型）

以下规则对模板站/签名接口站/纯 API 站都适用，只要源中有 async function 就必须遵守。

### 必读参考
- `references/references-async-function-patterns.md`（async 函数通用模式与陷阱）
- `references/references-api-functions.md`（沙箱边界、全局函数与 this 上下文）

### 运行时边界

DS 源运行在 drpy 沙箱中，不是普通 Node.js 模块；源内可用 `request/pdfh/pd/pdfa/local/CryptoJS` 等注入能力，但不要假设 `fs/process/原生 require` 或 MCP 工具能在源代码里调用。

### 7 条必背规则
1. **`this.input` 是 URL 不是响应** → 必须 `await request(this.input)`
2. **纯数字 vod_id 必须设 `detailUrl`** → 引擎才能拼出详情 URL
3. **POST 用 `body` 不是 `data`** → `body: JSON.stringify({...})`
4. **`searchUrl` 必须带 `**`** → 否则 `this.KEY` 为空
5. **推荐要完整聚合** → 全量聚合 + 按 vod_id 去重
6. **async 函数用 `this.input` 拿 URL** → 不要手动拼 `HOST + path`
7. **不写重复同名属性** → 删掉空占位，只保留有逻辑的

### 其他通用经验
- 一级 async 优先用 `this.MY_CATE` / `this.MY_PAGE`，不要拆 URL
- `request` / `post` 是全局函数，不在 `this` 上
- detail 测试必须用一级真实返回的 `vod_id`
- 评估搜索时不要机械用默认词"斗罗大陆"，要换高频宽匹配词
- 区分"规则不通"和"评估器没串起来"——单接口通但评估不通是串联问题

### 搜索策略（必须先判断）
处理搜索前，必须先读 `references/references-old-encoding-search-site.md`，判断属于：
1. 原生搜索接口
2. suggest / 联想搜索 fallback
3. RSS fallback

### parser 语法边界
- `||` 优先用于同一 selector 下的属性 fallback（如 `img&&data-original||src`）
- 不要生成 `img&&data-original||img&&src` 这种跨 selector fallback（超出 parser 稳定边界）
- 优先保守、可验证的规则写法

### 签名/API 请求重建清单

遇到路线 B/C 的动态接口时，用这个顺序把浏览器请求还原到源代码：
1. 浏览器网络面板或 Playwright 确认 method、URL、query、body、headers。
2. 区分固定 header、Cookie、Authorization、time/sign/key 等动态字段。
3. 先用 `fetch_spider_url(url, options)` 复现原请求；POST 统一写 `body: JSON.stringify(...)`。
4. 签名参数如果来自页面 JS，优先定位最小算法；不要把整段前端代码搬进源。
5. 复现成功后再写 async；一级用 `this.MY_CATE/MY_PAGE`，搜索用 `this.KEY`。
6. 签名有时效时，在 async 内即时计算，不要写死抓包值。

---

## 二级字典规范

完整参考：`references/references-detail-dict-and-multiep.md`

### 字段槽位语义
| 字段 | 语义 |
|---|---|
| `title` | 片名;类型 |
| `img` | 封面图规则 |
| `desc` | 备注;年份;地区;演员;导演（5段固定槽位） |
| `content` | 简介规则 |
| `tabs` | 线路节点选择器 |
| `tab_text` | 线路名提取规则 |
| `lists` | 当前线路选集列表选择器（支持 #id/#idv） |
| `list_text` | 每集标题（默认 body&&Text） |
| `list_url` | 每集链接（默认 a&&href） |

### 排障要点（详见 reference）
- 页面多集但 detail 只吐 1 集 → 先查 `lists` 容器层级（从 ul 下沉到 li）
- detail 测试必须用一级真实返回的 `vod_id`，禁止主观简化
- 先让 detail 真通，再进入播放链排障
- 先保证 标题/描述/详情/图片/线路/列表 可用（最小化原则）

---

## 与其他 Skill 的分流

### 何时切换到 workflow
- 已变成系统性修源/评估/上传
- 源文件已存在，需要排查而非从零生成
- 任务包含仓库上传、标签修正

### 何时切换到 play-debug
- 主要卡在 lazy/直链/iframe/m3u8/parse 判断

### Handoff Packet

转交时必须带最小上下文，避免下游重新摸索：

| 目标 skill | 必带字段 |
|---|---|
| workflow | 源名、文件路径、站型、home/category/detail/search/play 验证结果、失败接口、证据工具 |
| play-debug | source_name、真实 detail ids、`vod_play_from`、`vod_play_url`、flag、当前 play 返回 |
| repo-upload | 文件路径、L1/L2/L3 验证等级、A/B/C 建议、用户标签要求 |

### 强制停手检查点
出现以上任一情况时，必须停止在 create skill 内扩写，明确切换。

---

## 五种编写模式速览

根据站型和复杂度，从以下五种模式中选一种。优先上层模式（代码量少、更稳定）。

| 模式 | 代码量 | 适用场景 | 关键字段 |
|------|--------|---------|---------|
| **模板继承** | 7-15行 | `guess_spider_template` 命中内置模板 | `模板: 'mxpro'`, `class_parse`, `url` |
| **字符串规则** | 15-30行 | DOM 结构稳定的 B1 静态站 | `一级: 'ul li;a&&title;...'` |
| **js: 内联** | 1-2行表达式 | 字符串规则中嵌入少量计算 | `一级: 'js:let x=input...'` |
| **async 函数** | 50-200行 | B2 签名 API、路线 C、反爬、非标站 | `一级: async function() { ... }` |
| **网盘型** | 100-300行 | 多网盘资源聚合 | `hostJs`, `line_order`, `lazy` 按 flag 分派 |

**核心原则**: 能用模板继承就不用字符串规则，能用字符串就不用 async 函数，逐步增加复杂度。

### 高级字段触发点

这些字段只在确有需求时使用，优先作为局部增强，不要替代简单规则：

| 字段 | 何时使用 | 注意 |
|---|---|---|
| `hostJs` | 站点域名漂移、需从配置页/发布页动态取 host | 返回最终 host；与 `line_order/lazy` 的共享状态要保持最小 |
| `预处理` | 进入接口前需要初始化 cookie/token/全局变量 | 只做初始化，不在里面抓完整列表 |
| `模板修改` | 模板整体适配但少数字段需在继承前调整 | 比复制整份模板更安全；只改目标模板的必要字段 |
| `二级访问前` | 详情页需要追加参数或转换 URL | 返回新 URL，避免在二级主体里重复拼接 |
| `proxy_rule` | 图片/播放/接口需要本地代理转换 | 仅用于必须代理的资源，避免扩大范围 |
| `play_json` | lazy 表面返回与最终 parse/jx/url 不一致 | 先用 `get_resolved_rule` 确认继承结果，再转 play-debug 深排 |
| `play_parse` | 需要显式控制是否启用解析/嗅探 | 不要用它掩盖错误的直链判断 |
| `sniffer` | 播放依赖嗅探页面中的真实媒体请求 | 需要浏览器/网络证据支撑 |
| `isVideo` | 站点 URL 后缀不稳定，需要自定义媒体识别 | 只用于媒体 URL 判断，不替代 lazy 逻辑 |

### 网盘/多线路源补充清单
- `hostJs` 只负责确定可用 host 或公共配置，不要混入分类抓取。
- `line_order` 用于稳定线路排序，避免客户端每次看到不同顺序。
- `lazy` 必须按 `flag` 分派不同网盘/解析逻辑，不要用一个分支处理所有线路。
- 网盘转存、解析、直链提取应分层验证；单条线路失败不要影响其他线路。

## 收尾输出模板

```markdown
## 建源结果
- 文件：spider/js/[源名].js
- 站型：路线 A / B / C / 特殊内容
- 实现方式：模板继承 / 字符串规则 / async / 混合 / 网盘型
- 验证：
  - home：通过 / 未通过（依据）
  - category：通过 / 未通过（class_id=...）
  - detail：通过 / 未通过（ids=...）
  - search：通过 / 未通过（keyword=...）
  - play：通过 / 未通过（flag/url=...）
- 未覆盖风险：登录态 / 签名时效 / 搜索翻页 / 多线路 / 反爬 / metadata 与实际能力不一致
- 下一步：继续修源 / 播放专项 / 转 repo-upload / 结束
```

