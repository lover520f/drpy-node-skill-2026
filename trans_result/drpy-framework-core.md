# drpy 框架核心机制总结

## 一、整体架构

```
请求 → API路由(controllers/api.js) → drpyS.js引擎 → 沙箱执行
                                               ├── init() 初始化模块
                                               │   ├── 读取/解密 JS 源码并计算 hash
                                               │   ├── getSandbox() 创建隔离沙箱
                                               │   ├── 执行 JS 源码获取 rule 对象
                                               │   ├── 模板修改(模板修改) → handleTemplateInheritance() 模板继承
                                               │   ├── initParse() 解析 host/url/headers/cookie/预处理
                                               │   └── 缓存 moduleObject / ruleObject
                                               └── invokeMethod() 调度分发
                                                   ├── home/class_parse → 首页分类
                                                   ├── homeVod/推荐 → 推荐内容
                                                   ├── category/一级 → 分类列表
                                                   ├── detail/二级 → 详情
                                                   ├── search/搜索 → 搜索
                                                   ├── play/lazy → 播放解析
                                                   └── proxy → 代理
```

## 二、沙箱隔离机制

### 2.1 创建过程 (`drpyS.js:273-334`)

```js
export async function getSandbox(env = {}) {
    // 1. 加载 WASM (CryptoJSW、simplecc)
    await CryptoJSW.loadAllWasm();
    
    // 2. 合并静态沙箱对象（预合并提升性能）
    const sandbox = {
        console, WebAssembly, setTimeout, ...  // 基础能力
        ...GLOBAL_STATIC_SANDBOX,              // pdfh/pd/pda/req/local等
        env动态变量,                             // getProxyUrl, requestHost等
    };
    
    // 3. 创建VM上下文
    const context = vm.createContext(sandbox);
    
    // 4. 注入初始化代码（ES6扩展 + 模板检查函数）
    SANDBOX_INIT_SCRIPT.runInContext(context);
    
    return { sandbox, context };
}
```

### 2.2 沙箱中可用的全局函数

| 函数 | 来源 | 用途 |
|---|---|---|
| `pdfh(html, rule)` | `jsoup` | 解析HTML提取文本 |
| `pd(html, rule, refUrl)` | `jsoup` | 解析HTML提取URL(属性值) |
| `pdfa(html, rule)` | `jsoup` | 解析HTML获取元素列表 |
| `pdfl(html, rule, ...)` | `jsoup` | 解析选集列表 |
| `pjfh(json, rule)` | `jsoup` | JSON模式pdfh |
| `pjfa(json, rule)` | `jsoup` | JSON模式pdfa |
| `pj(json, rule)` | `jsoup` | JSON模式pd |
| `request(url, options)` | `reqs` | HTTP GET请求 |
| `post(url, options)` | `reqs` | HTTP POST请求 |
| `req(url, options)` | `reqs` | 增强请求(返回{content, headers}) |
| `log(msg)` | `console` | 日志输出 |
| `setResult(d)` | `drpyCustom` | 格式化列表返回 |
| `base64Encode/Decode` | `crypto-util` | Base64编解码 |
| `md5(str)` | `crypto-util` | MD5哈希 |
| `local` | `drpyCustom` | 本地存储 |
| `ENV` | `env` | 环境变量 |
| `CryptoJS` | `crypto-js` | 完整加密库 |

### 2.3 this 上下文注入 (`drpysParser.js:1482-1546`)

关键机制：`invokeWithInjectVars()` 使用 **Proxy** 对象作为 `this`，当源内代码访问 `this.input` 时：

```js
let thisProxy = new Proxy(injectVars, {
    get(injectVars, key) {
        return injectVars[key] || rule[key]  // 优先取injectVars，其次rule
    },
    set(injectVars, key, value) {
        rule[key] = value;      // 同时写入rule和injectVars
        injectVars[key] = value;
    }
});
```

这意味着：`this.input` = 注入的 URL 字符串；`this.host` = rule.host。

补充细节：DS 源运行在 `vm.createContext` 创建的注入沙箱里，不是普通 Node.js 模块环境。源内应使用沙箱提供的 `request/pdfh/pd/local/CryptoJS` 等能力，不能假设 `fs/process` 或任意 npm 包天然可用。

`invokeWithInjectVars()` 对 `推荐` 的异常处理更宽松：推荐函数报错时可能被降级为空列表，而一级、二级、搜索、播放等接口错误通常更容易暴露为接口失败。因此“首页推荐空”不一定等同于整个源不可用，需要继续拆接口验证。

## 三、模块初始化流程 (`drpyS.js:344-472`)

