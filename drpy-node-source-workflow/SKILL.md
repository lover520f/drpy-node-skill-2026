---
name: drpy-node-source-workflow
description: 适用于 drpy-node 源开发、修复、调试、测试与仓库上传的总控工作流。用户提到"修源""调试 drpy 源""测试某个源""详情为空""播放不通""搜索异常""上传到仓库""修复 P0 源""评估源可用性""DS源开发""lazy 解析""上传 ds 源""源不通""源无效""低分评估"时使用。优先用于 drpy-node 项目中的 DS 源，负责先评估、再分流到对应子 skill（播放调试/仓库上传/新建源），最后做结果汇总与上传决策。提供完整的分类排查路线（模板站/纯API站/签名接口站）和检查点确认机制。
---

# drpy-node Source Workflow

## 快速索引

| 输入/问题 | 诊断目标 | 分流 |
|---|---|---|
| 已有源评估低分 | 判断 A 规则不通 / B 串联断 / C 播放链 | 本 skill 修复或转 play-debug |
| 只有网址 | 判断新建还是修复 | 转 source-create |
| detail 正常但 play 异常 | 播放链专项 | 转 play-debug |
| 用户要求上传/替换/改标签 | 发布守门 | 转 repo-upload |

## 执行契约

- 输入：源名/源文件/URL/评估结果/用户目标。
- 输出：证据链分级诊断、分流路线、修复后验证结果、上传建议。
- 原则：先评估 → 再分流 → 再修复 → 再验证 → 最后给上传建议/结束。

## 调度优先级

当本地环境已安装本 Skill 时：
- 本地 Skill 优先级 **高于** drpy-node MCP 的通用 prompts
- 当用户只给网址、不给文件名时，总控层应主动分析站点并推导源名

### 强约束
如果本地 Skill 已覆盖场景，不允许让 MCP 通用 prompt 抢占主流程。

---

## 前置检查：通用排查 Checklist

以下排查项对模板站/签名接口站/纯 API 站都适用。只要源中有 async function，就必须先过这个清单。

完整参考：`../drpy-node-source-create/references/references-async-function-patterns.md`

| 优先级 | 检查项 | 症状 |
|---|---|---|
| **P0** | 源身份一致：文件路径、源名、host、`@header` 类型与当前目标匹配 | 修错源、影视/漫画/小说类型漂移 |
| **P0** | `this.input` 是 URL 不是响应 | `JSON.parse(this.input)` 报错含 URL |
| **P0** | `detailUrl` 是否设置 | 二级全部为空 |
| **P0** | POST 用 `body` 不是 `data` | 搜索/一级 POST 请求服务端不认 |
| **P0** | `searchUrl` 带 `**` | `this.KEY` 为空 |
| **P1** | 推荐完整聚合 | 推荐只有几条 |
| **P1** | 不手动拼 URL | 代码冗余且易出错 |
| **P2** | 不写重复属性 | 维护混淆 |

### 强约束
不要因为"这是模板站"就跳过这些检查。只要源中有 async function，就必须先过通用清单。

---

## 总控闭环（5 步）

### Step 1：识别输入类型
- 仅有网址 → 先分析站点并推导源名
- 已有源名/源文件 → 进入修复与评估流程

### Step 2：评估现状
1. `drpy_read_file` 读取源文件，确认源名、host、`@header` 类型和用户目标一致
2. `drpy_check_syntax` + `validate_spider` 基础校验
3. `get_resolved_rule`（模板站适用），记录继承后的 `class_parse/url/searchUrl/play_json/play_parse/sniffer/isVideo`
4. `evaluate_spider_source` 全流程评估
5. 记录证据链：每个结论必须能对应到工具输出（语法/结构/单接口/全流程）

### Step 3：判断失败类型

