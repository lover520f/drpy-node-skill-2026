---
name: drpy-node-repo-upload
description: 适用于 drpy-node 源的仓库上传、替换、标签修正与上传前校验。用户提到“上传仓库”“替换上传”“改标签”“公开/私密”“仓库里的文件信息”“上传前检查”时使用。专注于确保上传动作规范、标签正确、结果可追踪。
---

# drpy-node Repo Upload

## 快速索引

| 用户意图 | 入口流程 | 终止条件 |
|---|---|---|
| 上传/替换源 | L1 校验 → A/B/C 档 → 用户确认 → upload → info 核验 | 上传后元数据一致 |
| 只检查不传 | L1/L2/L3 按目标验证 → 输出结论 | 不进入上传 |
| 只改标签 | house_verify → list 定位 → info 确认 → update_tags | 标签核验一致 |
| 改公开/私密 | house_verify → list 定位 → toggle_visibility → info 核验 | 可见性一致 |

## 执行契约

- 输入：本地源文件名/仓库文件关键词/file_id/cid/标签要求/是否公开。
- 输出：上传前 A/B/C + L1/L2/L3 依据，上传或修正后的 file_id/cid/tags/is_public。
- 原则：上传前先校验，用户怎么要求标签就怎么执行，上传后必须可追踪。

> drpy-node 的仓库发布守门 skill。
> 目标：**上传前先过红线检查，上传后结果可追踪，标签严格按用户要求执行。**

---

## 本 skill 只负责什么

- 上传到仓库
- 同名替换上传
- 标签修正
- 公开 / 私密切换
- 上传前检查与上传后回报

### 不负责什么
- 不负责深度修源
- 不负责播放专项排查
- 不负责新建源

---

## 适用场景

当用户说这些时启用：
- 上传到仓库
- 替换上传
- 改标签
- 仓库里这个文件的信息
- 改公开/私密
- 上传前帮我检查一下

## 上传前决策树

当用户说“上传仓库 / 替换上传 / 帮我先检查一下能不能传”时，优先按下面顺序快速决策：

1. **先看是不是本 skill 的边界内问题**
   - 如果用户其实是在要求修源 / 修播放 / 从零建源 → 先交回 workflow / play-debug / source-create
2. **先过最低校验线**
   - 至少跑：`drpy_check_syntax` + `validate_spider`
3. **再分 A / B / C 档**
   - A：建议上传
   - B：技术上可传，但不建议直接传
   - C：暂不应上传
4. **最后才执行上传动作**
   - 默认优先：`house_file(action='upload', auto_replace=true)`

### 强提醒
**不要把“技术上能上传”直接说成“建议现在上传”。**
如果用户目标是“修好再传 / 修到满分再传”，就必须把这层判断单独说清楚。

---

---

### 只检查 vs 检查后上传

| 用户表达 | 最低验证 | 允许结论 | 下一步 |
|---|---|---|---|
| “先检查能不能传” | L1 | 只能说语法/结构层面可进入下一步 | 不上传，除非用户继续要求 |
| “检查没问题就上传” | L2 | 可给 A/B/C 初判；A 才建议上传 | 展示确认模板后上传 |
| “最终版/修好后上传” | L3 | 可给完整 A/B/C 发布建议 | B/C 停止，A 再确认上传 |
| “先传再说” | L1 | B 档可执行但必须说明风险 | 用户确认后上传 |

---

## 上传前红线

满足任一就不要直接上传，除非用户明确要求“先传再说”：

- 语法报错
- 结构无效
- 详情为空
- 一级字段不规范
- 播放仍是假通过，但却准备当成已修好上传
- 标签规则未确认，但用户明显在意标签

---

## 上传档位与验证深度

上传建议必须同时给出 A/B/C 档和 L1/L2/L3 证据等级：

| 验证等级 | 必跑工具 | 可支持结论 | 适用场景 |
|---|---|---|---|
| L1 语法结构 | `house_verify` + `drpy_check_syntax` + `validate_spider` | 文件可被仓库接受的基础可能性 | 用户只要求“先检查能不能上传” |
| L2 单接口 | L1 + `test_spider_interface` 关键接口 | 关键链路局部可用 | 用户要求“修好这个问题再传” |
| L3 全流程 | L1 + `evaluate_spider_source` | 可给 A/B/C 完整发布建议 | 用户要求“最终版/修好后上传” |

| 档位 | 结论 | 最低证据 | 动作 |
|---|---|---|---|
| A | 建议上传 | L2；最终版要求 L3 | 用户确认后上传 |
| B | 技术可传但不建议 | L1 或 L2 | 说明风险，用户坚持才上传 |
| C | 暂不应上传 | 任意等级发现红线 | 不上传，转 workflow/play-debug |

**禁止**在只有 L1 时给出 A 档；只能说“语法结构层面可进入下一步”。

---

## 标签规则（速查版）

