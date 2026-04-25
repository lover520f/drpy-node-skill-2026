# drpy DS 源编写知识体系 — 写给 AI 的综合性学习总结

> 基于 drpy-node 框架源码 (`libs/drpyS.js`, `libs/drpysParser.js`, `libs_drpy/template.js`) 
> 和 187 个 JS 源文件分析 + MCP 工具源码分析

## 一、核心思维模型

### 框架本质

drpy 是一个**基于沙箱隔离的爬虫规则引擎**，核心流程：

```
用户请求 → API 路由 → 引擎调度 → 沙箱执行 JS 源
                              ├── 模板继承 (Object.assign)
                              ├── URL 模板渲染 (fyclass/fypage 替换)
                              ├── 分流调度 (字段类型判断)
                              └── 结果格式化 (setResult/vod 结构)
```

关键理解：**你写的 JS 源不是在写一个完整应用，而是在写一个被引擎调用的 "rule 配置"**。引擎负责 HTTP 请求、HTML 解析、结果格式化，你只需要告诉引擎"从哪里取什么数据"。

### 编程模型分层

```
第1层: 字符串 CSS 规则     → 一级: 'ul.vodlist li;a&&title;...'        最简、最稳
第2层: 字符串 + js: 前缀   → 一级: 'js:let body=input...'              少量计算
第3层: async 函数         → 一级: async function() { ... }            最大灵活
第4层: 全 async 函数      → 所有接口都是 async function               复杂 API 站
```

**原则**: 能用第1/2层就不用第3/4层。引擎的 `commonXxxParse()` 函数对字符串规则有深度优化和容错处理。

### 新增源码级认知

- DS 源运行在 `vm.createContext` 沙箱中，不是普通 Node.js 环境；优先使用注入的 `request/pdfh/pd/local/CryptoJS` 等能力。
- 完整生命周期是：读取/解密源码 → 创建沙箱 → 执行源码得到 rule → `模板修改(muban)` → 模板继承 → `initParse()` → `invokeMethod()`。
- `模板修改(muban)` 发生在继承前；继承合并为 `Object.assign(rule, templateRule, originalRule)`，源显式字段最终覆盖模板。
- `this` 是注入 Proxy：先取运行时 `injectVars`，再回退到 rule 字段；`this.input` 永远先理解为渲染后的 URL。
- `validate_spider` 只是 L1 结构验证；运行时证据来自 L2 `test_spider_interface` 和 L3 `evaluate_spider_source`。
- `common_lazy` 的简化模型不能替代源码与真实 play 测试；当前实现还要结合 `playParseAfter()` 与 `play_json` 看最终输出。

## 二、模板继承系统

### 12 个内置模板

| 模板 | 典型 URL 模式 | class_parse | double |
|------|-------------|-------------|--------|
| mx | `/vodshow/fyclass--------fypage---/` | `.top_nav li;...` | true |
| mxpro | `/vodshow/fyclass--------fypage---.html` | `.navbar-items li:gt(0):lt(10);...` | true |
| mxone5 | `/show/fyclass--------fypage---.html` | `.nav-menu-items&&li;...` | true |
| 首图 | `/vodshow/fyclass--------fypage---/` | `.myui-header__menu li:gt(0):lt(7);...` | true |
| 首图2 | `/list/fyclass-fypage.html` | `.stui-header__menu li:gt(0):lt(7);...` | true |
| vfed | `/index.php/vod/show/id/fyclass/page/fypage.html` | `.fed-pops-navbar&&ul.fed-part-rows&&a;...` | true |
| 海螺3 | `/vod_____show/fyclass--------fypage---.html` | `body&&.hl-nav li:gt(0);...` | true |
| 海螺2 | `/index.php/vod/show/id/fyclass/page/fypage/` | `#nav-bar li;...` | true |
| 短视 | `/channel/fyclass-fypage.html` | `.menu_bottom ul li;...` | true |
| 短视2 | `/index.php/api/vod#type=fyclass&page=fypage` | — | true |
| 采集1 | `/api.php/provide/vod/?ac=detail&pg=fypage&t=fyclass` | `json:class;` | false |
| 默认 | 空 | `#side-menu li;...` | false |

### 模板继承机制

```js
// 框架源码行为 (drpyS.js handleTemplateInheritance)
Object.assign(rule, templateRule, originalRule);
// templateRule 在前 → originalRule 在后 → originalRule 覆盖 templateRule
```