### Step 1: 读取文件 + Hash计算
```js
const fileContent = await readFile(filePath, 'utf-8');
const fileHash = computeHash(fileContent);
```

### Step 2: 获取沙箱 + 执行源码
```js
const {sandbox, context} = await getSandbox(env);
// 将JS源码包装为async函数，执行后获取rule对象
const js_code = await getOriginalJs(fileContent);  // DS格式自动解密
// 包装执行: _asyncGetRule = (async function() { [源码] return rule; })();
// 超时控制: 30秒
```

### Step 3: 模板修改 + 模板继承 (`handleTemplateInheritance`)
```js
// 自动模板匹配: 请求host页面，用每个模板的class_parse尝试解析
if (rule['模板'] === '自动') {
    // 遍历模板，匹配成功则继承
}

// 如果定义了模板修改，会在继承前拿到完整模板 map
if (typeof rule.模板修改 === 'function') {
    await rule.模板修改(muban);
}

// 普通模板继承: Object.assign(rule, templateRule, originalRule)
if (rule.模板 && muban.hasOwnProperty(rule.模板)) {
    const templateRule = muban[rule.模板];
    Object.assign(rule, templateRule, originalRule);
    // 注意: 模板属性在前，rule属性在后（rule覆盖模板）
}
```

要点：`模板修改(muban)` 的作用点是“继承前修改模板定义”；继承完成后 `模板` / `模板修改` 会被移除，源文件里显式写出的字段仍然最终覆盖模板字段。

### Step 4: 初始化解析 (`initParse`)
```js
rule.host = rule.host.rstrip('/');
// 执行hostJs动态获取host
// 拼接url、detailUrl、searchUrl（fyclass/fypage/fyid/fyfilter替换）
// 处理headers（UA常量替换、Cookie远程获取）
// 执行预处理函数
```

## 四、调度分发机制 (`drpyS.js:607-732`)

### 4.1 方法路由

`invokeMethod()` 根据 method 类型进行以下分支判断：

```js
switch (method) {
    case 'get_rule':    return moduleObject;
    case 'class_parse': injectVars = homeParse(...);
    case '推荐':        injectVars = homeVodParse(...);
    case '一级':        injectVars = cateParse(...);
    case '二级':        injectVars = detailParse(...);
    case '搜索':        injectVars = searchParse(...);
    case 'lazy':        injectVars = playParse(...);
}
```

### 4.2 字段类型分发（针对每个 method）

| rule字段类型 | 处理函数 | 适用场景 |
|---|---|---|
| `async function` | `invokeWithInjectVars()` | 复杂逻辑、API调用 |
| `string (CSS规则)` | `commonXxxParse()` | 简单DOM解析 |
| `js:开头字符串` | `executeJsCodeInSandbox()` | 需要沙箱内eval |
| `*` (星号) | 继承一级规则 | 推荐/搜索与一级相同 |
| `二级: {对象}` | `commonDetailListParse()` | 字典式二级配置 |
| `二级: '*'` | 跳过二级，一级链直接播放 | 极简源 |
| 未定义 | 默认返回值 | 保护性降级 |

### 4.3 URL 模板变量

| 变量 | 说明 | 示例 |
|---|---|---|
| `fyclass` | 分类ID | `url: '/show/fyclass--------fypage---.html'` |
| `fypage` | 页码 | 翻页自动替换 |
| `fyid` | 内容ID | `detailUrl: '/detail/fyid.html'` |
| `fyfilter` | 筛选URL | `filter_url`拼接位置 |
| `**` | 搜索关键词 | `searchUrl: '/search/**----------fypage---.html'` |
| `fyid` | 内容ID | `detailUrl: '/api/videos/fyid'` |
| `fl.xxx` | Jinja模板筛选 | `filter_url: '{{fl.area}}&{{fl.year}}'` |

### 4.4 翻页特殊语法

```js
// url中含 [url1][url2] 时: 第1页用url2，第2页起用url1
url: '/list/fyclass-fypage.html[/list/fyclass.html]'
// fypage的运算语法: 支持 (fypage-1)*20 等
url: '/api?offset=((fypage-1)*20)'
```

## 五、播放链路详解

### 5.1 lazy 返回值语义

| 返回值 | parse | jx | 含义 |
|---|---|---|---|
| `{parse:0, url: m3u8/mp4}` | 0 | — | 直链，直接播放 |
| `{parse:0, jx:1, url: ...}` | 0 | 1 | 站外解析链接 |
| `{parse:1, url: input}` | 1 | — | 交给嗅探系统 |
| `{parse:0, url: 'novel://...'}` | 0 | — | 小说内容 |
| `{parse:0, url: 'pics://...'}` | 0 | — | 漫画图片 |
| `{parse:0, url: 'push://...'}` | 0 | — | 投屏 |