**诊断工具调用（MCP 真实字段示例）：**
```text
# 逐接口拆分验证
test_spider_interface(source_name='源名', interface='category', class_id='分类ID')
test_spider_interface(source_name='源名', interface='detail', ids='一级返回的真实 vod_id')
test_spider_interface(source_name='源名', interface='search', keyword='高频宽匹配词')
test_spider_interface(source_name='源名', interface='play', play_url='二级返回的播放地址', flag='线路名')
```

单接口测试必须使用上游真实返回值：category → vod_id → detail → play_url，不要手推 ID。

- **A. 规则本身不通** → 单接口 test 也失败
- **B. 评估器没串起来** → 单接口通但 evaluate 不通（见「专项排查参考」→「评估器失败分流」B类优先检查）
- **C. 主要卡在播放链** → detail 正常但 play 异常

### 🛑 检查点 1：确认诊断结论
在进入分流前，向用户呈现诊断摘要：

```markdown
## 诊断结论
- 首页：通/不通
- 一级（category）：通/不通
- 二级（detail）：通/不通
- 搜索：通/不通
- 播放：通/不通
- 失败类型：A（规则不通）/ B（评估串联）/ C（播放链）
- 证据链等级：L1=语法结构；L2=单接口；L3=全流程 evaluate
- 关键工具输出：...（列出支撑结论的工具和结果）
- 建议路线：继续本 skill 修复 / 转 source-create / 转 play-debug / 转 repo-upload / 结束
- 用户确认后再进入修复
```

### Step 4：分流
| 问题类型 | 分流目标 |
|---|---|
| 新建源 | `drpy-node-source-create` |
| 播放链问题 | `drpy-node-play-debug` |
| 上传仓库 | `drpy-node-repo-upload` |
| 站型判断不清 | 按 Step 3 重新诊断 |

### Step 5：收束
- `evaluate_spider_source` 重新评估
- 给出是否有效结论
- 决定是否建议上传/替换/回滚；需要仓库动作时转 `drpy-node-repo-upload`，不在 workflow 内直接操作 `house_file`

### 🛑 检查点 2：确认操作方案
在收束前向用户确认：

```markdown
## 修复结果摘要
- 已修复：...（接口和改动简述）
- 仍未通：...
- 建议上传：A（建议上传）/ B（技术上可传）/ C（暂不应上传）

用户确认后再转 repo-upload 处理上传/替换/标签等仓库动作，或结束
```

### 强约束
不要把 workflow 变成"所有事都自己做完"。它的价值在于：先评估 → 再分流 → 最后收束。

### 工具证据链分级

| 等级 | 已执行工具 | 能支持的结论 | 不能直接下的结论 |
|---|---|---|---|
| L1 | `drpy_check_syntax` + `validate_spider` | 源非语法残档、rule 结构基本合法，可进入接口拆测 | 源可用、已修好、建议上传 |
| L2 | `test_spider_interface` 单接口 | 某个接口真实通/断、定位 failure point；必须说明接口名和真实输入 | 全链路稳定、可作为最终版发布 |
| L3 | `evaluate_spider_source` | 首页→一级→二级→播放→搜索串联评分；可支持上传建议 | 站点长期稳定 |

结论必须带等级：例如“L2 证据显示 detail 通、play 断”，不要说成“整个源不通”。L1 只能作为继续拆测的门槛，不能包装成“可用/已修好/可上传”。

### 大改确认点

以下情况必须先给出方案并等用户确认：
- 从模板继承改为全 async。
- 删除大段已有规则或重写推荐/一级/二级/搜索。
- 需要登录态、Cookie、Token 或复杂签名逆向。
- 准备上传、替换仓库文件或回滚改动。

确认模板：
```markdown
## 修复方案确认
- 当前证据链：L1 / L2 / L3
- 根因判断：A / B / C
- 拟改字段：...
- 为什么不是更小改动：...
- 验证计划：...
```

### 分流输出契约

转交子 skill 前，必须带上最小上下文，避免子流程重新摸索：