这意味着:
1. **源中显式定义的字段永远覆盖模板**
2. **模板的 class_parse 可能覆盖你的 class_name/class_url** — 如果不想用，显式设 `class_parse: ''`
3. **如果不想用模板的 double: true**，显式设 `double: false`

### 最小覆盖原则

最佳实践: 只覆盖与目标站不同的字段。例如樱花动漫:

```js
var rule = {
    title: '樱花动漫',
    模板: 'mxpro',                     // 继承所有 mxpro 规则
    host: 'http://www.yinghuadm.cn',   // 只覆盖 host
    url: '/show_fyclass--------fypage---.html',
    searchUrl: '/search_**----------fypage---.html',
    class_parse: '.navbar-items li:gt(1):lt(6);a&&Text;a&&href;_(.*?)\.html',
    tab_exclude: '排序',
    searchable: 2, quickSearch: 0, filterable: 0,
}
```

## 三、字符串规则语法

### 通用格式

```
列表选择器;标题;图片;描述;链接;详情
```

分号 `;` 分隔各字段；`&&` 连接选择器和属性；`||` 属性 fallback；`:eq(N)` 索引选取。

### 各接口语义

| 接口 | 字段槽位 |
|------|---------|
| 推荐/一级 | `列表;标题;图片;描述;链接;详情(可选)` |
| 搜索 | 同推荐/一级格式，额外支持 POST/JSON |
| 二级 | 对象格式: `{title, img, desc, content, tabs, lists}` |
| `*` | 继承一级规则 |
| `json:` 前缀 | 切换到 JSON 解析模式 |

### desc 五段槽位 (二级字典)

```
desc: '备注;年份;地区;演员;导演'
```

用 `;` 分隔五段固定语义。如果某个字段缺失，用空段位占位。

### 重要边界

- `||` 仅用于**同一选择器下的属性 fallback**: `img&&data-original||src`
- 不要写跨选择器 fallback: `img&&data-original||img&&src`
- 如遇到复杂情况，优先切换到 async function

## 四、async 函数模式

### this 上下文

`this` 是一个 **Proxy 对象**，访问 `this.xxx` 时:
1. 优先返回 `injectVars.xxx`（引擎注入的运行时变量）
2. 回退到 `rule.xxx`（rule 对象的字段）

```js
// 可用 this 变量（从 this 解构）
let { input, MY_URL, HOST, MY_CATE, MY_PAGE, MY_FL, KEY, fetch_params } = this;
// input    = 渲染后的完整 URL（不是响应！）
// MY_CATE  = 分类ID
// MY_PAGE  = 当前页码
// MY_FL    = 筛选条件对象
// KEY      = 搜索关键词
```

### 7 条铁律

1. **`this.input` 是 URL 不是响应** → 必须 `await request(this.input)` 拿响应体
2. **纯数字 vod_id 必须设 `detailUrl`** → 如 `detailUrl: '/api/videos/fyid'`
3. **POST 用 `body` (JSON.stringify)** → 不是 `data` 参数
4. **`searchUrl` 必须带 `**`** → 否则 `this.KEY` 为空
5. **推荐要全量聚合 + 去重** → 不要只取 featured
6. **async 函数优先用 `this.MY_CATE/MY_PAGE`** → 不要手动拼 URL
7. **`request`/`post` 是全局函数** → 不在 this 上

## 五、播放链路详解

### lazy 返回值语义

| 返回格式 | parse | jx | 含义 |
|---------|-------|-----|------|
| `{parse:0, url: 'https://...m3u8'}` | 0 | — | 直链 m3u8/mp4，直接播放 |
| `{parse:0, jx:1, url: 'https://...'}` | 0 | 1 | 站外解析链接（跳转第三方解析器）|
| `{parse:1, url: input}` | 1 | — | 交由嗅探系统处理 |
| `{parse:0, url: 'novel://...'}` | 0 | — | 小说内容 |
| `{parse:0, url: 'pics://...'}` | 0 | — | 漫画图片 |
| `{parse:0, url: 'push://...'}` | 0 | — | 投屏 |
| 字符串直返 | 自动 | 自动 | 交给 `playParseAfter()` 判断直链/解析/嗅探 |

`play_json` 会参与播放后处理，可能覆盖或影响最终 `parse` / `jx` / `url`。当 lazy 代码看似返回直链，但 `test_spider_interface(play)` 输出不同，必须同时检查模板继承后的 `lazy` 与 `play_json`。

### 三种模板默认 lazy