### 5.2 playParseAfter 后处理 (`drpysParser.js:683-734`)

```js
// 智能判断是否是直链
parse = SPECIAL_URL.test(playUrl) || /^(push:)/.test(playUrl) || 
        /\.(m3u8|mp4|m4a|mp3)/.test(playUrl) ? 0 : 1;
jx = tellIsJx(playUrl);  // 判断是否站外解析
```

补充：如果 rule 定义了 `play_json`，`playParseAfter()` 还会用它影响最终 `parse` / `jx` / `url` 结果。因此排查“lazy 返回值和最终播放结果不一致”时，除了看 lazy 本身，也要检查模板继承后的 `play_json`。

### 5.3 模板默认 lazy

| 模板 | lazy行为 |
|---|---|
| mx/mxpro/首图等 | `common_lazy`: 提取 `player_*` JSON → 判断encrypt → 返回直链/站外 |
| 默认 | `def_lazy`: `{parse:1, url:input}` 完全交由嗅探 |
| 采集1 | `cj_lazy`: 通过 `rule.parse_url` 调用解析接口 |

## 六、字符串规则解析语法

### 6.1 CSS选择器规则格式

```
规则 = 列表选择器;标题;图片;描述;链接;详情
```

- `;` 分号分隔各字段
- `&&` 连接选择器与属性：`标签&&属性` 或 `标签&&Text`(取文本)
- `||` 属性fallback：`img&&data-original||src`
- `:eq(N)` 按索引选取
- `:gt(N):lt(N)` 范围选取
- `+` 连接多个URL来源：`a&&href+span&&data-url`
- `json:` 前缀切换到JSON解析模式

### 6.2 示例

```js
// 一级列表: 从ul.vodlist的li中提取
一级: 'ul.vodlist li;a&&title;a&&data-original;.pic_text&&Text;a&&href'

// 二级字典
二级: {
    title: 'h1&&Text;.tag-link:eq(-1)&&Text',
    img: '.lazyload&&data-original||src',
    desc: '.desc&&Text;.year&&Text;.area&&Text;.actor&&Text;.director&&Text',
    content: '.intro&&Text',
    tabs: '.module-tab-item',           // 线路标签
    lists: '.module-play-list:eq(#id) a',  // 选集列表
    tab_text: 'div--small&&Text',        // 线路名提取
}
```

## 七、filter 筛选机制

### 7.1 filter_url 格式

```js
filter_url: 'filter/list?catid=fyclass&rank={{fl.排序}}&cat={{fl.类型}}'
filter_def: {
    电视剧: {排序: 'time', 类型: 'all'},
    // 各分类默认筛选值
}
```

### 7.2 filter 压缩编码

filter 长字符串可经过 gzip + base64 压缩，框架在 `homeParse()` 中自动解压：

```js
if (typeof(rule.filter) === 'string' && rule.filter.trim().length > 0) {
    let filter_json = ungzip(rule.filter.trim());
    rule.filter = JSON.parse(filter_json);
}
```

## 八、验证证据层级

| 层级 | 工具/方式 | 能证明什么 | 不能证明什么 |
|---|---|---|---|
| L1 | `drpy_check_syntax` / `validate_spider` | JS 语法和 rule 结构基本合法 | 接口真实可用、播放可播 |
| L2 | `test_spider_interface` | 单个接口经真实引擎调用可通/可断 | 首页→一级→二级→播放全链稳定 |
| L3 | `evaluate_spider_source` | 自动串联首页、一级、二级、搜索、播放的整体评分 | 站点长期稳定、所有线路都可播 |

`validate_spider` 只是结构校验，不应被写成“源可用”的证据；运行时结论必须来自 `test_spider_interface` 或 `evaluate_spider_source`。

## 九、关键设计哲学

1. **安全隔离**：所有JS源在 vm.createContext 沙箱中执行，无法访问Node.js原生API
2. **模板优先**：能走模板继承就不要手写，模板Object.assign合并保证了最小覆盖
3. **字符串规则优先**：能用CSS选择器字符串就不要写async function
4. **渐进增强**：从简单字符串规则 → 局部函数覆盖 → 全async函数，逐步增加复杂度
5. **缓存优化**：LRU + 多层缓存（moduleCache + ruleObjectCache + pageRequestCache）
6. **DS源格式**：`@header({...})` 元数据声明 + `var rule = {...}` 规则定义
