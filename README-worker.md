# worker_速度端点.js 使用说明

给你的 edgetunnel 加两个测速端点，让 CFIP优选 能测到「穿过你 Worker 回源」的真实速度，以及一个和 clash 面板同口径的延迟。

> **v1.8 起不用手贴了。** 打开 `cfip_gui.exe` → 菜单 **工具 → 给 Worker 加测速端点…** → 选你的 worker 文件 → 程序在同目录生成 `原名_已注入` → 全选复制粘进 CF → Deploy。
>
> 下面第二章起的手工流程只是备查（老版本工具、或想自己改片段时用）。

---

## 一、它是干什么的

装完后你的 Worker 多两个路径：

| 端点 | 干什么 | 返回头 |
|---|---|---|
| `/__speed?mb=N` | 回源 Google 视频 CDN 拉一个大文件，边拉边流式返回 | `X-Speed-Bytes`<br>`X-Upstream-Mode`<br>`X-Upstream-Landed` |
| `/__gen204` | Worker 内部 `fetch('https://www.gstatic.com/generate_204')` 再回 204 | `X-G204-Status`<br>`X-Upstream-Ms` |

为什么要绕这一圈：CFIP优选 直接测 jsdelivr/unpkg 上的静态文件，测到的是 **CF 自家的 CDN 缓存通道**，和你看 YouTube 走的 Worker `fetch` 出网通道完全不是一回事。加了这个端点，测速请求才会真的穿过「CF 边缘 → 你的 Worker → 外网」再回来。

代价是数字会变得很难看（静态源 200Mbps，真实路径常常 0.4~1.3Mbps），但那是真的。

---

## 二、手工流程（备查）

### 第一步：找到位置

下载你的 worker 代码（CF 后台 → Workers & Pages → 你的 Worker → 编辑代码 → 全选复制）。edgetunnel 里大概是：

```js
export default {
	async fetch(request, env, ctx) {
		// ← 就在这里，第十九行下面一行
		let 请求URL文本 = request.url.replace(...);
```

要找的是 `async fetch(request, env, ctx) {` 这一行**结尾的那个 `{`**。整段端点插在这个 `{` 的下一行。

### 第二步：腾出一行空位

光标放到 `{` 之后按一次回车，让原本的第 20 行变成第 21 行。**这一步是为了后面粘错了好回退**，不做也能粘。

### 第三步：整块复制粘贴