- **common_lazy** (mx/mxpro/首图等): 提取 `player_*` JSON 配置 → 支持 `encrypt:'1'`(unescape) / `encrypt:'2'`(base64Decode+unescape) 解密
- **def_lazy** (默认模板): 始终返回 `{parse:1, url:input}`，完全交由嗅探系统
- **cj_lazy** (采集1): 通过 `rule.parse_url` 调用第三方解析接口

### 播放链路排查顺序

```
detail 真通? → No → 先修 detail
lazy 类型 → common_lazy / def_lazy / cj_lazy
返回类型 → 直链 / 站外解析 / 播放页
加密检查 → encrypt 1/2
修复 → 正确的 parse/jx 返回值
验证 → test_spider_interface(play)
```

## 六、五种编写模式

### 1. 纯模板继承
- **代码量**: 7-15 行自定义字段
- **适用**: 标准 CMS 站，`guess_spider_template` 命中内置模板
- **关键**: 验证 class_parse、url、searchUrl 匹配真实站点

### 2. 字符串规则
- **代码量**: 15-30 行
- **适用**: 页面结构稳定的 DOM 解析站
- **关键**: CSS 选择器需精确；JSON 模式用 `json:` 前缀

### 3. async 函数驱动
- **代码量**: 50-200 行
- **适用**: 非标 CMS、签名 API、有反爬措施的站
- **关键**: 遵守 7 条铁律

### 4. js: 内联代码
- **适用**: 字符串规则中嵌入少量 JS 逻辑
- **关键**: 代码在沙箱中通过 `executeJsCodeInSandbox()` 运行

### 5. 网盘资源型
- **适用**: 百度/夸克/阿里/天翼等多网盘资源聚合
- **关键**: `hostJs` 动态获取域名；`lazy` 根据 flag 分派不同解析逻辑；`line_order` 控制线路排序

## 七、URL 模板变量速查

| 变量 | 说明 | 示例 |
|------|------|------|
| `fyclass` | 分类ID | `/show/fyclass--------fypage---.html` |
| `fypage` | 页码 | 翻页自动替换 |
| `fyid` | 内容ID | `/detail/fyid.html` |
| `fyfilter` | 筛选参数 | `filter_url` 拼接位置 |
| `**` | 搜索关键词 | `/search/**----------fypage---.html` |
| `fl.xxx` | Jinja 筛选 | `filter_url: '{{fl.area}}&{{fl.year}}'` |
| `[url1][url2]` | 翻页特殊语法 | 第1页用url2，第2页起用url1 |

## 八、MCP 工具调用策略

### 快捷速查

| 目标 | 首选工具 | 辅助工具 |
|------|---------|---------|
| 判断站型 | `guess_spider_template` | `analyze_website_structure` |
| 分析 DOM | `analyze_website_structure` | `fetch_spider_url` |
| 模板匹配 | `get_resolved_rule` | `validate_spider` |
| 写源 | `get_spider_template` | `get_claw_ds_skill('zh')` |
| 规则调试 | `debug_spider_rule` | `extract_website_filter` |
| 接口测试 | `test_spider_interface` | `evaluate_spider_source` |
| 播放排障 | `extract_iframe_src` | `test_spider_interface(play)` |
| 标准验证 | `drpy_check_syntax` | `validate_spider` |
| 结构证据 | `validate_spider` | L1：只证明语法/结构合法，不证明运行时可用 |
| 单接口证据 | `test_spider_interface` | L2：真实引擎调用某接口 |
| 全链证据 | `evaluate_spider_source` | L3：首页→一级→二级→播放→搜索串联评分 |
| 规则调试 | `debug_spider_rule(pdfa)` | `pdfa` 传纯 CSS selector，不传完整分号规则 |
| 仓库操作 | `house_verify` → `house_file` | 上传后用 info 核验 tags/is_public/cid |

### 全流程评估链

```
home(首页:20分) → category(一级:20分) → detail(二级:25分)
→ play(播放:25分) → search(搜索:10分) = 100分
```

一级+二级+播放均通过 = 源有效。

## 九、常见陷阱与模式

### 模板继承陷阱
| 陷阱 | 表现 | 修复 |
|------|------|------|
| `class_parse` 残留覆盖 | 分类不对 | 显式设 `class_parse: ''` |
| `double: true` 导致推荐空 | 首页无推荐 | 设 `double: false` |
| 模板 url 不匹配 | 分类翻页异常 | 验证 `url` 模式 |
| hand-write 覆盖模板 | 修错方向 | 先删手写，验证模板内置 |

