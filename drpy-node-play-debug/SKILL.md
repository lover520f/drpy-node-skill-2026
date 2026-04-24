---
name: drpy-node-play-debug
description: 适用于 drpy-node 源的播放链路排查与 lazy 修复。用户提到"播放不通""lazy 不对""是不是直链""play.html 被当直链""iframe 提取""m3u8 提取""parse:0/1 判断""假播放""站外解析""加密链接""player_* 配置""乱码链接""webplay"时使用。专注于判断播放结果是否真实可播，通过 30 秒速查决策树快速定位问题类型，优先修复 lazy 逻辑。包含 common_lazy/def_lazy/cj_lazy 三种模板默认行为、encrypt 解密、iframe/m3u8 提取和特殊内容协议(pics/novel/push)的完整覆盖。
---

# drpy-node Play Debug

## 快速索引

| 播放表现 | 优先判断 | 首选动作 |
|---|---|---|
| detail 没有 `vod_play_url` | 不是播放问题 | 交回 workflow 修二级 |
| `parse:0` 返回 `/play/*.html` | 假直链 | 改 parse:1 或提取 iframe/m3u8 |
| `player_*` url 乱码 | encrypt/base64/sign | 先解密再判直链/解析 |
| 浏览器可播但静态 HTML 无 m3u8 | JS 运行时生成 | Playwright network 兜底 |
| 多线路部分可播 | 按 flag/input 分派 | 不要一刀切 lazy |

## 执行契约

- 输入：source_name、detail ids、play_url、flag、当前 lazy 返回值或错误表现。
- 输出：播放类型判断、lazy 修复方案、play 接口验证结果。
- 停手：detail 不稳定、一级/搜索同时异常、上传/评估整源诉求出现时交回 workflow。

## 调度优先级

当本地环境已安装本 Skill 时：
- 本 Skill 优先级 **高于** drpy-node MCP 的通用调试 prompt
- 播放链排障必须优先按本 Skill 的判断顺序执行

---

## 30 秒速查决策树

当用户说"播放不通 / lazy 不对 / play.html 被当直链"时：

### Step 1：检查前提
- 一级/二级/搜索也同时异常？→ 交回 `drpy-node-source-workflow`
- detail 还没稳定产出 `vod_play_from/vod_play_url`？→ 先修二级，不要判 lazy

**工具调用（MCP 真实字段示例）：**
```text
test_spider_interface(source_name='源名', interface='detail', ids='一级返回的真实 vod_id')
test_spider_interface(source_name='源名', interface='play', play_url='二级返回的真实播放地址', flag='线路名')
```


### Step 2：判断当前 lazy 更接近哪类模板默认
| 类型 | 特征 | 参考 |
|---|---|---|
| **common_lazy** | 播放页有 `player_*` JSON，可能有 encrypt | references-play-lazy-summary.md §1 |
| **def_lazy** | 站点播放页交给解析系统，`parse:1` | references-play-lazy-summary.md §2 |
| **cj_lazy** | 采集站/解析接口，有 `parse_url` | references-play-lazy-summary.md §3 |

### Step 3：判断返回类型
| 类型 | 特征 | 正确返回 | 验真要点 |
|---|---|---|---|
| 直链媒体 | m3u8/mp4/m4a/mp3 | `{parse:0, jx:0, url}` | URL 应可请求且 content-type/内容像媒体或 m3u8 |
| 站外解析链接 | 外部解析器地址 | `{parse:0, jx:1, url}` | 不能把普通网页误当直链 |
| 网站播放页 | 当前站播放页 URL | `{parse:1, url:input}` | parse:1 是嗅探意图，不是失败本身 |

### URL 类型快速判定表
| URL 特征 | 类型 | 正确返回格式 |
|----------|------|-------------|
| 以 `.m3u8` / `.mp4` / `.mp3` / `.m4a` 结尾 | 直链媒体 | `{parse:0, jx:0, url}` 或直接返回 URL 字符串 |
| 跨域域名，含 `jx` / `解析` / `play` / `player` / `url=` | 站外解析链接 | `{parse:0, jx:1, url}` |
| 本域名且路径含 `/play/` / `/vodplay/` | 网站播放页 | `{parse:1, url:input}` |

### 🛑 检查点：确认诊断后再修复
在编写或修改 lazy 前，先确认：
- detail 是否稳定产出 `vod_play_from/vod_play_url`
- 当前返回类型是直链/站外解析/播放页中的哪一种
- 当前是否属于假通过
- 预期应该返回什么类型

确认模板：
```markdown
## 播放诊断确认
- detail 状态：稳定 / 不稳定
- 当前 play 返回：...
- 类型判断：直链 / 站外解析 / 网站播放页 / 特殊协议
- 假通过风险：有 / 无
- 拟修复 lazy：...
- 验证计划：test_spider_interface(play)
```