### 总原则
- 不自己脑补标签
- 不因为文件名带 `[优]` 就自动加 `优`
- 用户没明确说要什么时，默认从简

### 示例
- 用户说“只要 ds” → `ds`
- 用户说“ds 和优” → `ds,优`
- 用户说“别乱打标签” → 只保留明确要求的标签

### 发现标签错了怎么办
- 先用 `house_file(list)` 按文件名搜索定位仓库文件，获取 file_id
- 再用 `house_file(action='info', cid=...)` 确认当前标签
- 然后 `house_file(action='update_tags', file_id=..., tags='...')` 修正
- 回报最终标签

### 独立标签修正流程（不经过上传）
当用户仅要求改标签不上传时：
1. `house_verify` 验证仓库连通性
2. `house_file(list)` 搜索定位文件（源名不明确时）
3. 确认当前 tags
4. 执行 `update_tags`
5. 按「标签修正模板」回报结果

### 自动标签检测

`house_file(upload)` 内部有自动类型检测逻辑（详见 `references/references-upload-decision.md` 第三节）。

**用户没有明确要求标签时**，可利用自动检测作为默认值。但用户明确要求标签时，以用户要求为准。

---

## 替换上传 vs 新建上传

### 默认策略
优先：
```text
house_file(action='upload', auto_replace=true)
```

### 适合替换上传
- 同名文件已存在
- 用户要更新最新版
- 不需要保留并行新条目

### 只有这些情况才考虑新建
- 用户明确要求保留独立新条目
- 同名但本质是不同源
- 需要同时保留多个版本

---

## references：上传前判断参考

如涉及”这个源到底算不算修好、应不应该传”，必须参考：
- `references/references-upload-decision.md`

---

## 仓库对象定位规则

| 用户给的信息 | 定位方式 | 需要确认 |
|---|---|---|
| 本地源名/路径 | `list_sources()` 或直接校验路径 | 是否就是要上传的文件 |
| 仓库关键词 | `house_file(action='list', search='关键词')` | 多个匹配时让用户选 file_id |
| file_id | 用于 replace/update_tags/toggle_visibility | 操作对象名称和当前 tags/is_public |
| cid | `house_file(action='info', cid='...')` | info 返回的 file_id 是否与目标一致 |

如果 `list` 返回多个同名/近似文件，不要凭排序选择；必须列出候选并确认。
如果用户同时给出 file_id 与 cid 且二者不一致，停止操作并要求确认真实目标。
如果用户同时要求“只要 ds”又要求“保留原标签”，优先提问澄清，不自行合并。

### 非上传变更核验

`update_tags`、`toggle_visibility`、`replace` 与上传一样必须核验结果：

1. 操作前用 `house_file(info, cid=...)` 或 `house_file(list)` 记录当前对象。
2. 操作后再次 `house_file(info, cid='目标 cid')`；如果只有 file_id 没有 cid，先用 list/info 找到 cid。
3. 核验字段：file_id、cid、filename、tags、is_public。
4. 不一致时只报告差异并询问下一步，不静默二次修改。

---

## 边界条件处理

### 源文件名不明确
用户只说”这个源”而不提供具体源名或路径：
1. 先用 `list_sources()` 列出 `spider/js/` 下所有源文件
2. 结合用户描述（如”上次修的动漫源”）缩小范围
3. 确认最终文件名后再继续
4. 不要假设唯一匹配而跳过二次确认

### 仓库文件不明确
用户说”仓库里这个文件”但不提供 file_id：
1. 先用 `house_file(list)` 按文件名关键词搜索仓库文件
2. 获取匹配文件的 file_id 后再执行 update_tags / toggle_visibility 等操作
3. 不要混淆 `list_sources()`（本地文件）和 `house_file(list)`（仓库文件）

### house_verify 失败
仓库连接无法建立时：
1. 检查 `manage_config(get)` 确认 `HOUSE_TOKEN` 和 `HOUSER_URL` 配置
2. 如配置为空或错误，提示用户补充
3. 不要跳过验证直接执行上传操作

### 上传网络错误
上传请求因网络问题失败时：
1. 区分是服务端拒绝（4xx/5xx）还是连接超时
2. 连接性问题：提示稍后重试，不自动重试
3. 服务端拒绝：检查文件大小、格式、权限，必要时确认 TOKEN 有效性

### 标签更新失败
`house_file(update_tags)` 返回错误时：
1. 检查 file_id 是否正确（用 `house_file(list)` 交叉验证）
2. 检查 tags 格式是否合规（逗号分隔的字符串）
3. 确认 TOKEN 对该文件有修改权限

---

## 快速入口选择

根据用户意图选择入口：

| 用户意图 | 入口流程 |
|---------|---------|
| 上传/替换源 | 走 Step 1 → Step 6 完整流程 |
| 只检查不传 | 走 Step 1(校验) → Step 3 → 输出结论 → 终止（不进入 Step 4 上传） |
| 只改标签 | 走 house_verify → house_file(list) 定位 → update_tags → 回报 |
| 切换公开/私密 | 走 house_verify → house_file(list) 定位 → toggle_visibility → 回报 |

