# drpy API 与全局函数参考

> 来源：drpy-node-mcp `get_drpy_libs_info` + 源码分析

## 〇、运行时边界

DS 源运行在 `vm.createContext` 创建的注入沙箱里，不是普通 Node.js 模块环境。源内应使用沙箱提供的全局能力（如 `request`、`req`、`pdfh`、`pd`、`pdfa`、`local`、`ENV`、`CryptoJS`、`base64Decode`），不要假设 `fs`、`process`、原生 `require` 或任意 npm 包可用。

async 方法里的 `this` 由引擎注入，`this.input` 是渲染后的 URL，不是响应体；真实响应需要 `await request(this.input)` 或使用 `req(this.input)` 获取。

## 一、核心请求函数

| 函数 | 说明 |
|---|---|
| `request(url, options?)` | 主HTTP请求，返回响应体(string)。options: `{headers, method, data/body, timeout, encoding}` |
| `post(url, options?)` | POST请求快捷方式 |
| `req(url, options?)` | 底层请求封装，返回 `{content, headers, code}` |
| `fetch(url, options?)` | fetch兼容的请求 |
| `reqs(urls, options?)` | 批量请求 |

## 二、解析函数

### HTML/XML 解析

| 函数 | 说明 |
|---|---|
| `pdfh(htmlOrNode, rule)` | 提取节点文本/属性。rule: `a&&Text` / `img&&src` / `.title&&Text` |
| `pd(htmlOrNode, rule, baseUrl?)` | 提取并标准化URL(自动解析相对路径) |
| `pdfa(htmlOrNode, selector)` | 获取节点列表(纯CSS选择器) |
| `pdfl(htmlOrNode, rule)` | 选集列表解析 |
| `jsp(baseUrl?)` | 创建jsoup解析器实例 |
| `jsoup` | jsoup类 |

### JSON 解析

| 函数 | 说明 |
|---|---|
| `pjfh(json, rule)` | JSON模式pdfh |
| `pjfa(json, rule)` | JSON模式pdfa |
| `pj(json, rule)` | JSON模式pd |

## 三、async 函数 this 上下文

当使用 async function 时，引擎通过 Proxy 注入 this：

```js
// 必须从 this 解构
一级: async function () {
    let { input, MY_URL, HOST, MY_CATE, MY_PAGE, MY_FL,
          pdfa, pdfh, pd, pjfa, pjfh, pj, KEY, fetch_params } = this;
    // input = 渲染后的URL
    // MY_CATE = 分类ID
    // MY_PAGE = 当前页码
    // MY_FL = 筛选条件对象
    // KEY = 搜索关键词
}
```

## 四、URL 工具

| 函数 | 说明 |
|---|---|
| `urljoin(base, path)` | 拼接/解析相对URL |
| `buildUrl(url, obj)` | 构建带query的URL |
| `getQuery(url)` | 解析query参数 |
| `parseQueryString(qs)` | query字符串→对象 |
| `buildQueryString(params)` | 对象→query字符串 |
| `urlDeal(url)` | URL处理 |
| `tellIsJx(url)` | 判断是否为解析URL |
| `encodeUrl(str)` | URL编码 |
| `urlencode(str)` | URL编码别名 |

## 五、结果格式化

| 函数 | 说明 |
|---|---|
| `setResult(d)` | 格式化视频列表返回。`d` 是 `{title, url, desc, pic_url, content}[]` |
| `setHomeResult(d)` | 首页结果格式化 |
| `setResult2(d)` | 替代格式化 |

`setResult` 内部将字段映射：
- `title` → `vod_name`
- `url` → `vod_id`
- `desc` → `vod_remarks`
- `pic_url` / `img` → `vod_pic`
- `content` → `vod_content`

## 六、接口返回契约

### 6.1 列表类接口

`home` / `推荐` / `一级` / `搜索` 最终都应产出 `vod_*` 字段列表，常见最小字段为：