确认后再进入修复，避免方向错误。

### lazy 编辑验证清单

修改 lazy 前后必须保留证据：
1. 修改前：记录 `detail` 输出里的 `vod_play_from/vod_play_url`。
2. 修改前：记录 `play` 当前返回值和假通过判定。
3. 修改时：只改 lazy 相关逻辑，不顺带重写一级/二级。
4. 修改后：重新测同一个 `play_url + flag`。
5. 多线路：至少测一条直链线路和一条解析/网页线路。



### 强提醒
- **`parse:1 + 播放页链接` 不一定是错**，可能是 def_lazy 的合理输出
- **拿到 http 链接 ≠ 拿到直链**，先判断是不是 m3u8/mp4
- **遇到 player_* 配置优先查 encrypt**，很多站不是没链接而是没解密

---

## 框架 playParseAfter 后处理机制

lazy 返回后，框架调用 `playParseAfter()` 做最终判断：

```js
// drpysParser.js:683-734 核心逻辑
parse = SPECIAL_URL.test(playUrl) || /^(push:)/.test(playUrl) || 
        /\.(m3u8|mp4|m4a|mp3)/.test(playUrl) ? 0 : 1;
jx = tellIsJx(playUrl);  // 判断是否站外解析
```

这意味着：
- **只要 lazy 返回的 URL 以 m3u8/mp4/m4a/mp3 结尾** → 自动设 `parse:0`
- **即使 lazy 返回 `{parse:1, url}`，但 url 是直链** → 框架会覆盖为 parse:0
- **`tellIsJx(url)` 判断站外解析**（源码锚点：`drpysParser.js` 的 `playParseAfter()` 调用链）→ 自动设 `jx:1`

所以在 lazy 中返回字符串 `"https://example.com/movie.m3u8"` 也是安全的——框架会自动处理。

## tellIsJx 判断规则（站外解析识别）

框架的 `tellIsJx(url)` 通过 URL 特征判断是否为站外解析链接：

```
匹配站外解析的典型特征：
- 域名与当前站点 host 不同（跨域）
- URL 含常见解析关键字: jx, 解析, api, player, parse, open, url
- URL 指向已知解析服务商域名模式
```

手动判断时可参考此规则，但最终以框架 `tellIsJx` 正式行为为准。

## 特殊情况：webplay 线路

部分站点同时提供多条播放线路，其中某些线路是「webplay」需要特殊处理：

```
线路A: /play/1-1.html  (普通播放页)
线路B: /webplay/1-1.html  (webplay 模式)
```

对于 webplay 线路，通常需要在 lazy 中检测 URL 特征并做不同处理：
1. 普通线路 → 模板默认 lazy（common_lazy 或 def_lazy）
2. webplay 线路 → 可能需要 fetch 页面内容并提取 iframe/m3u8

## 模板默认 lazy 逻辑

完整参考：`references/references-play-lazy-summary.md`

### common_lazy 关键代码
```js
// 播放页 → 解析 player_* JSON → 判断直链/站外/回退
if (/\.(m3u8|mp4|m4a|mp3)/.test(url)) {
    return { parse: 0, jx: 0, url };           // 直链
} else if (tellIsJx(url)) {
    return { parse: 0, jx: 1, url };            // 站外解析
} else {
    return input;                                // 回退
}
```

### 加密处理
| encrypt | 处理 |
|---|---|
| `'1'` | `unescape(url)` |
| `'2'` | `base64Decode(url)` → `unescape()` |

### def_lazy
```js
return { parse: 1, url: input, js: '' }
```

### cj_lazy
检查 `rule.parse_url` 配置 + 是否走 `json:` 解析接口。

---

## 假通过识别

`test_spider_interface(play)` 返回 success 不等于真实可播，必须继续看返回内容：

| 假通过表现 | 判定 | 下一步 |
|---|---|---|
| `parse:0` 但 url 是 `/play/...html` 或普通 HTML 页 | 播放页被误当直链 | 改为 `parse:1` 或提取真实 m3u8/iframe |
| `parse:0` 但 url 不是 http/特殊协议/媒体后缀 | 加密或拼接错误 | 查 encrypt/base64/sign |
| `jx:1` 但 url 是本站详情/播放页 | 站外解析误判 | 保留本站播放页走 parse:1 |
| 返回空 url 或原 input | lazy 没提取到有效目标 | 回到 player_* / iframe / 网络请求排查 |

以下情况必须二次确认后再改：
- lazy 需要重写超过一个分支或影响多条线路。
- 需要从 common_lazy 改为完全自定义 lazy。
- 需要引入 Playwright 抓到的签名 API 或动态 token。
- 需要删除模板继承的 lazy 或 parse_url。

