# 特殊内容类型协议参考

> 用途：编写非影视类型源时，lazy 返回值的特殊协议处理

## 一、漫画类型

```js
// @header 中声明类型
@header({ '类型': '漫画' })

// lazy 返回 pics:// 协议
lazy: async function () {
    let html = await request(input);
    let arr = pdfa(html, '.single-content&&img');
    let urls = arr.map(it => pdfh(it, 'img&&data-src'));
    return {
        parse: 0,
        url: 'pics://' + urls.join('&&'),
        js: '',
    };
}
```

**要点**：
- 使用 `pics://` 前缀，后面用 `&&` 连接所有图片URL
- `parse` 必须为 0
- 典型选择器：`.comic-page img`、`.single-content img`

## 二、小说类型

```js
// @header 中声明类型
@header({ '类型': '小说' })

// lazy 返回 novel:// 协议
lazy: async function () {
    let html = await request(content_url);
    let json = JSON.parse(html);
    let ret = JSON.stringify({ title, content: json.data.content });
    return { parse: 0, url: 'novel://' + ret, js: '' };
}
```

**要点**：
- 使用 `novel://` 前缀
- 后面跟 `JSON.stringify({title, content})`
- content 是 HTML 格式的章节内容
- `parse` 必须为 0

## 三、音频/音乐类型

```js
// @header 中声明类型
@header({ '类型': '听书' })

// lazy 返回直链
lazy: async function () {
    let html = await request(input);
    let music = html.match(/var\s+music\s*=\s*(\{[\s\S]*?\})/)[1];
    music = JSON5.parse(music);
    input = urljoin(input, "//mp4.example.com/" + music.file + ".m4a");
    return input;  // 直接返回字符串，框架自动判断为 parse:0
}
```

**要点**：
- 直接返回 m4a/mp3 直链 URL（字符串形式）
- 框架的 `playParseAfter` 会自动识别 `.m4a` / `.mp3` 后缀并设 `parse:0`
- 也支持返回 `{parse:0, url: '...'}` 对象形式

## 四、投屏类型

```js
// push:// 协议
return { parse: 0, url: 'push://' + playUrl, js: '' };
```

用于将视频推送到投屏设备。

## 五、返回格式汇总

| 内容类型 | 协议 | lazy 返回格式 |
|---------|------|-------------|
| 视频直链 | — | `{parse:0, url: 'https://...m3u8'}` 或字符串 |
| 站外解析 | — | `{parse:0, jx:1, url: 'https://...'}` |
| 小说 | `novel://` | `{parse:0, url: 'novel://' + JSON.stringify({title, content})}` |
| 漫画 | `pics://` | `{parse:0, url: 'pics://url1&&url2&&url3'}` |
| 音乐 | — | `{parse:0, url: 'https://...mp3'}` 或字符串 |
| 投屏 | `push://` | `{parse:0, url: 'push://...'}` |

## 六、建源与验证标准

特殊内容源的成功标准不是“返回 m3u8/mp4”：

1. `@header` 的 `类型` 必须与内容一致，不能用普通影视 metadata 包装漫画/小说/听书。
2. detail 应产出章节、曲目、文件或分享资源；`vod_play_from` / `vod_play_url` 仍需线路与列表结构稳定。
3. play/lazy 的成功标准按协议判断：漫画看 `pics://` 图片列表，小说看 `novel://` 或正文内容，音乐看 mp3/m4a 直链，网盘/投屏看 `push://` 或网盘专用输出。
4. 验证顺序仍是 home → category → detail → play；只是 play 的断言从“视频直链可播”换成“目标协议/内容有效”。
5. 上传或打标签时不要在 create 阶段自行决定仓库 tags，交给 repo-upload 按用户要求和验证证据处理。