| 分流目标 | 必带上下文 | 子流程完成后回收什么 |
|---|---|---|
| `source-create` | URL、推导源名、站型证据、用户目标 | 新源文件路径、五接口验证结果 |
| `play-debug` | source_name、真实 ids、play_url、flag、detail 输出摘要、继承后的 `play_json/play_parse/sniffer/isVideo` | lazy 根因、修改字段、play 复测结果 |
| `repo-upload` | 本地文件路径、源名、内容类型、A/B/C 建议、L1/L2/L3 证据、用户明确标签 | file_id、cid、tags、info 核验 |

### 分流执行规则

分流不是只在结论里写一句“转某 skill”，而是要把当前上下文带过去并暂停本流程的具体修复：

1. 命中新建源 → 调用/执行 `drpy-node-source-create`，输入 URL、推导源名、站型证据和用户目标；回收新源路径与五接口验证结果。
2. 命中播放链 → 调用/执行 `drpy-node-play-debug`，输入 source_name、真实 ids、play_url、flag、detail 摘要和播放相关 rule 字段；回收 lazy 根因与同一 play_url 复测结果。
3. 命中上传/标签 → 调用/执行 `drpy-node-repo-upload`，输入本地文件、源名、内容类型、A/B/C 建议、L1/L2/L3 证据和用户明确标签；回收 file_id/cid/tags/info 核验。
4. 子 skill 不可用时，按本 skill 的证据链先输出诊断，不要临时拼一个不完整替代流程。

---

### A/B/C 路由速判

| 当前证据 | 路由 | 禁止事项 |
|---|---|---|
| L1 失败 | 先修语法/结构 | 不跑 evaluate 冒充全链路 |
| category 失败 | 修一级/url/class_parse | 不直接修 detail/play |
| category 通、detail 空 | 修 detailUrl/二级字典/lists | 不转 play-debug |
| detail 通、play 假通过/空 | 转 play-debug | 不把 play success 当可播 |
| 单接口通、evaluate 首页/一级丢分 | B 类串联排查 | 不重写整源 |
| 五接口与 evaluate 都通过 | 进入上传建议 | 不绕过 repo-upload 确认 |

---

### 安全边界

| 动作类型 | 可直接做 | 需要确认 |
|---|---|---|
| 只读诊断 | 读源、语法结构校验、单接口测试、evaluate | 无 |
| 最小修复 | 改单个字段、修 selector、补 `detailUrl/searchUrl` | 改动前给出字段和验证计划 |
| 高风险重写 | 模板改全 async、删除大段逻辑、引入签名/登录态 | 必须等用户确认 |
| 仓库动作 | 无 | 上传、替换、改标签、公开/私密全部转 repo-upload 确认 |

### Reference 使用规则

本 skill 引用两类资料：
- 本地总控 reference：`references/references-template-summary.md`、`references/references-framework-internals.md`、`references/references-search-strategies.md`。
- 子 skill reference：`../drpy-node-source-create/references/...`，用于 async、纯 API、二级字典等细节。

如果某个 reference 文件不可达，不要中断诊断；按本文件内的 P0/P1 清单继续最小排查，并在输出中标注“reference 未读取，依据本 skill 内置规则处理”。

---

### 子 skill 不可用时的兜底

| 场景 | 本 skill 可兜底完成 | 必须停止的边界 |
|---|---|---|
| URL-only 新建 | 完成站型判断、源名推导、建源方案草案 | 实际写新源应回到 source-create 或等用户确认 |
| 播放链异常 | 确认 detail 稳定、提取 play_url/flag、判定是否假通过 | lazy 多分支重写交给 play-debug |
| 上传/标签 | 给出 A/B/C + L1/L2/L3 建议 | 仓库元数据变更必须交给 repo-upload |

这样即使子 skill 暂不可用，也能完成“诊断与分流”，但不冒充完成专门实现。

---

## 专项排查参考

### 模板站排查路线

当站点命中内置模板但评估不顺时，严禁直接大面积手写覆盖。