| 字段 | 含义 | 来源 |
|---|---|---|
| `vod_name` | 标题 | `title` 或手动组装 |
| `vod_id` | 详情 ID/URL | `url` 或手动组装 |
| `vod_pic` | 封面 | `pic_url` / `img` |
| `vod_remarks` | 备注 | `desc` |
| `vod_content` | 简介 | `content` |

使用 `setResult(d)` 时可传 `{title, url, desc, pic_url, content}`，框架会映射成 `vod_*` 字段；全 async 手动返回时也应保持这些字段稳定。

### 6.2 detail 接口

二级详情必须产出稳定的播放字段：

- `vod_play_from`：线路名，多个线路用 `$$$` 分隔。
- `vod_play_url`：播放列表，与 `vod_play_from` 的线路数量一一对应。
- 同一线路内多集用 `#` 分隔，每集格式为 `标题$播放地址`。

如果 `vod_id` 是纯数字或接口 ID，必须设置 `detailUrl`，否则引擎无法渲染 `this.input`。

### 6.3 lazy/play 接口

`lazy` 可以返回字符串或对象。字符串会进入 `playParseAfter()` 自动判断；对象可显式控制：

| 返回 | 语义 |
|---|---|
| `{parse:0, url:m3u8}` | 直链免嗅探 |
| `{parse:1, url:input}` | 交给嗅探 |
| `{parse:0, jx:1, url:pageUrl}` | 站外解析 |
| `{parse:0, url:'novel://...'}` | 小说协议 |
| `{parse:0, url:'pics://...'}` | 漫画协议 |
| `{parse:0, url:'push://...'}` | 投屏/网盘协议 |

`play_json` 会在播放后处理中影响最终 `parse` / `jx` / `url`，当 lazy 返回和接口最终输出不一致时要检查模板继承后的 `play_json`。

## 七、加密与编解码

| 函数 | 说明 |
|---|---|
| `base64Encode(str)` / `base64Decode(str)` | Base64 |
| `md5(str)` / `md5X(str)` | MD5 |
| `aesX(data, key, iv)` / `aes(data, key, iv)` | AES加解密 |
| `desX` / `des` | DES加解密 |
| `rsaX` / `RSA` | RSA加解密 |
| `rc4Encrypt/Decrypt` / `rc4` | RC4 |
| `gzip(str)` / `ungzip(str)` | Gzip压缩 |
| `CryptoJS` | 完整CryptoJS库 |
| `JSEncrypt` / `NODERSA` | RSA库 |
| `forge` | node-forge库 |
| `CryptoJSW` | WASM加速版CryptoJS |

## 八、User-Agent 常量

| 常量 | 值 |
|---|---|
| `MOBILE_UA` | `Mozilla/5.0 (Linux; Android 12; ...)` |
| `PC_UA` | `Mozilla/5.0 (Windows NT 10.0; Win64; x64) ... Chrome/...` |
| `UC_UA` | UC浏览器UA |
| `IOS_UA` | iOS Safari UA |
| `UA` | 默认UA |
| `randomUa.generateUa()` | 随机UA生成 |

## 九、其他工具函数

| 函数 | 说明 |
|---|---|
| `log(msg)` / `print(msg)` | 日志输出 |
| `sleep(ms)` | 异步等待 |
| `deepCopy(obj)` | 深拷贝 |
| `computeHash(data)` | 计算哈希 |
| `jsonToCookie(obj)` / `cookieToJson(str)` | Cookie转换 |
| `toBeijingTime(ts?)` | 转北京时间 |
| `fixAdM3u8Ai(url, content, headers?)` | 修复带广告的m3u8 |
| `forceOrder(list)` | 强制排序 |
| `ENV.get(key)` / `ENV.set(key, val)` | 环境变量 |
| `local.get(store, key)` / `local.set(store, key, val)` | 本地持久化存储 |
| `OcrApi(img)` | OCR识别 |
| `simplecc` | 简繁转换 |
| `DataBase` / `database` | SQLite数据库 |
| `pupWebview` | Puppeteer浏览器实例(如可用) |
| `batchFetch(urls)` | 批量并发请求 |