### async 函数陷阱
| 陷阱 | 表现 | 修复 |
|------|------|------|
| `this.input` 被当响应 | `JSON.parse` 报错 | `await request(this.input)` |
| 缺 `detailUrl` | 二级全空 | 补 `detailUrl` |
| POST 用了 `data` | 服务端不认 | 用 `body: JSON.stringify(...)` |
| `searchUrl` 无 `**` | KEY 为空 | 补 `**` 占位 |
| 推荐不聚合 | 只有几条 | 请求多页 + 按 video_id 去重 |

### 播放链陷阱
| 陷阱 | 表现 | 修复 |
|------|------|------|
| `parse:1` 被当错误 | 误判为不通 | 确认是否为 def_lazy 正常行为 |
| HTTP 地址被当直链 | 非 m3u8/mp4 | 检查 URL 扩展名 |
| encrypt 未解密 | 播放器白屏 | 处理 `encrypt: '1'/'2'` |
| 多线路一刀切 | 部分线路不通 | 按 input 特征分别处理 |

## 十、设计哲学

1. **安全隔离**: 所有 JS 源在 vm.createContext 沙箱中执行，无法访问 Node.js 原生 API
2. **模板优先**: 能走模板继承就不要手写，Object.assign 保证最小覆盖
3. **字符串规则优先**: 能用 CSS 选择器字符串就不要写 async function
4. **渐进增强**: 从简单字符串 → 局部函数覆盖 → 全 async 函数，逐步增加复杂度
5. **缓存优化**: LRU + 多层缓存（moduleCache + ruleObjectCache + pageRequestCache）
6. **DS 源格式**: `@header({...})` 元数据 + `var rule = {...}` 规则定义

### 写源决策树

```
用户给出 URL
  ↓
guess_spider_template(url)
  ↓
├── 命中模板 → 模板继承 + 最小覆盖
│   ├── 验证 class_parse/url/searchUrl
│   ├── 处理 double/tab_exclude 等细节
│   └── 按需覆盖推荐/一级/二级
│
├── 有 HTML 但不命中 → 分析 DOM
│   ├── 结构类似 CMS → 手动指定模板
│   ├── 数据在 JSON → JSON模式 (json:)
│   └── 现代UI/反爬 → async 函数
│
└── 页面空(SPA) → API 驱动
    └── 网络分析 → 全 async 函数
```

### 选型 checklist

| 站型/特征 | 首选路线 | 关键验证 |
|---|---|---|
| 标准 CMS，模板命中 | 模板继承最小覆盖 | `get_resolved_rule` + 分类/搜索/播放单测 |
| HTML 直出列表 | 静态 DOM 字符串规则 | `debug_spider_rule(pdfa/pdfh/pd)` |
| XHR/API 带 sign/token | 签名 API async | 浏览器真实请求 + `fetch_spider_url` 复现 |
| SPA 空壳 | 纯 API async | 找真实 API，设置 `detailUrl/searchUrl` |
| 漫画/小说/音乐 | 特殊协议 | `pics://` / `novel://` / 音频直链 |
| 网盘/多线路 | flag 分派 lazy | `vod_play_from/url` 对齐 + 分线路测试 |
| 需要代理 | `proxy_rule` | 代理 URL 与 content-type 正确 |
| CatVod/JS0 示例 | 只作对照 | 不直接混入 DS 的 `var rule` 契约 |

## 十一、进阶知识

### hostJs 动态域名

```js
hostJs: async function () {
    let html = await request('https://config.example.com/domains');
    let json = JSON.parse(html);
    return json.domains[0];  // 返回可用域名
}
```

### 模板修改

```js
模板修改: async function(muban) {
    muban.mxpro.一级 = '自定义选择器'; // 在继承前动态修改模板
}
```

### 二级访问前预处理

```js
二级访问前: async function () {
    return this.MY_URL + '?full=1'; // 返回新URL覆盖默认请求
}
```

### proxy_rule 代理规则

```js
proxy_rule: async function (params) {
    return [200, 'text/plain', 'proxy content']; // 自定义代理响应
}
```

### filter 筛选系统

```js
filter_url: 'filter/list?catid=fyclass&rank={{fl.排序}}&cat={{fl.类型}}'
filter_def: { 电视剧: {排序: 'time', 类型: 'all'} }
// filter 可 gzip+base64 压缩，框架自动解压
```

### 特殊内容协议

- **漫画**: `lazy` 返回 `pics://url1&&url2&&url3`
- **小说**: `lazy` 返回 `novel://` + JSON.stringify({title, content})
- **音乐**: `lazy` 直接返回 m4a/mp3 直链
