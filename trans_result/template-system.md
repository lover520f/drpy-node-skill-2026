# drpy 模板系统详解

## 一、概述

模板系统定义在 `libs_drpy/template.js`，提供 12+ 个内置模板。
模板是预配置的 rule 对象，包含 class_parse、url、searchUrl、推荐、一级、二级、搜索、lazy 等完整字段。

## 二、所有内置模板

| 模板名 | CMS类型 | URL模式 | class_parse | 适用场景 |
|---|---|---|---|---|
| **mx** | 苹果CMS旧版 | `/vodshow/fyclass--------fypage---/` | `.top_nav li;...` | 经典苹果CMS |
| **mxpro** | 苹果CMS Pro | `/vodshow/fyclass--------fypage---.html` | `.navbar-items li:gt(0):lt(10);...` | 新版苹果CMS Pro |
| **mxone5** | 苹果CMS One5 | `/show/fyclass--------fypage---.html` | `.nav-menu-items&&li;...` | mxone5主题 |
| **首图** | 首图CMS | `/vodshow/fyclass--------fypage---/` | `.myui-header__menu li:gt(0):lt(7);...` | 首图主题 |
| **首图2** | 首图CMS v2 | `/list/fyclass-fypage.html` | `.stui-header__menu li:gt(0):lt(7);...` | 首图v2主题 |
| **vfed** | VFed CMS | `/index.php/vod/show/id/fyclass/page/fypage.html` | `.fed-pops-navbar&&ul.fed-part-rows&&a;...` | VFed主题 |
| **海螺3** | 海螺CMS v3 | `/vod_____show/fyclass--------fypage---.html` | `body&&.hl-nav li:gt(0);...` | 海螺v3主题 |
| **海螺2** | 海螺CMS v2 | `/index.php/vod/show/id/fyclass/page/fypage/` | `#nav-bar li;...` | 海螺v2主题 |
| **短视** | 短视频 | `/channel/fyclass-fypage.html` | `.menu_bottom ul li;...` | 短视频站 |
| **短视2** | 短视频v2 | `/index.php/api/vod#type=fyclass&page=fypage` | — | API驱动短视频 |
| **采集1** | 采集站 | `/api.php/provide/vod/?ac=detail&pg=fypage&t=fyclass` | `json:class;` | 通用采集站 |
| **默认** | 通用兜底 | 空 | `#side-menu li;...` | 无法匹配时兜底 |

## 三、各模板关键差异

### 3.1 二级字典结构

所有模板的二级采用字典对象格式，但字段选择器不同：

```js
// mxpro 模板二级字典
二级: {
    title: 'h1&&Text;.module-info-tag-link:eq(-1)&&Text',
    img: '.lazyload&&data-original||data-src||src',
    desc: '.module-info-item:eq(-2)&&Text;.module-info-tag-link&&Text;...',
    content: '.module-info-introduction&&Text',
    tabs: '.module-tab-item',
    lists: '.module-play-list:eq(#id) a',
    tab_text: 'div--small&&Text',
}

// 首图2 模板二级字典 (带多级回退)
二级: {
    title: '.stui-content__detail .title&&Text;...',
    title1: '.stui-content__detail .title&&Text;...',  // title1是回退
    img: '.stui-content__thumb .lazyload&&data-original',
    desc: '.stui-content__detail p&&Text;...',
    desc1: '.stui-content__detail p:eq(4)&&Text;;;...',  // desc1是回退
    content: '.detail&&Text',
    tabs: '.stui-pannel__head h3',
    tabs1: '.stui-vodlist__head h3',  // tabs1是回退
    lists: '.stui-content__playlist:eq(#id) li',
}
```

### 3.2 推荐 double 差异

| 模板 | double | 推荐规则 |
|---|---|---|
| mx | true | `.cbox_list;*;*;*;*;*` (继承一级) |
| mxpro | true | `.tab-list.active;a.module-poster-item.module-item;...` |
| 首图 | true | `ul.myui-vodlist.clearfix;li;...` |
| 海螺3 | true | `.hl-vod-list;li;...` |
| 短视 | true | `.indexShowBox;ul&&li;...` |

`double: true` 表示推荐需要两层解析：先取外层容器，再从内层提取数据。

### 3.3 搜索规则差异

| 模板 | 搜索规则 | 说明 |
|---|---|---|
| mx | `*` | 继承一级规则 |
| mxpro | `body .module-item;...` | 独立搜索规则 |
| 短视2 | `json:list;name;pic;;id` | JSON接口搜索 |
| 采集1 | `*` | 继承一级 |

## 四、lazy 播放器类型

### 4.1 common_lazy (默认)

```js
// 通用免嗅探播放器 - 所有CMS模板默认使用
async () => {
    let html = await request(input);
    // 提取播放器配置: r player_xxx = {...}
    let hconf = html.match(/r player_.*?=(.*?)</)[1];
    let json = JSON5.parse(hconf);
    let url = json.url;
    // 解密处理
    if (json.encrypt == '1') url = unescape(url);
    if (json.encrypt == '2') url = unescape(base64Decode(url));
    // 判断直链/站外
    if (/\.(m3u8|mp4)/.test(url)) return {parse: 0, url};
    return url && url.startsWith('http') && tellIsJx(url) 
        ? {parse: 0, jx: 1, url} : input;
}
```