#### 排查顺序（按此顺序逐项检查）
1. 查模板默认定义 → `references/references-template-summary.md`
2. `class_parse` 是否残留覆盖？→ 补 `class_parse: ''`
3. `double` 是否导致推荐空？→ 补 `double: false`
4. 真实分类 `url` → 必须验证真实分类页和翻页
5. 真实搜索 `searchUrl`
6. 删除手写 一级/搜索，优先验证模板内置规则
7. `test_spider_interface` 拆开验证各接口
8. 最后才允许最小覆盖

#### 强禁止
未完成 1~7 之前，禁止一口气重写 推荐/一级/搜索/二级。

#### 关键原则
**不要把模板问题、分类 URL 问题、评估器串联问题，误当成单纯的一级选择器问题。**

---

### 纯 API 站排查路线

当页面源码为空、所有数据走 JSON API 时，走此路线（不套用模板站 checklist）。

完整参考：`../drpy-node-source-create/references/references-pure-api-async-site.md`

#### 排查顺序
1. `this.input` 是否被误当响应 → 必须 `await request(this.input)`
2. `detailUrl` 是否设置 → 纯数字 vod_id 必须设
3. 搜索 `searchUrl` 是否带 `**` → 否则 KEY 为空
4. POST 是否用了 `body` 而非 `data`
5. 外部 API 是否需要 Authorization
6. 推荐是否完整聚合

#### 强约束
纯 API 站排障路线与模板站完全不同，不要混用模板站的 checklist。

---

### 评估器失败分流

| 类型 | 表现 | 处理 |
|---|---|---|
| **A. 规则不通** | 单接口 test 也失败 | 修复规则本身 |
| **B. 评估没串起来** | 单接口通，评估不通 | 检查首页 class / class_parse / double / url / searchUrl |

#### B 类分数速映
`evaluate_spider_source` 各接口分数可快速映射到排查步骤：

| 丢失分数段 | 指向问题 | 对应步骤 |
|-----------|---------|---------|
| 首页 20 分 + 一级 20 分同时丢失 | `class_parse` 未命中导致 class 为空 | Step 2-3 |
| 仅首页 20 分丢失，一级正常 | `double` 配置不匹配 | Step 4 |
| 仅搜索 10 分丢失 | `searchUrl` 错误或搜索页 DOM 独立 | Step 5 |
| 二级 25 分丢失 | `detailUrl` 缺失或二级字段映射错误 | 见「二级 detail 规范」 |
| 播放 25 分丢失 | lazy 逻辑异常 | 转 `drpy-node-play-debug` |

#### B 类优先检查（顺序执行，逐项排查）
```
Step 1. 查 get_resolved_rule(path) → 看模板继承后的 class_parse/double/url 是否被覆盖
Step 2. 查首页 class 是否为空 → class_parse 是否命中（未命中 → 补 class_parse）
Step 3. 查 class_parse 是否残留覆盖 → 显式设 class_parse: ''
Step 4. 查 double 是否导致推荐为空 → 单层推荐优先 double: false
Step 5. 查分类 url / 搜索 searchUrl 是否真实
Step 6. 查手写 一级/搜索 是否反而扰乱模板内置链路（删除手写，先用模板内置）
```

#### B 类深层机制理解

B 类失败的根本原因在于**评估器串联机制**与**模板继承机制**的交互。排查时先记住完整初始化链：

```text
init() → getSandbox()/vm.createContext → 执行源文件获得 rule
      → handleTemplateInheritance() → initParse() → invokeMethod()
```

含义：
- 源运行在沙箱中，不能假设 `require/fs/process` 可用；只能用沙箱注入的 `request/pdfh/pd/CryptoJS/local` 等能力。
- `handleTemplateInheritance()` 发生在正式调度前，源中显式字段会覆盖模板字段。
- `initParse()` 会处理 host、headers、URL 模板、预处理等运行态配置；这些异常会表现为后续接口通断问题。
- `invokeMethod()` 再按 `推荐/一级/二级/搜索/lazy` 和字段类型分流。

