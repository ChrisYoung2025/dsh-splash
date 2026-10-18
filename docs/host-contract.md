# 宿主桥协议（把片头接进你自己的壳）

三个片头都是**单个 HTML 文件**，不依赖构建、不依赖 npm，只用浏览器的 canvas / CSS / WebAudio。
通过 `postMessage` 与宿主通信，通过 `window.dsh550c` 接受宿主的指令。

## 1. 网页 → 宿主

统一从 `window.chrome.webview.postMessage`（WebView2）发出；在普通浏览器里这些消息会被丢掉，不影响播放。

| 消息 | 含义 | 宿主该做什么 |
|---|---|---|
| `{ "t": "ready" }` | 文档已就绪，可以下播放指令 | 保存状态，立刻发 `play(...)` |
| `{ "t": "done" }` | 已经淡到片头底色，可以收起遮罩 | 移除 / 隐藏承载片头的窗口或 iframe |

```js
// WebView2 (C#)
webView.CoreWebView2.WebMessageReceived += (s, e) => {
    using var doc = JsonDocument.Parse(e.WebMessageAsJson);
    string type = doc.RootElement.GetProperty("t").GetString();
    if (type == "ready") webView.CoreWebView2.ExecuteScriptAsync("window.dsh550c.play('simple')");
    if (type == "done")  splashPanel.Visible = false;
};
```

```js
// 浏览器 iframe（同源时）
const win = iframe.contentWindow;
win.addEventListener("message", e => { /* 同上 */ });
```

> 用 `file://` 打开时，iframe 属于不透明源，父页面拿不到 `contentWindow`。
> 所以每个片头都自带「没有宿主就自己播一遍」的兜底，`demo/index.html` 靠改 `iframe.src` 切换，不需要跨文档调用。

## 2. 宿主 → 网页

三个片头都挂同一个命名空间 `window.dsh550c`（历史原因，与 DSH 桌面壳保持兼容）：

| 方法 | 参数 | 说明 |
|---|---|---|
| `play(mode)` | `"simple"` / `"full"` / `"gundam"` / `"matrix"` | 开始播放。未知值按该文件自己的默认档处理 |
| `skip()` | — | 立刻开始交接（用户点画面 / 按 Esc 时也应调用） |
| `note(text)` | 任意字符串 | 把宿主侧的启动阶段文字送进片头底部状态行（完整档同时写进终端日志） |
| `hurry()` | — | **可选**，只有数字雨档实现：主页面已经就绪时把剩余停留压到最短，片尾卡与飘散仍会播完 |

```js
window.dsh550c.play("matrix");
window.dsh550c.note("正在启动本地服务…");
setTimeout(() => window.dsh550c.hurry(), 3000);
```

## 3. 各片头的行为差异

| 文件 | 支持的 `play()` 参数 | 自带自动播放 | 额外调试参数 |
|---|---|---|---|
| `splashes/matrix/index.html` | 忽略（固定数字雨） | 是（无宿主时自动播） | `?speed=2` 倍速、`?speed=0.5` 慢放 |
| `splashes/550c/index.html` | `simple`（默认）/ `full` / `gundam` | 是（`?mode=` 指定档位） | `?mode=simple｜full｜gundam` |

## 4. 宿主侧的三个注意点

1. **自动播放**：Chromium 系内核只放行**静音**视频的自动播放，所以高达档强制 `muted`；
   别为了自动播放给内核加 `--autoplay-policy=no-user-gesture-required`，会影响正常导航。
2. **首帧**：先给承载片头的控件铺上片头底色（数字雨 `#050403`、550C `#0a0805`），
   等 `ready` 再显示，避免白屏闪一下。
3. **兜底时限**：片头脚本万一卡住（渲染进程崩、资源缺失），宿主必须在 ~30s 后强制收起遮罩，
   不能把主界面一直挡着。