源码 caveat：当前 `template.js` 中的 `common_lazy` 会提取并计算 `result`，但返回处可能仍表现为 `input` 或经过 `playParseAfter()` 后处理的结果。排查播放问题时不要只按上面的简化模型判断，必须用 `get_resolved_rule(path)` 看最终 lazy/play_json，再用 `test_spider_interface(play)` 复测真实输出。

### 4.2 def_lazy (默认模板)

```js
async () => { return {parse: 1, url: input, js: ''} }
// 完全交由嗅探系统处理
```

### 4.3 cj_lazy (采集站模板)

```js
async () => {
    if (/\.(m3u8|mp4)/.test(input)) return {parse: 0, url: input};
    if (rule.parse_url.startsWith('json:')) {
        // 通过解析接口获取
        let html = await request(purl);
        let json = JSON.parse(html);
        if (json.url) return {parse: 0, url: json.url};
    }
    return rule.parse_url + input;  // 拼接解析地址
}
```

## 五、模板继承实战

### 5.1 自动模板匹配 (`模板: '自动'`)

```js
// 源中设置
rule = { 模板: '自动', host: 'https://example.com' }

// 框架自动行为:
// 1. 请求 host 页面
// 2. 用每个模板的 class_parse 尝试解析
// 3. 第一个成功解析出分类的模板被选中
// 4. 等同于手动设置了该模板名
```

### 5.2 最小覆盖模式（推荐）

```js
// 樱花动漫 - 整个源只需要7行自定义字段!
var rule = {
    title: '樱花动漫',
    模板: 'mxpro',         // 继承mxpro模板的所有规则
    host: 'http://www.yinghuadm.cn',
    url: '/show_fyclass--------fypage---.html',          // 仅覆盖url
    searchUrl: '/search_**----------fypage---.html',      // 仅覆盖searchUrl
    class_parse: '.navbar-items li:gt(1):lt(6);a&&Text;a&&href;_(.*?)\.html',  // 仅覆盖class_parse
    tab_exclude: '排序',
    searchable: 2,
    quickSearch: 0,
    filterable: 0,
}
```

### 5.3 模板修改函数

```js
var rule = {
    模板: 'mxpro',
    模板修改: async function(muban) {
        // 在执行模板继承之前修改完整模板 map
        muban.mxpro.一级 = '自定义选择器';
    }
}
```

`模板修改(muban)` 的输入不是当前 rule，而是所有模板组成的 map；它发生在继承之前，适合对某个模板做少量全局修正。继承合并仍是 `Object.assign(rule, templateRule, originalRule)`，所以源文件里显式写出的字段最终会覆盖模板字段。

### 5.4 继承优先级

模板继承使用 `Object.assign(rule, templateRule, originalRule)`：
1. 先应用模板的所有属性
2. **后用源rule覆盖** → 源的显式定义永远优先

## 六、模板判断决策树

```
站点URL
  ↓
guess_spider_template(url) 请求分析
  ↓
├── 命中内置模板 → 路线A: 继承模板 + 最小覆盖
│   ├── 验证class_parse是否生效
│   ├── 验证url翻页是否正确
│   ├── 验证searchUrl
│   └── 按需覆盖推荐/一级/二级/搜索
│
├── 不命中但有HTML → 路线B: 分析DOM结构
│   ├── 如果结构类似CMS → 手动指定模板
│   ├── 如果Tailwind/React等现代UI → 纯async函数
│   └── 如果数据在JSON → JSON模式(json:)
│
└── 页面基本为空(SPA) → 路线C: API驱动
    └── 网络分析 → 找到真实API → 全async函数
```

## 七、模板调试建议

1. `guess_spider_template(url)` 是启发式判断，只说明页面分类特征像某个模板，不保证 `url/searchUrl/二级/lazy` 全部正确。
2. 写入源后先用 `get_resolved_rule(path)` 查看继承后的最终字段，重点看 `class_parse`、`double`、`url`、`searchUrl`、`二级`、`lazy`、`play_json`。
3. 命中模板后仍要逐接口验证：home/class → category → detail → play → search；不要只凭模板名判断可用。
4. 首页推荐为空时优先查 `double` 与推荐容器层级；搜索为空时优先查真实搜索页结构，不要默认 `搜索: '*'` 一定可用。
5. 如果只少数字段不匹配，优先最小覆盖字段；不要复制整份模板再改。

## 八、理解 `*` 星号继承

在字符串规则中，`*` 表示继承一级规则的对应位置：

```js
// 一级: '列表;标题;图片;描述;链接;详情'
// 搜索: '*;*;*;*;*;*' = 完全继承一级
// 搜索: ';*;*;;*;*'  = 部分继承
// 推荐: '*'          = double:false且继承一级
```

当 `搜索: '*'` 或 `推荐: '*'` 时，框架自动调用 `getPP()` 从一级规则逐位继承。