```js
// 引擎 invokeMethod 分发逻辑 (drpyS.js)
switch (method) {
    case '推荐': injectVars = homeVodParse(...); break;  // 依赖 class + double
    case '一级': injectVars = cateParse(...); break;     // 依赖 url 模板渲染
    case '分类': injectVars = homeParse(...); break;      // 依赖 class_parse
}
```

- 首页空分类 → 后续 category 无 class_id 可用 → 全链断裂
- class_parse 未命中 → class 为空 → 评估器拿不到分类 ID
- `get_resolved_rule(path)` 可查看模板继承后的最终字段值，快速判断哪个字段被覆盖

详见 `references/references-framework-internals.md` 第七章节「调度分发机制」。

---

### 搜索排查

#### 搜索词适配（高频误判点）
当自动评估仅搜索失败，其他接口正常时：
1. 不要立刻判定"搜索规则失效"
2. 不要急着改 `searchUrl`
3. 应先换高频宽匹配词验证（通用词 `我的`，动漫站 `异世界`，或取一级 vod_name 片段）
4. 换词后正常 → 判定为评估参数问题

#### 搜索策略参考
处理搜索前必须先判断属于：
1. 原生搜索接口
2. suggest / 联想搜索 fallback
3. RSS fallback

参考：`references/references-search-strategies.md`

---

### 二级 detail 规范

完整参考：`../drpy-node-source-create/references/references-detail-dict-and-multiep.md`

#### 测试输入
- detail 测试**必须**使用一级真实返回的 `vod_id`
- 禁止主观简化为纯数字 id 或手推 id
- 正确顺序：先跑 category → 取真实 vod_id → 再测 detail

#### 字典槽位
- `desc` 五段有固定语义：备注;年份;地区;演员;导演

#### 多集只吐 1 集
先查 `lists` 容器层级（从 ul 下沉到 li），用 `debug_spider_rule(pdfa)` 验证各层项数。不要直接切 async。

```bash
# 排查示例
debug_spider_rule(url, '.anthology-list-box ul li a', pdfa)
# 先看 ul 层返回几项 → 再看 li 层返回几项
# 若 ul 层返回1项但 li 层返回多项 → lists 容器需从 ul 下沉到 li
```
如果 CSS 层级调整后仍只有 1 集，再用 `test_spider_interface(detail)` 配合真实 vod_id 复现，确认是否 `detailUrl` 缺失或二级字典字段映射错误。详见 `../drpy-node-source-create/references/references-detail-dict-and-multiep.md`。

#### 最小化原则
先保证 标题/描述/详情/图片/线路/列表，不强补 年份/地区/演员/导演。

---

### `*` 模板字段理解

当模板字段中出现 `搜索: '*'` 或含多个 `*` 的摘要写法时，必须先读：
- `references/references-template-summary.md`

#### 当前理解
- 单个 `*`：整体继承一级
- 多个 `*`：按分号位置逐位继承一级对应槽位

#### 强约束
不要在没理解 `*` 的源码级行为前，就机械改写成完整手写规则。

---

### parser 语法边界

当规则"看起来合理但接口异常"时：
1. 是否超出 parser 已明确支持的边界？
2. 应先修规则写法，而非要求引擎兼容
3. `||` 优先用于同一 selector 属性 fallback（`img&&data-original||src`）
4. 不要生成跨 selector fallback（`img&&data-original||img&&src`）

## 收尾输出模板

```markdown
## 修源结果
- 源：...
- 证据链等级：L1 / L2 / L3
- 根因类型：A 规则不通 / B 评估串联 / C 播放链 / 其他
- 已改动字段：...
- 验证结果：
  - home：...
  - category：...
  - detail：...
  - search：...
  - play：...
  - evaluate：.../100
- 上传建议：A 建议上传 / B 技术可传但不建议 / C 暂不应上传
- 下一步：转 play-debug / 转 repo-upload / 继续修复 / 结束
```

**先尊重引擎设定，再要求更宽容的兼容。**
