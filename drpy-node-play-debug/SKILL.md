---
name: drpy-node-play-debug
description: 适用于 drpy-node 源的播放链路排查与 lazy 修复。用户提到“播放不通”“lazy 不对”“是不是直链”“play.html 被当直链”“iframe 提取”“m3u8 提取”“parse:0/1 判断”“假播放”“站外解析”时使用。专注于判断播放结果是否真实可播，并优先修复 lazy 逻辑。
---

# drpy-node Play Debug

## 调度优先级（新增强规则）

当本地环境已安装本 Skill 时：

- 本 Skill 的优先级 **高于** drpy-node MCP 的通用调试 prompt
- 通用 MCP 调试流程仅作无本地 Skill 时的兜底说明
- 播放链排障必须优先按本 Skill 的判断顺序执行，而不是先按通用 prompt 线性试错

---

## 与其他 drpy-node skills 的边界（新增）

### 本 Skill 负责什么
- 判断播放结果是否真实可播
- 修复 lazy 逻辑
- 判断 `parse:0/1`、`jx:0/1` 是否合理
- 提取 iframe / m3u8 / 站外解析链接

### 本 Skill 不负责什么
- 不负责从零写完整源
- 不负责系统性重建首页/一级/二级规则
- 不负责仓库上传决策

### 何时交回 `drpy-node-source-workflow`
如果问题已经不只是播放链，而是：
- 一级/二级/搜索也同时异常
- 自动评估整体很差，需要重新分流
- 需要决定是否上传 / 替换 / 回滚

则应交回总控 workflow，不要在 play-debug 内越权扩散。

---

## 新增 references：模板系统里的默认 lazy 逻辑

来源参考：`E:\gitwork\drpy-node\libs_drpy\template.js`

### 1. `common_lazy`
模板系统里的通用免嗅探 lazy 逻辑大致是：

```js
let html = await request(input);
let hconf = html.match(/r player_.*?=(.*?)</)[1];
let json = JSON5.parse(hconf);
let url = json.url;

if (json.encrypt == '1') {
  url = unescape(url);
} else if (json.encrypt == '2') {
  url = unescape(base64Decode(url));
}

if (/\.(m3u8|mp4|m4a|mp3)/.test(url)) {
  result = { parse: 0, jx: 0, url };
} else {
  result = url && url.startsWith('http') && tellIsJx(url)
    ? { parse: 0, jx: 1, url }
    : input;
}
```

#### 这意味着什么
- 模板默认会先抓播放页 HTML
- 然后解析 `player_*` 配置
- 会处理常见加密字段：
  - `encrypt == 1` → `unescape`
  - `encrypt == 2` → `base64Decode + unescape`
- 如果拿到的是直链媒体：
  - `m3u8/mp4/m4a/mp3`
  - 应返回：
    ```js
    { parse: 0, jx: 0, url }
    ```
- 如果拿到的是站外解析链接，且 `tellIsJx(url)` 为真：
  - 模板默认倾向：
    ```js
    { parse: 0, jx: 1, url }
    ```
- 如果都不是，模板可能直接回退为 `input`

---

### 2. `def_lazy`
模板系统默认嗅探 lazy：

```js
return { parse: 1, url: input, js: '' }
```

#### 这意味着什么
- 如果站点播放页本身可交给前端/外部解析器处理
- 默认就是：
  ```js
  parse: 1
  url: input
  ```
- 所以用户看到 `parse:1 + 网站播放页链接`，不一定是错，可能正是模板默认策略

---

### 3. `cj_lazy`
采集站 lazy 逻辑：
- 直链 `m3u8/mp4` → `parse:0`
- `parse_url` 以 `json:` 开头 → 先请求 JSON 解析接口再取 `json.url`
- 否则拼接 `rule.parse_url + input`

#### 这意味着什么
采集站排查时，不能只看最终 `url`，还要看：
- `rule.parse_url` 是否配置
- 是不是 `json:` 解析接口
- 返回 JSON 里是否真的有 `url`

---

## 播放排障时的强制判断顺序

### A. 先判断当前 lazy 更接近哪类模板默认逻辑
1. `common_lazy` 风格：
   - 播放页里有 `player_*` JSON
   - 可能有 `encrypt`
   - 可能需要解密 `json.url`
2. `def_lazy` 风格：
   - 站点播放页本身交给解析系统
   - 直接 `parse:1 + input`
3. `cj_lazy` 风格：
   - 采集站 / 解析接口 / JSON 解析链

---

### B. 再判断返回应该是直链还是解析页
#### 直链媒体
如果最终拿到的是：
- `m3u8`
- `mp4`
- `m4a`
- `mp3`

优先应返回：
```js
{ parse: 0, jx: 0, url }
```

#### 站外解析链接
如果最终拿到的是外部解析地址，而不是直链媒体：
```js
{ parse: 0, jx: 1, url }
```

#### 网站播放页本身
如果当前源策略就是把网站播放页交给解析系统：
```js
{ parse: 1, url: input }
```

---

## 新增强提醒

### 提醒 1
**不要把 `parse:1 + 网站播放页链接` 一概判成错。**
如果该站模板默认就是 `def_lazy` 风格，这可能是合理输出。

### 提醒 2
**不要把站外解析链接误判成直链。**
拿到 http 链接不等于拿到直链，先判断是不是 `m3u8/mp4` 等媒体文件。

### 提醒 3
**遇到 `player_*` 配置时，优先查 `encrypt`。**
很多站不是没链接，而是 `json.url` 还没解密。

### 提醒 4（新增）
**不要把规则越界写法先甩锅给引擎。**
如果播放链里的字段/规则写法超出 parser 明确支持边界，应先检查：
- 这是不是规范写法
- 模板/现有源是否这样写
- 是否应先修规则和说明，而不是直接要求引擎兼容

---

## 建议后续 MCP 增强
理想情况下，MCP 最好还能提供：
- 读取模板默认 lazy 定义摘要
- 展示当前模板实际继承到的 lazy 类型

这样 AI 在判断 `parse:0/1/jx:0/1` 时，不会脱离模板默认上下文盲判。
