# drpy-node Source Workflow 专项参考

时间：2026-04-18 14:27 CST
用途：专门服务于“总控工作流”阶段，帮助在评估失败时快速判断是模板问题、入口问题还是规则本身问题。

---

## 1. 总控层的标准分流

### 情况 A：首页 class 为空
优先检查：
1. 模板残留 `class_parse`
2. `class_name/class_url` 是否被覆盖

### 情况 B：首页 list 为空，但页面明明有推荐卡片
优先检查：
1. 模板默认 `double`
2. 推荐是单层还是双层

### 情况 C：一级不通
优先检查：
1. 分类 `url` 是否真实
2. 是否该删掉手写 `一级` 回到模板内置
3. 分类页节点是否真实存在

### 情况 D：搜索不通
优先检查：
1. `searchUrl` 是否真实
2. 是否该删掉手写 `搜索` 回到模板内置
3. 搜索页容器是否和首页/分类容器不同

---

## 2. 模板内置优先判断表

| 场景 | 优先动作 |
|---|---|
| 一级不通且当前站明显命中模板 | 先删手写 `一级` 测模板内置 |
| 搜索不通且当前站明显命中模板 | 先删手写 `搜索` 测模板内置 |
| 首页推荐为空但真实节点存在 | 先测 `double:false` |
| 首页 class 为空但规则里写了分类 | 先测 `class_parse:''` |

---

## 3. 评估器失败时必须拆测的接口

优先用：
- `test_spider_interface(home)`
- `test_spider_interface(category)`
- `test_spider_interface(detail)`
- `test_spider_interface(play)`

目的：
- 区分“规则本身不通”
- 和“自动评估串联没接上”

证据边界：
- L1：`drpy_check_syntax` / `validate_spider` 只证明结构可检查，不能说已修好或可上传。
- L2：`test_spider_interface` 是真实引擎单接口测试，必须记录接口名、真实 `class_id/ids/play_url/flag`。
- L3：`evaluate_spider_source` 依赖 localDsCore 串联 home→category→detail→play→search，可作为上传建议依据，但不保证站点长期稳定。

---

## 4. MCP 工具边界

| 工具 | 适合判断 | 边界 |
|---|---|---|
| `analyze_website_structure` | 静态 DOM 结构、候选选择器 | 页面可能截断，不能证明动态 API 或播放可用 |
| `debug_spider_rule` | 单个 cheerio/pd 规则是否命中 | `pdfa` 模式只传纯 CSS，不能把完整分号规则塞进去 |
| `test_spider_interface` | 单接口真实引擎输出 | 输出可能截断；结论只覆盖该接口和该输入 |
| `evaluate_spider_source` | 全流程串联评分 | 依赖上游结果自动取下游参数，断点要回到单接口拆测 |

---

## 5. 强纪律

### 纪律 A
不要把模板问题、入口问题、评估器串联问题误判成选择器问题。

### 纪律 B
不要在没验证模板内置规则前，就大面积手写覆盖。

### 纪律 C
上传、替换、改标签不在 workflow 内直接执行；必须带本地路径、源名、内容类型、A/B/C 建议、L1/L2/L3 证据和用户明确标签转 repo-upload。