---

### 真实可播验证清单

拿到 `play` 返回值后，按返回类型做验真，不要只看接口 success：

| 返回类型 | 必验项目 | 失败时动作 |
|---|---|---|
| m3u8 | 请求状态 200/206；响应或首段内容含 `#EXTM3U`；跳转后仍是 m3u8 | 补 headers/referer；若是播放页伪装则改 parse:1 |
| mp4/mp3/m4a | 状态 200/206；content-type 或 URL 后缀像媒体；非 HTML 文本 | 若返回 HTML/403，查防盗链或签名时效 |
| 站外解析 | `jx:1` 且 URL 不是本站详情/播放页；包含目标媒体参数 | 保留为解析，不要误设直链 parse:0 |
| 网站播放页 | `parse:1`；页面能让嗅探器或浏览器产生媒体请求 | 静态抓不到时走 Playwright 网络兜底 |
| 特殊协议 | `pics://` / `novel://` / `push://` 格式完整 | 转对应内容类型检查，不套视频直链规则 |

如果媒体 URL 依赖 Referer/Cookie/User-Agent，要把 headers 放进返回结构或 lazy 请求链路里验证；不要只复制裸 URL。

---

## iframe / m3u8 提取

### iframe 提取
用 `extract_iframe_src(url)` 工具从播放页中提取 iframe 的 src。

### m3u8 嗅探
用 `fetch_spider_url(url)` 请求播放页 URL，在响应中搜索 `.m3u8` 链接。

### Playwright 网络兜底
如果 `fetch_spider_url` / `extract_iframe_src` 都找不到 m3u8，但浏览器里能播放，说明媒体 URL 可能由 JS 运行时/XHR 生成：
1. 用 `browser_navigate(play_url)` 打开播放页。
2. 用 `browser_network_requests(filter='m3u8|mp4|api|play|url')` 查看运行时请求。
3. 若请求需要点击播放按钮，先 `browser_snapshot` 找按钮，再点击后重新看网络。
4. 从网络请求中提取真实媒体 URL 或签名 API，再回写 lazy。

不要用静态 HTML 抓取失败直接判定站点无直链。

### player_* 配置提取
部分站的播放器配置（含视频地址）不在 HTML 元素属性中，而是嵌入在 `<script>` 标签的 JSON 对象里：
1. 用 `pdfa(html, 'script')` 列出所有 script 标签内容
2. 搜索含 `player_` 前缀的变量（如 `player_0`、`player_data`、`player_js`）
3. 找到后用 `JSON.parse` 或正则提取 url 字段
（若 player_* 配置在 HTML 属性中，仍优先用模板默认 common_lazy 处理）

### 多线路混合处理
当一个源有多条线路（如"线路A直链m3u8 + 线路B需要解析"），lazy 不应一刀切。
应根据 input 的 URL 特征分别处理：
```js
lazy: async function () {
    let {input} = this;
    if (/\.m3u8/.test(input)) return {parse: 0, url: input};     // 直链线路
    if (/\.mp4/.test(input)) return {parse: 0, url: input};       // 直链线路
    return {parse: 1, url: input};                                  // 需解析线路
}
```

### 加密链接处理
部分站的播放链接是加密的（如 base64 编码的 m3u8 地址），需要在 lazy 中先解密：
```js
lazy: async function () {
    let {input} = this;
    // 如果链接是加密的，先解密
    if (!input.startsWith('http')) {
        try { input = base64Decode(input); } catch(e) {}
    }
    if (/\.m3u8/.test(input)) return {parse: 0, url: input};
    return {parse: 1, url: input};
}
```

### 加密排查子流程（从乱码到解密）
当播放页有 player_* JSON 配置且 url 看起来像乱码时，按下述顺序排查：

```
1. 找到 player_* JSON → 查找 encrypt 字段的值
2. 如果 encrypt: '1'   → 对 url 执行 unescape() 解码
3. 如果 encrypt: '2'   → 对 url 执行 base64Decode() → unescape() 解码
4. 如果无 encrypt 字段 → url 可能是 base64 编码字符串，尝试 base64Decode()
5. 如果解码后仍是乱码 → 检查是否包含自定义签名(time/sign/key)
6. 验证：解码后的 URL 是否以 .m3u8/.mp4 结尾，或为有效 http 链接
```

### 常见加密模式（从真实源中总结）
| 加密方式 | 特征 | 处理 |
|---------|------|------|
| `encrypt: '1'` | player_* JSON 中 | `unescape(url)` |
| `encrypt: '2'` | player_* JSON 中 | `base64Decode(url)` → `unescape()` |
| URL 非 http 开头 | 如 base64 字符串 | `base64Decode(input)` |
| 自定义签名 | URL 含 `time/sign/key` | 需逆向签名算法 |
| 动态解密 WASM | 页面加载 wasm 解密 | 需调用 wasm 或预解密映射 |