## 标准流程

### Step 1：确认源文件 + 验证仓库连接
- 源名不明确 → 先用 `list_sources()` 列出文件，二次确认后再继续
- `house_verify` — 验证仓库连通性；失败时按「边界条件处理」章节排查

### Step 2：检查是否满足上传条件
最少检查：
- `drpy_check_syntax`
- `validate_spider`

如用户要求”修好后再传”，建议补：
- `evaluate_spider_source`
- 或关键接口单测

### 🛑 检查点：确认检查结果再继续
在进入 Step 3 前，先向用户呈现：
- 语法/结构校验结果
- 初步 A/B/C 档判断
- 用户确认后再继续执行或终止

### 强制工具检查点
在给出“A / B / C 档判断”前，至少应明确自己已经做了哪类工具验证：
- 只做了 `drpy_check_syntax` / `validate_spider` → 只能说明“语法/结构层面”
- 做了 `evaluate_spider_source` 或关键接口单测 → 才能进一步说明“接口可用性层面”

**禁止**在完全没有说明验证依据时，直接下结论“建议上传 / 不建议上传”。

### Step 3：判断是否建议上传
必须输出 A/B/C 档位和 L1/L2/L3 验证依据：
- A：建议上传
- B：技术上可传，但不建议直接上传
- C：暂不应上传

根据用户意图分支：
- 用户目标是"只评估不传"或"修好再传"且当前为 B/C 档 → 输出结论后终止，不进入 Step 4
- 用户目标是立即上传且当前为 A/B 档 → 继续 Step 4

### 🛑 元数据变更确认点

以下操作会改变仓库可见状态或元数据，执行前必须展示目标对象并确认：
- `update_tags`
- `toggle_visibility`
- `replace`
- 带 `auto_replace=true` 的上传

确认模板：
```markdown
## 仓库操作确认
- 操作：upload / replace / update_tags / toggle_visibility
- 目标：file_id=... / cid=... / 文件名=...
- 当前 tags/is_public：...
- 目标 tags/is_public：...
- 验证依据：L1 / L2 / L3
```
### Step 4：执行仓库操作
确认后按用户目标执行：
```text
# 上传/同名替换
house_file(action='upload', path='spider/js/源名.js', auto_replace=true, tags='用户确认的标签', is_public=true/false)

# 仅标签修正
house_file(action='update_tags', file_id=目标ID, tags='用户确认的标签')

# 仅可见性切换
house_file(action='toggle_visibility', file_id=目标ID)
```

### Step 5：回报结果
必须回报：
- 文件名
- file_id
- cid
- tags
- 是否公开
- 是新上传还是替换上传
- 上传后核验：用返回的 `cid` 调 `house_file(action='info', cid=...)` 确认仓库记录与期望一致

### Step 5.5：上传后核验
上传或替换成功后，除非工具返回已经包含完整元数据，否则补一次：
```text
house_file(action='info', cid='上传返回的 cid')
```
核验重点：
- 仓库记录是否能查到
- tags 是否符合用户明确要求或自动检测预期
- is_public 是否符合用户要求
- file_id/cid 是否与上传返回一致

如果核验不一致，先回报差异，再询问是否修标签或可见性，不要静默二次修改。

### Step 6：必要时修标签
```text
house_file(action='update_tags')
```

### 🛑 检查点：确认操作结果
上传/标签修正完成后向用户呈现：
- 文件名、file_id、CID、tags
- 上传后 info 核验结果
- 是否公开
- 操作类型（新上传 / 替换上传 / 标签修正）
- 用户确认后可继续修标签或结束

---

## 推荐输出模板

### 上传结论模板
```markdown
## 上传前结论
- 语法：通过 / 未通过
- 结构：通过 / 未通过
- 关键接口：...
- 档位判断：A（建议上传）/ B（技术上可传，不建议）/ C（暂不应上传）
- 验证依据：L1(语法结构) / L2(单接口) / L3(全流程)
- 原因：...
```

### 上传结果模板
```markdown
## 上传结果
- 文件名：...
- 仓库 ID：...
- CID：...
- 标签：...
- 可见性：公开 / 私密
- 上传类型：新上传 / 替换上传
- info 核验：一致 / 不一致（差异：...）
```

### 标签修正模板
```markdown
## 标签修正结果
- 文件名：...
- 仓库 ID：...
- CID：...
- 旧标签：...
- 新标签：...
- 修正方式：update_tags
```

---

## 明确禁止事项

- 不要未测就上传
- 不要标签乱打
- 不要把半成品说成最终版
- 不要用户说只要 `ds` 时还附加别的标签
- 不要上传后只说“传好了”，不报 file_id / cid / tags