把 `worker_速度端点.js` 从头到尾（**连最外层的 `{ }`、连首尾的 `// ==== 测速端点开始/结束`===` 注释一起**）粘进那个空行。

三个注意：

- **不要只粘里面那半截**。片段最外层是一对 `{ }` 包着的裸块，不是 `if...else` 的一部分。少粘一层括号会把 `async fetch` 的函数体提前关掉，Deploy 时报 `SyntaxError: Unexpected token 'else'`
- **原有代码一行都别动**。片段只新增，不修改
- **缩进随便**。JS 不在乎，粘进去变丑不影响运行

### 第四步：保存并部署

点 Deploy。如果报 `SyntaxError`，八成是没粘全（少了最外层 `{` 或 `}`），删掉重粘。

### 第五步：验证

```bash
curl "https://你的域名/__speed?mb=1" -o NUL -D -
```

看响应头里有没有 `X-Speed-Bytes` 和 `X-Upstream-Mode`。有就成功了。

```bash
curl "https://你的域名/__gen204" -D -
```

应该返回 `204`，头里有 `x-g204-status: 204` 和 `x-upstream-ms: 20` 左右。

**看不到这两个头的三种情况**

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 返回 200 + 613 字节的 `Welcome to nginx!` | 片段没粘 / 没 Deploy / 粘到别的地方了 | 重来一遍 |
| 返回 200 + 404 页面 | edgetunnel 把它当普通路径处理了 | 确认粘在 `async fetch(...) {` 的**第一行** |
| 返回 500 / 1101 | 片段粘贴位置把原函数体切断了 | 删掉重粘，注意最外层 `{ }` |

---

## 三、参数说明

`/__speed` 支持四个 URL 查询参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `mb` | 50 | 拉多少 MB，1~200。CFIP优选 默认给 200 |
| `u` | 空 | 填 `dl` 换 Google 静态 CDN（`dl.google.com` 上的 chrome deb 包），更稳但更慢 |
| `n` | 0 | 视频 CDN 候选轮转，0~2。短读或 502 时换一个 |
| `token` | 空 | 只有你在 Worker 环境变量里设了 `SPEED_TOKEN` 才需要 |

`/__gen204` 不接受参数，也不占下载额度。

---

## 四、为什么这么设计

**用 `globalThis.fetch` 而不是 `fetch`。** Worker 里 `fetch` 会被路由到本 Worker 的处理函数，直接调会自己请求自己。`globalThis.fetch` 拿的是运行时原始实现，才能真正出网。

**为什么要手动截断。** Google 视频 CDN（`redirector.gvt1.com`）返回 302，跳转时会丢掉 `Range` 头，上游可能给你整个 781MB。所以拿到响应体后挂一个 `TransformStream`，数到 `mb` 字节就 `close()`，剩下的不再往下读。

**为什么响应是 chunked 而不是 Content-Length。** 因为包了 `TransformStream`，CF 会强制改写 `Transfer-Encoding: chunked` 并丢掉 `Content-Length`。所以 CFIP优选 **不能**用 `Content-Length` 判断端点是否存在，改看 `X-Speed-Bytes`。

**为什么三个视频 CDN 候选依次试。** 三个都是 `redirector.gvt1.com` 上的不同大小包（ndk-r27 781MB / emulator 412MB / build-tools 58MB），但不同机房的落地情况不一样，会短读或 302 循环。失败的换下一个。

**为什么默认走视频 CDN 不走静态 CDN。** 静态 CDN（`dl.google.com`）走的是 Google 自家的 CDN 边缘，离你近、有缓存，测出来的数会虚高；视频 CDN 走的是真实回源，更接近刷 YouTube 的实际体验。要更稳的结果就用 `u=dl`。

---

## 五、额度消耗

一次测速 = 一次 Worker 请求。免费版每天 10 万次。

20 个 IP × 重复 3 次 = 60 次请求，占 0.06%。不用算着用。

`/__speed` 的下载字节不消耗 Worker 额度，只消耗 CF 的出口流量（免费、无上限）。

---

## 六、常见问题

**Q：装完 Worker 就打不开了，一直 1101？**

片段粘错位置把 `async fetch` 的函数体切断了。删掉重粘，确保：①连最外层 `{ }` 一起粘；②粘在 `async fetch(request, env, ctx) {` 的**下一行**；③原有代码一个字没动。

**Q：Deploy 报 `SyntaxError: Unexpected token 'else'`？**

片段最外层的 `}` 被漏掉了，导致原代码后半截的 `else if` 变成孤儿。重粘一次，从 `// ==== 测速端点开始 ====` 到 `结束 ====` 整段一起复制。

**Q：`/__speed` 一直「上游短读」？**

上游给了一点数据就断。换 `n=1` 或 `n=2` 换一个候选源，或者 URL 加 `u=dl` 走静态 CDN。CFIP优选 里就是「高级 → 回源线路」换一个。

**Q：测速速度只有 1Mbps，正常吗？**

正常。**CF 免费版 Worker 的 `fetch` 出网通道真实上限就是 1~2Mbps**，换什么上游都一样（实测 gvt1 和 dl.google.com 都在 0.5~1.7Mbps）。瓶颈在 Worker 出口，不在上游。

**Q：`/__gen204` 是什么？**

和 clash / mihomo / Shadowrocket 测节点延迟完全相同的路径：Worker 内部去 `fetch` gstatic 的 `generate_204`，拿到 204 就立刻回 204 给你。实测 Worker 回源那一段只多花 18~47ms，所以测出来的数和 clash 面板基本能对上。

**Q：为什么 `/__speed` 不占下载额度？**

`X-Speed-Bytes` 这个头就够了，不需要真的把 200MB 全下发。CFIP优选 读到够用的字节就断开。

---

## 七、编码：这里最容易踩的坑

### `_worker.js` 末尾那一大段"乱码"是什么

不是加密，是**一个超长块注释**：从 `/*` 开始，内容是希伯来文圣经原文（Genesis 到历代志下），到 `*/` 结束。4.7MB，占全文件 93%。

**它对运行没有影响**，删掉程序照跑。但会让 CF 编辑器很卡、粘贴很慢、更容易粘漏。想瘦身就把那段 `/* ... */` 整段删掉。

### 编码会不会影响 js？会，而且是致命的

edgetunnel 里有大量**中文变量名**（`缓存SOCKS5白名单`、`调试日志打印` 等），这些名字对编码极其敏感。实测：

| 保存成 | 结果 |
|---|---|
| UTF-8（带或不带 BOM） | ✅ 正常 |
| ANSI / GBK（记事本默认就是它！） | ❌ `SyntaxError: Invalid or unexpected token` |
| UTF-16 LE（记事本里的「Unicode」） | ❌ 同样报错 |

记事本「另存为」时编码默认是 ANSI，中文系统上就是 GBK，直接粘上去必然部署失败。

### 本程序怎么处理

`cfip_gui.exe` 读文件时会看 BOM：

- 有 UTF-16LE BOM → 转成 UTF-8
- 有 UTF-8 BOM → 保留
- 没有 BOM → 做一次 UTF-8 合法性校验，**不通过就判定为 ANSI/GBK，自动转成 UTF-8** 再注入

注入完成的提示里会写原文件是什么编码、已经转成什么。**所以用记事本存成 ANSI 也没关系，程序会救回来。**

### 也不用非得存成 `.js`

记事本存 `.txt` 一样能注入，程序输出扩展名跟着输入走（`.txt` → `_已注入.txt`）。CF 后台粘代码不看扩展名。