## 特殊协议处理

播放链不仅处理视频，还处理小说、漫画、音乐等内容类型：

### comic (漫画) — pics:// 协议
```js
lazy: async function () {
    let html = await request(input);
    let urls = pdfa(html, '.comic-pages&&img')
        .map(it => pdfh(it, 'img&&data-src'));
    return { parse: 0, url: 'pics://' + urls.join('&&'), js: '' };
}
```

### novel (小说) — novel:// 协议
```js
lazy: async function () {
    let html = await request(content_url);
    let json = JSON.parse(html);
    let ret = JSON.stringify({ title, content: json.data.content });
    return { parse: 0, url: 'novel://' + ret, js: '' };
}
```

### audio (音乐) — 直链 m4a/mp3
```js
lazy: async function () {
    let html = await request(input);
    let music = html.match(/var\s+music\s*=\s*(\{[\s\S]*?\})/)[1];
    music = JSON5.parse(music);
    return urljoin(input, music.file + ".m4a");  // 返回字符串
}
```

### push (投屏) — push:// 协议
```js
return { parse: 0, url: 'push://' + playUrl, js: '' };
```

---

## 与其他 Skill 的边界

### 本 Skill 负责
- 判断播放结果是否真实可播
- 修复 lazy 逻辑
- 判断 parse:0/1、jx:0/1
- 提取 iframe / m3u8 / 站外解析链接

### 本 Skill 不负责
- 从零写完整源
- 系统性重建首页/一级/二级规则
- 仓库上传决策

### 何时交回 workflow
- 一级/二级/搜索也同时异常
- 问题已超出播放链，涉及整体重建
- 用户目标是评估/上传/回滚

### 强制停手检查点
出现以下任一情况时，暂停深挖 lazy，交回 workflow：
- detail 还没稳定产出 vod_play_from/vod_play_url
- 问题已明显超出播放链
- 用户目标已变成"评估整份源是否可用"

---

## 最小化原则

本 Skill 默认目标是让播放链路可用，不是把整份源补全。

### 不要在播放排障时顺带做的
- 补年份/地区/演员/导演
- 美化搜索字段
- 整体源评估

这些属于源整体完善项，应交回 create/workflow。

---

## 实战排查案例：从 play.html 到正确 lazy

### 场景
某源 play 测试返回 `{"url": "/play/1-1.html", "parse": 0}`，播放器不工作。

### 排查步骤

```
1. 前提检查：test_spider_interface(detail) → detail 通，vod_play_url 有值 ✓
2. lazy 类型：源未写自定义 lazy → 走模板默认（mxpro → common_lazy）
3. 返回类型：url 是 /play/1-1.html → 网站播放页，不是直链
4. 根因诊断：common_lazy 预期提取 player_* JSON 输出直链，
   但实际返回了 /play/1-1.html → 说明播放页可能没有 player_* 配置，
   或 common_lazy 解析失败
5. 修复方案 A（简单）：去掉自定义 lazy，让框架用 def_lazy 兜底
   → 返回 {parse:1, url:input}，交由嗅探系统
6. 修复方案 B（直链）：用 extract_iframe_src 提取 iframe → m3u8
   → 返回 {parse:0, url: 'https://...m3u8'}
7. 验证：test_spider_interface(play) 确认修复后返回值
```

### 关键教训
- **不要见 play.html 就判 lazy 写错** — 先区分是预期行为还是解析失败
- **修复应优先选简单方案** — 嗅探可用就不必强行提取直链
- **始终从 detail 状态开始排查** — detail 不通时排查 lazy 毫无意义

## 收尾输出模板

```markdown
## 播放修复结果
- 源：...
- detail 状态：稳定 / 不稳定
- play_url / flag：...
- 根因：假直链 / encrypt / iframe / 站外解析 / 多线路 / 其他
- lazy 改动：...
- 验证结果：
  - 修改前 play：...
  - 修改后 play：...
  - 是否真实可播：是 / 否 / 需要浏览器确认
- 下一步：交回 workflow / 继续播放专项 / 结束
```

## 排查顺序总结

```
1. detail 真通？ → No → 交回 workflow
2. lazy 类型判断 → common_lazy / def_lazy / cj_lazy
3. 返回类型判断 → 直链 / 站外 / 播放页
4. 加密检查 → encrypt 1/2
5. 修复 lazy → 用正确 parse/jx 返回
6. 验证 → test_spider_interface(play)
```
